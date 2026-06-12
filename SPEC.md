# Spec — Afya GitHub Actions Reusable Workflows

**Versão:** 1.0  
**Data:** 2026-06-12  
**Repositório:** `adhfoundation/github-actions-workflows`

---

## 1. Visão Geral

Este repositório centraliza **reusable workflows** do GitHub Actions utilizados pelos times de engenharia da Afya. O objetivo é evitar duplicação de lógica de CI/CD entre os repositórios da organização, garantindo consistência, rastreabilidade de versão e manutenção simplificada.

Todos os workflows ficam em `.github/workflows/` e são invocados via `workflow_call` por outros repositórios.

> **Por que não Composite Actions?**  
> Composite Actions de repositórios privados exigem GitHub Enterprise Cloud. Para manter o repositório privado sem custo adicional, o padrão adotado é exclusivamente Reusable Workflows.

---

## 2. Objetivos

| Objetivo | Descrição |
|---|---|
| Reuso | Eliminar cópias de workflows espalhadas por dezenas de repositórios |
| Padronização | Um padrão único de build, deploy e validação para toda a organização |
| Versionamento | Evolução controlada via sufixos (`-v2`, `-v3`, `-v4`) sem quebrar chamadores existentes |
| Segurança | Centralizar práticas de segurança (OIDC, secrets, scanning) |
| Observabilidade | Notificações e integrações com ferramentas externas (Teams, Jira) |

---

## 3. Arquitetura

```
adhfoundation/github-actions-workflows
└── .github/
    └── workflows/
        ├── build-*          # Construção de imagens Docker
        ├── deploy-*         # Deploy via ArgoCD
        ├── helm-check-*     # Validação de APIs Kubernetes
        ├── k8s-check-*      # Compatibilidade de versão K8s
        ├── notify-*         # Notificações (Microsoft Teams)
        ├── jira-*           # Integrações com Jira
        ├── controller-*     # Gestão de releases
        ├── create-*         # Criação de releases
        ├── provision-*      # Provisionamento de infraestrutura
        └── terminate-*      # Teardown de infraestrutura
```

**Infraestrutura alvo:**
- Cloud: AWS (ECR, EC2, Secrets Manager, IAM OIDC)
- GitOps: ArgoCD + Helm
- Kubernetes: clusters gerenciados (EKS ou similar)
- Registro de imagens: Amazon ECR (público e privado)

---

## 4. Catálogo de Workflows

### 4.1 Build de Imagens Docker

#### `build-image-to-ecr.yml` (v1)
Build e push de imagem para ECR com autenticação OIDC e scan Trivy.

| Campo | Detalhe |
|---|---|
| Inputs obrigatórios | `aws-creds-role-to-assume`, `aws-region`, `docker-tag`, `ecr-url`, `ecr-repo` |
| Inputs opcionais | `docker-build-context`, `docker-file-path`, `docker-build-target`, `docker-build-args`, `scan-image`, `trivy-severity`, `trivy-fail-on-findings` |
| Secrets opcionais | `PIP_EXTRA_INDEX_URL`, `GH_ACCESS_TOKEN`, variáveis `NEXT_PUBLIC_*` |
| Runner | `ubuntu-22.04` |
| Timeout | 30 minutos |
| Scan | Trivy roda por default (`scan-image: true`); resultados sobem ao GitHub Security via SARIF |

#### `build-image-to-ecr-v2.yaml` (v2 — recomendado)
Versão evoluída com scan Trivy integrado, GitHub App JWT para pacotes privados e suporte a ECR público.

| Campo | Detalhe |
|---|---|
| Inputs obrigatórios | `docker-tag`, `environment` |
| Inputs de scan | `scan-image` (default `true`), `trivy-severity` (default `CRITICAL,HIGH`), `trivy-fail-on-findings` (default `false`) |
| Secrets obrigatórios | Nenhum além do `GITHUB_TOKEN` nativo |
| Secrets opcionais | `NPM_TOKEN`, credenciais GitHub App |
| Variáveis de ambiente | `vars.ECR_URL`, `vars.ECR_REPO`, `vars.AWS_CREDS_ROLE_TO_ASSUME`, `vars.AWS_REGION` (via GitHub Environment) |
| Scan | Trivy escaneia a imagem no ECR usando as credenciais AWS do job; SARIF enviado ao GitHub Code Scanning — anotações aparecem automaticamente em PRs sem configuração adicional nos repositórios chamadores |

---

### 4.2 Deploy via ArgoCD

Quatro versões em produção, cada uma adicionando capacidades:

| Versão | Arquivo | Diferenciais |
|---|---|---|
| v1 | `deploy-argocd.yml` | Base; todos os parâmetros explícitos |
| v2 | `deploy-argocd-v2.yaml` | `vars.*` para configs ArgoCD, `additional-helm-set-values` |
| v3 | `deploy-argocd-v3.yaml` | Prefixo `argocd-` nos inputs, runner `ubuntu-24.04` |
| v4 | `deploy-argocd-v4.yaml` | gRPC keep-alive, sync policy, auto-prune, self-heal, wait configurável |

#### `deploy-argocd-v4.yaml` — inputs principais

| Input | Default | Descrição |
|---|---|---|
| `app-name` | `${{ github.event.repository.name }}` | Nome da aplicação ArgoCD |
| `environment` | — | Environment do GitHub (obrigatório) |
| `image-tag` | — | Tag da imagem a ser deployada |
| `sync-policy` | `manual` | `manual` ou `automated` |
| `argocd-wait` | `true` | Aguardar sincronização completa |
| `argocd-wait-timeout` | `300` | Timeout em segundos |
| `auto-prune` | `false` | Remover recursos órfãos |
| `self-heal` | `false` | Reverter alterações manuais |
| `grpc-keep-alive-min` | `20s` | Intervalo de keep-alive gRPC |

#### `deploy-argocd-preview.yaml`
Deploy de ambientes efêmeros para Pull Requests.

- App criada como `{environment}-{app-name}-{preview-number}`
- `preview-number` padrão: número do PR (`${{ github.event.number }}`)
- `revision` padrão: branch do PR (`${{ github.head_ref }}`)

---

### 4.3 Validação de APIs Kubernetes

#### `helm-check-api.yml` — Pluto
Detecta uso de APIs Kubernetes depreciadas em Helm charts.

- Busca arquivos `*value*.yaml` no caminho configurado
- Executa `helm template` + `pluto detect`
- Versão K8s alvo padrão: `v1.27`
- Saída em markdown (configurável)

#### `k8s-check-api.yml` — Kubent
Detecta APIs removidas usando cluster KinD local.

- Sobe cluster KinD, instala `helm`, `kubectl` e `kubent`
- Versão K8s alvo padrão: `1.26`
- Complementar ao Pluto (diferentes motores de detecção)

---

### 4.4 Notificações

#### `notify-teams.yaml`
Envia Adaptive Cards para Microsoft Teams via Power Automate.

| Evento (`event-type`) | Comportamento |
|---|---|
| `pr` | Notifica criação de novo PR |
| `review_requested` | Menciona o revisor pelo email (via JSON mapping) |
| `review` | Notifica conclusão da review com emoji por estado |

- Webhook opcional: se `TEAMS_WEBHOOK_URL` não estiver configurado, o workflow é silencioso
- Mapping `github-to-teams-mapping`: JSON `{"github_login": "email@empresa.com"}`

---

### 4.5 Integração com Jira

#### `jira-issue-required-v2.yml`
Impede merge de PRs sem tarefa Jira vinculada e no status correto.

- Extrai padrão `RMP-XXXX` da descrição do PR
- Falha se nenhum issue for encontrado
- Consulta API Jira e valida status `"Ready for Deployment"`
- Falha se qualquer issue não estiver no status esperado

**Secrets necessários:** `GH_ACCESS_TOKEN`, `jira_base_url`, `jira_email`, `jira_api_token`

---

### 4.6 Gestão de Releases

#### `controller-draft-release.yml`
Cria e atualiza draft releases automaticamente a partir de PRs mergeados.

- Usa `release-drafter/release-drafter@v5`
- Lê configuração em `configs/draft-release.yml` do repositório chamador
- Requer PAT com permissão de `contents:write`

#### `create-release.yml`
Cria releases versionadas com semver.

| Input | Descrição |
|---|---|
| `bump-type` | `major`, `minor` ou `patch` |
| `prefix` | Prefixo da tag (ex: `v`, `release-`) |
| **Output** | `release-tag`: tag criada |

---

### 4.7 Infraestrutura EC2

#### `provision-vm.yml`
Provisiona instâncias EC2.

- Inputs: `aws-region`, `instance-type`, `image-id`, `aws-creds-role-to-assume`
- Output: `INSTANCE_ID` para encadeamento

#### `terminate-vm.yml`
Termina instâncias EC2.

- Inputs: `aws-region`, `instance-id`, `aws-creds-role-to-assume`
- Uso típico: teardown de ambientes de preview/teste

---

### 4.8 AWS Secrets Manager

#### `aws-secrets-manager-action.yml`
Cria ou atualiza segredos no AWS Secrets Manager.

- **Status:** Incompleto — esqueleto presente, implementação pendente
- Inputs: `secret-name`, `secret-value`, `aws-region`
- Secret: `AWS_ROLE_TO_ASSUME`

---

## 5. Convenções e Padrões

### Nomenclatura de arquivos
- Formato: `kebab-case` com extensão `.yml` (legado) ou `.yaml` (novos)
- Versionamento: sufixo `-v2`, `-v3`, `-v4` para evoluções sem breaking change

### Nomenclatura de inputs
- `kebab-case` em minúsculas
- Prefixos por domínio: `docker-*`, `argocd-*`, `aws-*`, `helm-*`
- Booleanos: sem prefixo de verbo (ex: `public-repository`, `auto-prune`)

### Nomenclatura de secrets
- `SCREAMING_SNAKE_CASE` para secrets de infraestrutura
- `kebab-case` aceitável na definição do workflow

### Variáveis de ambiente vs. inputs
A partir de v2/v3, valores de infraestrutura específicos do ambiente (URLs, clusters, projetos ArgoCD) são lidos de `vars.*` (GitHub Environment Variables) em vez de passados como inputs — reduz acoplamento entre chamador e workflow.

### Runner padrão
- Workflows legados: `ubuntu-22.04`
- Workflows novos: `ubuntu-24.04` ou `ubuntu-latest`

---

## 6. Como Usar

### Chamando um workflow
```yaml
jobs:
  deploy:
    uses: adhfoundation/github-actions-workflows/.github/workflows/deploy-argocd-v4.yaml@main
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.docker-tag }}
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

### Encadeando build + deploy
```yaml
jobs:
  build:
    uses: adhfoundation/github-actions-workflows/.github/workflows/build-image-to-ecr-v2.yaml@main
    with:
      docker-tag: ${{ github.sha }}
      environment: staging

  deploy:
    needs: build
    uses: adhfoundation/github-actions-workflows/.github/workflows/deploy-argocd-v4.yaml@main
    with:
      environment: staging
      image-tag: ${{ github.sha }}
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

### Pinando versão
Para estabilidade, referencie por SHA ou tag em vez de `@main`:
```yaml
uses: adhfoundation/github-actions-workflows/.github/workflows/deploy-argocd-v4.yaml@v1.2.3
```

---

## 7. Dependências Externas

| Ferramenta | Versão usada | Propósito |
|---|---|---|
| `docker/setup-qemu-action` | v3 | Builds multi-plataforma |
| `docker/setup-buildx-action` | v3 | BuildKit avançado |
| `docker/build-push-action` | v5 | Build e push de imagens |
| `aquasecurity/trivy-action` | v0.36.0 | Scan de vulnerabilidades (OS + libs) |
| `github/codeql-action/upload-sarif` | v3 | Upload de resultados SARIF ao GitHub Security |
| `aws-actions/configure-aws-credentials` | v4 | Autenticação OIDC AWS |
| `aws-actions/amazon-ecr-login` | v2 | Login no ECR |
| `FairwindsOps/pluto` | v5 | Detecção de APIs K8s depreciadas |
| `kubent` | latest | Detecção de APIs K8s removidas |
| `helm/kind-action` | v1.8.0 | Cluster KinD para testes |
| `release-drafter/release-drafter` | v5 | Draft releases automáticos |
| `zwaldowski/semver-release-action` | v3 | Cálculo de versão semver |
| `ncipollo/release-action` | v1 | Criação de releases GitHub |

---

## 8. Lacunas Identificadas

| Item | Situação | Impacto |
|---|---|---|
| `aws-secrets-manager-action.yml` | Esqueleto incompleto | Funcionalidade prometida não disponível |
| Mistura de extensões `.yml`/`.yaml` | Inconsistente | Dificulta scripts e automações |
| Ausência de testes dos workflows | Sem validação automatizada | Regressões silenciosas |
| Versões não fixadas para ações externas | `latest` em algumas | Quebras inesperadas por atualização |
| Sem CHANGELOG | Sem histórico de mudanças | Dificulta adoção de novas versões |
| `configs/draft-release.yml` não versionado | Esperado no chamador | Documentação insuficiente do contrato |

---

## 9. Próximos Passos Sugeridos

1. **Completar `aws-secrets-manager-action.yml`** — ou remover o arquivo para não causar confusão
2. **Padronizar extensão de arquivos** — migrar todos para `.yaml`
3. **Criar CHANGELOG.md** — documentar breaking changes entre versões
4. **Fixar versões de ações externas** — substituir `@latest` por SHAs ou tags semver
5. **Adicionar workflow de lint/test** — validar sintaxe dos workflows em PRs deste repositório
6. **Documentar `deploy-argocd-preview.yaml`** — workflow de preview não aparece no README atual
7. **Consolidar EC2 workflows** — `provision-vm.yml` e `terminate-vm.yml` estão sem exemplos de uso
