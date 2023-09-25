# Afya Reusable Workflows
Repositorio de github actions

Workflows centralizados num unico lugar, para reuso, manutenção e escalabilidade de código.

Ref: https://www.zup.com.br/blog/reusable-workflow-e-composite-action

## Where put our reusable workflows?
Reusable workflows must be in the folder `./github/workflows`, no exception.

## Why not use Composite Actions?
To use `Composite Actions` from a private repository, the organization must be
in a `Github Enterprise Cloud`, a premium feature. Otherwise this repo must be public

## Usage
In your project repository, call one of our reusable workflows using the following sample code:

```
name: Test reusable workflow

on:
  workflow_dispatch:

jobs:
  call_remote_workflow:
    # change example.yaml to your desirable workflow
    # change master to your workflow version
    uses: Afya/github-actions-workflows/.github/workflows/example.yaml@master
    # insert here your workflow inputs
    with:
      input1: value1
    secrets:
      somesecrettoken: ${{ secrets.YOUR_REPO_SECRET }}
```

## New workflow template
```
# .github/workflows/template.yaml
name: Reusable Workflow

on:
  workflow_call:
    inputs:
      input1:
        type: string
        required: false
        default: 'default value'
    secrets:
      somesecrettoken:
        required: true

jobs:
  example:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Run ls
        shell: bash
        run: |
          ls
 ```

