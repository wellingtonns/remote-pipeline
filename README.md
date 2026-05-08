# Templates de pipeline

Estrutura compartilhada para padronizar build, push, atualizacao de imagem e deploy Kubernetes para projetos frontend e backend.

Os manifests Kustomize ficam em um repositorio separado:

```text
https://github.com/wellingtonns/kustomize-share
```

## Estrutura

```text
templates-pipeline/
  workflows/
    app-pipeline.yml
    build-frontend-nixpacks.yml
    build-backend-nixpacks.yml
    update-kustomize-image.yml
    deploy-kustomize.yml
    frontend-pipeline-example.yml
    backend-pipeline-example.yml
```

## Como usar

1. Copie os arquivos de `templates-pipeline/workflows` para `.github/workflows` no repositorio que vai centralizar os templates.
2. Em cada projeto, crie um workflow pequeno chamando `app-pipeline.yml`.
3. Passe `stack_type: frontend` ou `stack_type: backend` para o template principal escolher o build correto.
4. Passe `kustomize_repository` apontando para o repositorio separado de Kustomize.
5. Apos o build e push da imagem, o template chama `update-kustomize-image.yml` para atualizar a tag no overlay Kustomize usando `newTag`.
6. O deploy pode ser feito por GitOps, como Argo CD ou Flux, observando o repositorio de Kustomize. Se quiser deploy direto pelo GitHub Actions, use `deploy: true`.

## Exemplo de pipeline principal

```yaml
name: App Clinica

on:
  push:
    branches:
      - main

jobs:
  pipeline:
    uses: ./.github/workflows/app-pipeline.yml
    with:
      app_name: app-clinica
      stack_type: frontend
      context: app-clinica
      image_name: ghcr.io/${{ github.repository_owner }}/app-clinica
      kustomize_repository: wellingtonns/kustomize-share
      kustomize_path: app/app-clinica
      image_alias: app-image
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
