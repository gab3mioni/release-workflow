# release-workflow

Workflow reutilizável do GitHub Actions. Quando um PR de `develop` para `main` é mergeado, ele calcula a próxima versão semver a partir dos Conventional Commits, atualiza o changelog, cria a tag e publica o GitHub Release.

## Uso

Crie `.github/workflows/release.yml` no seu repositório:

```yaml
name: Release

on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    if: github.event.pull_request.merged && github.event.pull_request.head.ref == 'develop'
    permissions:
      contents: write
    uses: gab3mioni/release-workflow/.github/workflows/release.yml@v1
```

O `permissions: contents: write` no job é obrigatório. Repositórios com `default_workflow_permissions: read` limitam o token sem ele, e o push da tag falha.

Para mudar um input, adicione `with:` no job:

```yaml
    uses: gab3mioni/release-workflow/.github/workflows/release.yml@v1
    with:
      changelog-path: ''
```

## Inputs

| Input | Default | Descrição |
| --- | --- | --- |
| `tag-prefix` | `v` | Prefixo da tag. |
| `changelog-path` | `CHANGELOG.md` | Arquivo de changelog commitado em `main`. Vazio publica só a tag e o Release, sem commit. |
| `sync-branch` | `develop` | Branch que recebe o commit de release depois. Vazio desliga o sync. |
| `initial-version` | `0.1.0` | Versão usada quando o repositório não tem tag com o prefixo. |

## Regra de bump

A versão sai dos commits desde a última tag, via [git-cliff](https://git-cliff.org).

| Commit | Em 0.x | Em 1.x ou maior |
| --- | --- | --- |
| Breaking change (`feat!:`, `BREAKING CHANGE:`) | minor | major |
| `feat` | minor | minor |
| Qualquer outro tipo (`fix`, `chore`, `docs`...) | patch | patch |

Merge commits, commits `chore(release)` e commits fora do padrão Conventional Commits são ignorados.

Para usar outra regra, coloque um `cliff.toml` próprio na raiz do repositório. O workflow usa esse arquivo no lugar do padrão. O `tag-prefix` continua valendo, porque o workflow passa `--tag-pattern` ao git-cliff.

## Merge de develop para main

O PR `develop → main` precisa ser mergeado com merge commit. Com squash ou rebase o job falha com:

```
Error: <sha> is not a merge commit. Merge develop into main with a merge commit (not squash or rebase); nothing was published.
```

Nada é publicado nesse caso.

## Primeiro release

Sem tag com o prefixo, a versão é `initial-version`. O changelog cobre só os commits do PR.

## Branch protection

Se `main` exige PR para receber commits, o workflow não consegue commitar o changelog. Use `changelog-path: ''`. A tag e o Release são publicados normalmente.

## Falhas

- **main andou durante o job.** O commit do changelog e a tag vão num único push atômico. Se `main` recebeu outro commit, o push é rejeitado e nada é publicado. Rode o job de novo.
- **Tag já existe.** Se a versão calculada já tem tag e Release, não há commits que gerem release. O job falha com `Tag <tag> already exists: no releasable commits since the last release.`
- **Release não criada.** Se `gh release create` falha depois do push, a tag fica sem Release. Rode o job de novo: ele pula changelog, tag e sync e só publica a Release, com as notas daquela tag.
- **Sync rejeitado.** Se `sync-branch` divergiu de `main`, o push falha com um warning e o job termina verde. Faça o merge de `main` em `sync-branch` manualmente.

## Versões deste repositório

- `@v1`: tag móvel, movida à mão para o último release `v1.x.y`.
- `@v1.0.0`: tag fixa, nunca muda.

Mudanças que quebram o contrato (inputs, comportamento) saem como `v2`.

## Custo

Um job em `ubuntu-latest`, de 20 a 40 segundos. Em repositório privado é cobrado como 1 minuto. PRs que não vêm de `develop` geram um job skipped, sem custo. Repositórios públicos não pagam.
