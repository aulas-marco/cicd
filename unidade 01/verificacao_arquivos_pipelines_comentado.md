
# Validação e Testes de Linting para Pipelines de GitHub Actions

Este documento descreve como configurar e utilizar ferramentas de linting para validar a configuração de pipelines YAML no GitHub Actions, garantindo que eles estejam bem formatados, livres de erros de sintaxe, e seguindo as melhores práticas.

## yamllint

`yamllint` é uma ferramenta de linting para arquivos YAML que verifica a sintaxe, o estilo e as práticas recomendadas.

Para instalá-lo, use o comando pip:

```bash
pip install yamllint
```

Depois, você deve criar um arquivo de configuração `.yamllint` para personalizar as regras que deseja aplicar. Um exemplo mais completo é mostrado abaixo:

```yaml
# Este arquivo define regras de linting para arquivos YAML

# Extende a configuração padrão do yamllint
extends: default

rules:
  # Define o comprimento máximo de linha permitido
  line-length:
    max: 120
    level: warning  # Emite um aviso se o comprimento da linha for excedido

  # Define a indentação para usar 2 espaços
  indentation:
    spaces: 2
    level: error  # Gera um erro se a indentação estiver incorreta

  # Verifica se o documento começa com '---'
  document-start:
    present: true
    level: error  # Gera um erro se a marca de início do documento estiver ausente

  # Limita os valores verdadeiros a 'true' ou 'false'
  truthy:
    allowed-values: ['true', 'false']

  # Define as regras para os espaços ao redor de dois pontos
  colons:
    max-spaces-before: 0
    max-spaces-after: 1

  # Define as regras para os espaços ao redor de vírgulas
  commas:
    max-spaces-before: 0
    max-spaces-after: 1

  # Requer um mínimo de 2 espaços entre comentários e o conteúdo
  comments:
    min-spaces-from-content: 2
    level: warning  # Emite um aviso se não houver espaço suficiente

  # Define regras para os espaços dentro de colchetes
  brackets:
    min-spaces-inside: 0
    max-spaces-inside: 0

  # Enforce the use of Unix-style newlines
  new-lines:
    type: unix
    level: error  # Gera um erro se o formato de nova linha estiver incorreto

  # Proíbe espaços em branco no final das linhas
  trailing-spaces:
    level: error  # Gera um erro se houver espaços em branco à direita
```

Para rodar o lint no seu pipeline YAML:

```bash
yamllint .github/workflows/deploy.yml
```

## ActionLint

`actionlint` é uma ferramenta de linting para pipelines do GitHub Actions. Ela verifica não apenas a sintaxe YAML, mas também as configurações e valores específicos do GitHub Actions.

Para instalar no macOS:

```bash
brew install actionlint
```

Para outras plataformas, você pode baixar o binário diretamente do repositório do `actionlint`.

Para rodar o linting no seu pipeline:

```bash
actionlint .github/workflows/deploy.yml
```

## GitHub Actions Linter

Você pode integrar o linting diretamente ao seu pipeline usando uma ação GitHub.

### Exemplo de Integração:

```yaml
# Nome do pipeline que descreve a ação que está sendo realizada
name: Pipeline de Deploy .NET na AWS

# Define em quais eventos o pipeline será executado
on:
  push:
    branches:
      - main  # Executa o pipeline quando há push na branch 'main'
  pull_request:
    branches:
      - main  # Executa o pipeline em pull requests direcionados para a branch 'main'

jobs:
  # Define o job de linting
  lint:
    runs-on: ubuntu-latest  # Especifica o sistema operacional da máquina virtual onde o job será executado
    steps:
      - name: Baixar código
        uses: actions/checkout@v2  # Baixa o código do repositório

      - name: Verificar sintaxe YAML
        uses: github/super-linter@v3  # Usa o Super-Linter para verificar o YAML e outras regras de linting
        env:
          VALIDATE_YAML: true  # Habilita a validação de arquivos YAML
          VALIDATE_GITHUB_ACTIONS: true  # Habilita a validação das configurações específicas do GitHub Actions
          DEFAULT_BRANCH: main  # Define a branch padrão para a validação
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Usa o token de autenticação do GitHub para acessar o repositório
```

Esse exemplo usa o `super-linter`, que pode verificar YAML, JSON, Markdown, e outros tipos de arquivos.

### Validação de CI/CD Pipeline via Super-Linter

O Super-Linter é uma ação de GitHub que oferece suporte para uma ampla gama de linguagens, incluindo YAML e configurações específicas para pipelines CI/CD.

No seu pipeline GitHub, adicione um job de validação usando Super-Linter:

```yaml
# Nome do pipeline que descreve a ação que está sendo realizada
name: Validate CI Pipeline

# Define em quais eventos o pipeline será executado
on:
  pull_request:
    paths:
      - '.github/workflows/**'  # Executa o pipeline quando arquivos em '.github/workflows/' são alterados

jobs:
  # Define o job de linting
  lint:
    runs-on: ubuntu-latest  # Especifica o sistema operacional da máquina virtual onde o job será executado
    steps:
      - name: Baixar código
        uses: actions/checkout@v2  # Baixa o código do repositório

      - name: Run Linter
        uses: github/super-linter@v3  # Usa o Super-Linter para verificar o YAML e outras regras de linting
        env:
          VALIDATE_YAML: true  # Habilita a validação de arquivos YAML
          VALIDATE_GITHUB_ACTIONS: true  # Habilita a validação das configurações específicas do GitHub Actions
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Usa o token de autenticação do GitHub para acessar o repositório
```

Esse job será executado em todas as pull requests que alterem arquivos na pasta `.github/workflows/`, verificando se o pipeline está bem formatado e segue as melhores práticas.

## Prettier para YAML

Prettier é uma ferramenta de formatação de código que pode ser usada para garantir que o YAML seja formatado de maneira consistente.

Para instalar, use o npm:

```bash
npm install --save-dev prettier prettier-plugin-yaml
```

Para rodar, faça o seguinte:

```bash
npx prettier --write .github/workflows/deploy.yml
```

---

Essas alterações melhoram a precisão e a aplicabilidade das ferramentas de linting em seus pipelines de GitHub Actions.
