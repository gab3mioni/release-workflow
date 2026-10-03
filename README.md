# release-workflow

Workflow reutilizável do GitHub Actions. Quando um PR é mergeado na branch padrão, ele calcula a próxima versão semver a partir dos Conventional Commits, atualiza o changelog, cria a tag e publica o GitHub Release.

## Uso

Crie `.github/workflows/release.yml` no seu repositório:

```yaml
name: Release

on:
  pull_request:
    types: [closed]

jobs:
  release:
    if: github.event.pull_request.merged && github.event.pull_request.base.ref == github.event.repository.default_branch
    permissions:
      contents: write
    uses: gab3mioni/release-workflow/.github/workflows/release.yml@v1
```

O caller é o mesmo para todo repositório. Sem filtro de `branches`, ele funciona com `main`, `master` ou outra branch padrão.

O `permissions: contents: write` no job é obrigatório. Repositórios com `default_workflow_permissions: read` limitam o token sem ele, e o push da tag falha.

Para mudar um input, adicione `with:` no job:

```yaml
    uses: gab3mioni/release-workflow/.github/workflows/release.yml@v1
    with:
      changelog-path: ''
```

## Modelos de branch

O workflow detecta o modelo pelas branches que existem no `origin`, nesta ordem. A branch padrão é ignorada na detecção.

| Existe | Origem aceita | Merge commit | Sync após o release |
| --- | --- | --- | --- |
| `staging` | `staging` | obrigatório | `staging`, depois `develop` (se existir) |
| `develop` | `develop` | obrigatório | `develop` |
| só a branch padrão | qualquer | não obrigatório (squash ok) | nenhum |

No modelo só com a branch padrão, o squash gera um único commit e o git-cliff lê a mensagem dele. Em PR com mais de um commit, o GitHub usa o título do PR. Em PR com um commit só, usa a mensagem desse commit. Para valer sempre o título, ative "Default to pull request title" em Settings > General > Pull Requests. Em qualquer caso, a mensagem final precisa seguir Conventional Commits.

PR de outra origem não gera release. Exemplo: um hotfix direto em `main` num repositório com `develop`. O job termina verde com este notice e nada é publicado:

```
Notice: This repository releases from develop into main; hotfix into main is not a release, nothing was published.
```

## Inputs

| Input | Default | Descrição |
| --- | --- | --- |
| `tag-prefix` | `v` | Prefixo da tag. |
| `changelog-path` | `CHANGELOG.md` | Arquivo de changelog commitado na branch padrão. Vazio publica só a tag e o Release, sem commit. |
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

## Merge commit

Nos modelos `staging` e `develop`, o PR para a branch padrão precisa ser mergeado com merge commit. Com squash ou rebase o job falha com:

```
Error: <sha> is not a merge commit. Merge staging into main with a merge commit (not squash or rebase); nothing was published.
```

Nada é publicado nesse caso.

## Primeiro release

Sem tag com o prefixo, a versão é `initial-version`. O changelog cobre só os commits do PR.

## Branch protection

Se a branch padrão exige PR para receber commits, o workflow não consegue commitar o changelog. Use `changelog-path: ''`. A tag e o Release são publicados normalmente.

## Falhas

- **A branch padrão andou durante o job.** O commit do changelog e a tag vão num único push atômico. Se a branch padrão recebeu outro commit, o push é rejeitado e nada é publicado. Rode o job de novo.
- **Tag já existe.** Se a versão calculada já tem tag e Release, não há commits que gerem release. O job falha com `Tag <tag> already exists: no releasable commits since the last release.`
- **Release não criada.** Se `gh release create` falha depois do push, a tag fica sem Release. Rode o job de novo: ele pula changelog, tag e sync e só publica a Release, com as notas daquela tag.
- **Sync rejeitado.** O sync faz push do commit de release em cada branch do modelo, em ordem. Se uma branch divergiu, o push dela é rejeitado com um warning e a próxima branch é tentada. O job termina verde. O warning é `Could not fast-forward <branch> to <tag>. Merge <base> into <branch> manually.` Faça esse merge à mão.

## Versões deste repositório

- `@v1`: tag móvel, movida à mão para o último release `v1.x.y`.
- `@v1.0.0`: tag fixa, nunca muda.

Mudanças que quebram o contrato (inputs, comportamento) saem como `v2`.

## Custo

Um job em `ubuntu-latest`, de 20 a 40 segundos. Em repositório privado é cobrado como 1 minuto. O job roda para todo PR mergeado na branch padrão. Nos modelos `staging` e `develop`, um PR de outra origem roda um job curto que termina com um notice. PRs mergeados em outras branches geram um job skipped, sem custo. Repositórios públicos não pagam.
