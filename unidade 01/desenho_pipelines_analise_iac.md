
# Exemplo de Template CloudFormation para Validar Acesso Público a Buckets S3

Este documento descreve como criar um arquivo de template AWS CloudFormation (`template.yaml`) que define um bucket S3 com uma configuração que garante o bloqueio de acesso público. Além disso, mostra como integrar este template a um workflow do GitHub Actions para validação automatizada.

## Estrutura do Arquivo `template.yaml`

Abaixo está o exemplo de um arquivo `template.yaml` que cria um bucket S3 com políticas de bloqueio de acesso público:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Template para criar um bucket S3 com bloqueio de acesso público.

Resources:
  MeuBucketS3:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: meu-bucket-exemplo
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

Outputs:
  NomeDoBucketS3:
    Description: "O nome do bucket S3"
    Value: !Ref MeuBucketS3
```

### Explicação do Template

- **AWSTemplateFormatVersion**: Define a versão do formato do template CloudFormation.
- **Description**: Uma breve descrição do que o template faz.
- **Resources**: A seção onde os recursos da infraestrutura são definidos. Neste exemplo, um recurso de bucket S3 é criado.
- **Type**: Especifica o tipo de recurso que está sendo criado (`AWS::S3::Bucket` no caso de um bucket S3).
- **Properties**: Define as propriedades do recurso. Aqui, estamos definindo o nome do bucket e a configuração para bloquear o acesso público.
  - **PublicAccessBlockConfiguration**: Configurações que garantem que o acesso público ao bucket S3 seja bloqueado.
    - **BlockPublicAcls**: Bloqueia ACLs públicas.
    - **BlockPublicPolicy**: Bloqueia políticas públicas.
    - **IgnorePublicAcls**: Ignora todas as ACLs públicas.
    - **RestrictPublicBuckets**: Restringe a configuração de buckets públicos.
- **Outputs**: Seção que define as saídas do template, como o nome do bucket S3 criado.

## Integração com GitHub Actions

O GitHub Actions permite automatizar fluxos de trabalho para construção, teste e implementação de código. Neste caso, vamos usar um workflow para validar automaticamente se o template de infraestrutura (IaC) segue as políticas de segurança da empresa, como o bloqueio de acesso público a buckets S3.

### Estrutura do Arquivo de Workflow `validar-iac.yml`

Abaixo está um exemplo de como criar um workflow do GitHub Actions para validar o template `template.yaml`:

```yaml
name: Validar IaC

on:
  pull_request:
    branches:
      - main
      - develop

jobs:
  validar-iac:
    runs-on: ubuntu-latest

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Instalar AWS CLI
        run: |
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip awscliv2.zip
          sudo ./aws/install

      - name: Instalar cfn-lint
        run: pip install cfn-lint

      - name: Lint nos templates CloudFormation
        run: cfn-lint -t infrastructure/template.yaml

      - name: Validar Políticas do Bucket S3
        run: |
          if grep -q '"PublicAccessBlockConfiguration":' infrastructure/template.yaml; then
            echo "Bloqueio de acesso público está definido, continuando..."
          else
            echo "Bloqueio de acesso público não está definido! Falhando..."
            exit 1
          fi
```

### Explicação do Workflow

- **name**: Nome do workflow (`Validar IaC`).
- **on**: Define quando o workflow será acionado. Neste exemplo, o workflow é acionado em pull requests feitos para as branches `main` e `develop`.
- **jobs**: Define o trabalho (`validar-iac`) que será executado.
  - **runs-on**: Especifica o ambiente onde o trabalho será executado (`ubuntu-latest`).
  - **steps**: Lista os passos que serão executados dentro do trabalho:
    - **Baixar código**: Usa a ação `checkout` para fazer o download do código do repositório.
    - **Instalar AWS CLI**: Instala o AWS CLI, necessário para interagir com serviços da AWS.
    - **Instalar cfn-lint**: Instala o `cfn-lint`, uma ferramenta para validar templates CloudFormation.
    - **Lint nos templates CloudFormation**: Executa o `cfn-lint` para verificar se o template segue as melhores práticas e está livre de erros.
    - **Validar Políticas do Bucket S3**: Valida se o bloqueio de acesso público está definido no template. Se não estiver, o pipeline falha.

### Como Usar

1. **Salvar o Template no Repositório**:
   - Salve o arquivo `template.yaml` na pasta `infrastructure` ou em outra de sua escolha dentro do repositório.
   - Certifique-se de que o caminho para o arquivo `template.yaml` no workflow (`cfn-lint -t infrastructure/template.yaml`) corresponda ao local onde o arquivo foi salvo.

2. **Configurar o Workflow**:
   - Salve o conteúdo YAML do workflow em um arquivo chamado `validar-iac.yml` dentro da pasta `.github/workflows` no seu repositório.

3. **Executar o Workflow**:
   - Toda vez que um pull request for criado ou atualizado nas branches `main` ou `develop`, o workflow será executado automaticamente. Se o bloqueio de acesso público ao bucket S3 não estiver configurado corretamente, o pipeline falhará, impedindo a integração de código que não esteja em conformidade.

### Testar o Template Manualmente

Se desejar testar o template manualmente antes de integrá-lo ao GitHub Actions:

1. **Usar AWS CLI**: Execute o comando abaixo para criar o stack a partir do template:

   ```sh
   aws cloudformation create-stack --stack-name MeuStack --template-body file://caminho/para/template.yaml
   ```

2. **Console AWS**: Carregue o `template.yaml` diretamente no console da AWS ao criar um novo stack CloudFormation.


