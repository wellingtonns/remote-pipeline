# Templates de pipeline e Kustomize

Estrutura compartilhada para padronizar build, push, atualizacao de imagem e deploy Kubernetes para projetos frontend e backend.

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
  kustomize-share/
    base/
      deployment.yaml
      service.yaml
      ingress.yaml
      kustomization.yaml
    app/
      app-clinica/
        kustomization.yaml
        namespace.yaml
        deployment-patch.yaml
        ingress-patch.yaml
      example-frontend/
        kustomization.yaml
        namespace.yaml
        deployment-patch.yaml
        ingress-patch.yaml
      example-backend/
        kustomization.yaml
        namespace.yaml
        deployment-patch.yaml
        ingress-patch.yaml
```

## Como usar

1. Copie os arquivos de `templates-pipeline/workflows` para `.github/workflows` no repositorio que vai centralizar os templates.
2. Em cada projeto, crie um workflow pequeno chamando `app-pipeline.yml`.
3. Passe `stack_type: frontend` ou `stack_type: backend` para o template principal escolher o build correto.
4. Apos o build e push da imagem, o template chama `update-kustomize-image.yml` para atualizar a tag no overlay Kustomize usando `newTag`.
5. O deploy pode ser feito por GitOps, como Argo CD ou Flux, observando o repositorio de Kustomize. Se quiser deploy direto pelo GitHub Actions, use `deploy: true`.

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
      kustomize_path: templates-pipeline/kustomize-share/app/app-clinica
      image_alias: app-image
      deploy: true
      deploy_ref: main
    secrets: inherit
```

## Padrao da imagem

Por padrao os exemplos publicam no GitHub Container Registry:

```text
ghcr.io/OWNER/IMAGE_NAME:TAG
```

Use secrets quando publicar em outro registry:

- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`

Para deploy direto no cluster:

- `KUBE_CONFIG`: kubeconfig em base64.

## Atualizacao com Kustomize

Cada overlay em `kustomize-share/app/<app>` define a imagem assim:

```yaml
images:
  - name: ghcr.io/example/app
    newName: ghcr.io/example/app
    newTag: latest
```

O template `update-kustomize-image.yml` executa:

```bash
kustomize edit set image app-image=ghcr.io/example/app:nova-tag
```

Isso altera somente a tag do app no manifesto Kustomize.
