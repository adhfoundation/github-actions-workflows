# Afya GitHub Actions Reusable Workflows

Repositório centralizado de Reusable Workflows do GitHub Actions para os times de engenharia da Afya.

## O que é este repositório

Biblioteca de workflows reutilizáveis (`workflow_call`) invocados por outros repositórios da organização `adhfoundation`. Não usa Composite Actions — o repositório é privado e Composite Actions de repos privados exigem GitHub Enterprise Cloud.

Todos os workflows ficam em `.github/workflows/`. Documentação completa em `SPEC.md`.

## Domínios cobertos

- **Build Docker → ECR:** `build-image-to-ecr.yml` (v1), `build-image-to-ecr-v2.yaml` (v2, com Docker Scout)
- **Deploy ArgoCD:** `deploy-argocd.yml` → `deploy-argocd-v4.yaml` (4 versões), `deploy-argocd-preview.yaml`
- **Validação K8s:** `helm-check-api.yml` (Pluto), `k8s-check-api.yml` (Kubent)
- **Notificações:** `notify-teams.yaml` (Microsoft Teams via Power Automate)
- **Jira:** `jira-issue-required-v2.yml` (gate de deploy por status de tarefa)
- **Releases:** `controller-draft-release.yml`, `create-release.yml`
- **Infra EC2:** `provision-vm.yml`, `terminate-vm.yml`
- **Secrets AWS:** `aws-secrets-manager-action.yml` (incompleto)

## Convenções

- Nomes de arquivo: `kebab-case`, extensão `.yml` (legado) ou `.yaml` (novos)
- Versionamento de workflows: sufixo `-v2`, `-v3`, `-v4` — nunca quebra chamadores existentes
- Inputs: `kebab-case`, prefixados por domínio (`docker-*`, `argocd-*`, `aws-*`)
- Secrets: `SCREAMING_SNAKE_CASE`
- Configs específicas de ambiente (URLs, clusters): lidas de `vars.*` no GitHub Environment — não passadas como input
- Runner padrão atual: `ubuntu-24.04` (legado usa `ubuntu-22.04`)

## Como workflows são chamados

```yaml
jobs:
  deploy:
    uses: adhfoundation/github-actions-workflows/.github/workflows/deploy-argocd-v4.yaml@main
    with:
      environment: production
      image-tag: ${{ github.sha }}
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

## Skills recomendadas (Claude Code)

Antes de abrir uma PR com novo workflow ou alteração em existente, rodar:

- `/security-review` — verifica script injection, supply chain, credenciais estáticas e flags não sanitizadas. Especialmente importante neste repo porque workflows são compartilhados por dezenas de repositórios — uma vulnerabilidade aqui tem blast radius amplo.
- `/code-review` — revisa contratos de inputs/outputs e possíveis regressões em chamadores existentes.

## Lacunas conhecidas

- `aws-secrets-manager-action.yml` é só um esqueleto, sem implementação
- Mistura de extensões `.yml` e `.yaml` sem padrão fixo
- Sem CHANGELOG entre versões dos workflows
- Algumas ações externas sem versão fixada (`latest`)
