# Templates de pipeline

Estrutura compartilhada para padronizar build, push, atualizacao de imagem e deploy Kubernetes para projetos frontend e backend.

Os manifests Kustomize ficam em um repositorio separado:

```text
https://github.com/wellingtonns/kustomize-share
```

## Estrutura

```text
templates-pipeline/
  .github/
    workflows/
    app-pipeline.yml
    ensure-kustomize-app.yml
    build-frontend-nixpacks.yml
    build-backend-nixpacks.yml
    secret-scan.yml
    update-kustomize-image.yml
    deploy-kustomize.yml
  examples/
    frontend-pipeline-example.yml
    backend-pipeline-example.yml
    secret-scan-example.yml
```

## Como usar

1. Use os workflows reutilizaveis publicados em `wellingtonns/templates-pipeline/.github/workflows`.
2. Em cada projeto, crie um workflow pequeno chamando `wellingtonns/templates-pipeline/.github/workflows/app-pipeline.yml@main`.
3. Passe `stack_type: frontend` ou `stack_type: backend` para o template principal escolher o build correto.
4. Passe `kustomize_repository` apontando para o repositorio separado de Kustomize.
5. Se a pasta do app ainda nao existir no repositorio Kustomize, o template chama `ensure-kustomize-app.yml` para criar o overlay.
6. Apos o build e push da imagem, o template chama `update-kustomize-image.yml` para atualizar a tag no overlay Kustomize usando `newTag`.
7. O deploy pode ser feito por GitOps, como Argo CD ou Flux, observando o repositorio de Kustomize. Se quiser deploy direto pelo GitHub Actions, use `deploy: true`.

## Template de analise de secrets

Use `secret-scan.yml` para verificar se o repositorio tem credenciais expostas, como tokens, chaves privadas, senhas, arquivos `.env` commitados ou secrets de provedores cloud.

O template executa:

- `Gitleaks`: identifica padroes conhecidos de credenciais.
- `TruffleHog`: procura secrets validaveis, por padrao somente os verificados.

Exemplo de uso em um repositorio de aplicacao:

```yaml
name: Secret Scan

on:
  workflow_dispatch:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  secret-scan:
    uses: wellingtonns/templates-pipeline/.github/workflows/secret-scan.yml@main
    with:
      scan_path: .
      fail_on_leak: true
      scan_history: false
      run_gitleaks: true
      run_trufflehog: true
```

Parametros principais:

- `scan_path`: pasta analisada. Padrao: `.`.
- `fail_on_leak`: falha o workflow quando encontra segredo. Padrao: `true`.
- `scan_history`: quando `false`, analisa somente os arquivos atuais e a pipeline passa apos remover a secret. Quando `true`, analisa todo o historico Git para auditoria. Padrao: `false`.
- `gitleaks_image`: imagem Docker usada pelo Gitleaks. Padrao: `ghcr.io/gitleaks/gitleaks:latest`.
- `gitleaks_config_path`: caminho opcional para `.gitleaks.toml`, util para allowlist de falsos positivos.
- `trufflehog_extra_args`: argumentos extras do TruffleHog. Padrao: `--only-verified`.

Quando o job falhar, considere qualquer segredo apontado como comprometido: remova do historico se necessario, revogue a credencial no provedor de origem e gere uma nova.

## Exemplo de pipeline principal

```yaml
name: App Clinica

on:
  push:
    branches:
      - main

jobs:
  pipeline:
    uses: wellingtonns/templates-pipeline/.github/workflows/app-pipeline.yml@main
    with:
      app_name: app-clinica
      stack_type: frontend
      context: app-clinica
      image_name: ghcr.io/${{ github.repository_owner }}/app-clinica
      kustomize_repository: wellingtonns/kustomize-share
      kustomize_path: app/app-clinica
      image_alias: app-image
      ensure_kustomize: true
      app_namespace: app-clinica
      app_domain: app-clinica.example.com
      update_kustomize: true
      deploy: true
      deploy_ref: main
    secrets: inherit
```

## Secrets necessarios

Use secrets quando publicar em outro registry:

- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`

Para atualizar o repositorio separado de Kustomize:

- `KUSTOMIZE_REPO_TOKEN`: token com permissao de escrita no repositorio `kustomize-share`.

## Criacao automatica do overlay Kustomize

Quando `ensure_kustomize: true`, a pipeline verifica se o caminho informado em `kustomize_path` existe no repositorio `kustomize-share`.

Exemplo:

```yaml
kustomize_path: app/app-clinica
```

Se a pasta `app/app-clinica` nao existir, o template cria:

```text
app/app-clinica/
  kustomization.yaml
  namespace.yaml
  deployment-patch.yaml
  ingress-patch.yaml
```

Os nomes internos sao preenchidos usando `app_name`, `app_namespace`, `app_domain`, `image_name` e `image_alias`.

Para deploy direto no cluster:

- `KUBE_CONFIG`: kubeconfig em base64.

## Padrao da imagem

Por padrao os exemplos publicam no GitHub Container Registry:

```text
ghcr.io/OWNER/IMAGE_NAME:TAG
```

## Atualizacao com Kustomize

Cada overlay no repositorio `kustomize-share`, em `app/<app>`, define a imagem assim:

```yaml
images:
  - name: app-image
    newName: ghcr.io/example/app
    newTag: latest
```

O template `update-kustomize-image.yml` executa:

```bash
kustomize edit set image app-image=ghcr.io/example/app:nova-tag
```

Isso altera somente a tag do app no manifesto Kustomize.
