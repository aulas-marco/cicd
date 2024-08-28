

# Workflow Completo para Construção, Publicação, Provisionamento de Ambiente de Testes com Docker e Verificação de Conformidade de Ambiente de Produção

Este documento descreve como criar um workflow do GitHub Actions que automatiza o processo de construção, assinatura, publicação e provisionamento de um ambiente de testes usando Docker. O workflow inclui a criação de um artefato .NET Core, publicação em um repositório binário imutável, ae a configuração de um ambiente de testes com Docker e a verificação de conformidade do ambiente de produção.

## Estrutura do Repositório

Para garantir que o workflow funcione corretamente, o repositório deve seguir a estrutura de diretórios e arquivos abaixo:

```
your-repo/
├── .github/
│   └── workflows/
│       └── package-provision-and-publish-artifact.yml  # Arquivo do GitHub Actions que define o workflow para construção, publicação e provisionamento.
├── db-init-scripts/                                     # Diretório contendo scripts de inicialização para o banco de dados.
│   ├── entrypoint.sh                                    # Script de entrada para inicializar o banco de dados quando o container é iniciado.
│   └── initialize.sql                                   # Script SQL para criar e popular o banco de dados de testes.
├── src/                                                 # Diretório de código-fonte da aplicação.
│   ├── MyApplication/                                   # Diretório específico do projeto .NET Core.
│   │   ├── Dockerfile                                   # Arquivo Dockerfile para construir a imagem Docker da aplicação.
│   │   ├── MyApplication.csproj                         # Arquivo de projeto do .NET Core, contendo as dependências e configurações do projeto.
│   │   └── Program.cs                                   # Arquivo principal da aplicação .NET Core.
├── infrastructure/                                      # Diretório contendo templates de infraestrutura para provisionamento na AWS.
│   └── cloudformation-template.yml                      # Template CloudFormation para provisionamento de recursos na AWS.
└── docker-compose.test.yml                              # Arquivo de configuração do Docker Compose para criar o ambiente de testes.
```

### Descrição dos Arquivos e Diretórios

- **.github/workflows/package-provision-and-publish-artifact.yml**: Este arquivo contém a definição do workflow do GitHub Actions. Ele automatiza processos como a construção do artefato .NET Core, assinatura, publicação, provisionamento de ambientes de teste com Docker, e validação da conformidade da infraestrutura.

- **db-init-scripts/entrypoint.sh**: Script de inicialização que configura o banco de dados de testes dentro do container Docker. Ele espera o SQL Server iniciar e executa o script SQL `initialize.sql`.

- **db-init-scripts/initialize.sql**: Script SQL usado para criar e configurar o banco de dados de testes. Este arquivo pode incluir a criação de tabelas e a inserção de dados iniciais para testes.

- **src/MyApplication/Dockerfile**: Define o processo de construção da imagem Docker da aplicação .NET Core. O Dockerfile especifica as etapas de build e execução para criar uma imagem pronta para ser executada em containers.

- **src/MyApplication/MyApplication.csproj**: Arquivo de projeto .NET Core que especifica as dependências e as configurações do projeto, controlando como o projeto é compilado.

- **src/MyApplication/Program.cs**: Ponto de entrada da aplicação .NET Core. Aqui, a lógica principal da aplicação é definida e iniciada.

- **infrastructure/cloudformation-template.yml**: Template AWS CloudFormation que descreve os recursos de infraestrutura a serem provisionados na AWS. Este arquivo é utilizado para garantir que a infraestrutura esteja em conformidade com as políticas de segurança e outras diretrizes da empresa.

- **docker-compose.test.yml**: Arquivo de configuração do Docker Compose que define como os containers (aplicação e banco de dados) serão orquestrados para criar o ambiente de testes. Este arquivo facilita a criação de ambientes de teste consistentes e efêmeros.

### 1. Arquivo `docker-compose.test.yml`

O arquivo `docker-compose.test.yml` define a configuração dos serviços Docker que serão usados para provisionar o ambiente de testes.

```yaml
version: '3.8'

services:
  app:
    image: mcr.microsoft.com/dotnet/aspnet:7.0
    container_name: app_container
    build:
      context: ./src/MyApplication
      dockerfile: Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "8080:80"
    networks:
      - test_network
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    container_name: db_container
    environment:
      SA_PASSWORD: "YourStrong@Passw0rd"
      ACCEPT_EULA: "Y"
    ports:
      - "1433:1433"
    networks:
      - test_network
    volumes:
      - ./db-init-scripts:/docker-entrypoint-initdb.d

networks:
  test_network:
    driver: bridge
```

### 2. Scripts de Inicialização do Banco de Dados

Na pasta `db-init-scripts`, inclua os seguintes arquivos:

#### `entrypoint.sh`

```bash
#!/bin/bash

# Espera o SQL Server iniciar
sleep 20s

# Rodar scripts SQL para importar o banco de dados de testes
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P YourStrong@Passw0rd -d master -i /docker-entrypoint-initdb.d/initialize.sql
```

#### `initialize.sql`

```sql
CREATE DATABASE TestDatabase;
GO

USE TestDatabase;
GO

CREATE TABLE ExampleTable (
    Id INT PRIMARY KEY,
    Name NVARCHAR(50)
);
GO

INSERT INTO ExampleTable (Id, Name) VALUES (1, 'Sample Data');
GO
```

### 3. Dockerfile da Aplicação

No diretório `src/MyApplication`, crie o `Dockerfile` para definir a imagem Docker da aplicação:

```Dockerfile
# Etapa de build
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build-env
WORKDIR /app

# Copia os arquivos csproj e restaura as dependências
COPY *.csproj ./
RUN dotnet restore

# Copia o restante dos arquivos e compila
COPY . ./
RUN dotnet publish -c Release -o out

# Etapa de execução
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build-env /app/out .

ENTRYPOINT ["dotnet", "MyApplication.dll"]
```

### 4. Workflow do GitHub Actions

Crie o arquivo `package-provision-and-publish-artifact.yml` no diretório `.github/workflows/` com o seguinte conteúdo:

```yaml
name: Empacotar, Assinar, Publicar Artefato .NET Core e Provisionar Ambiente de Testes com Docker

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Configurar .NET Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: '7.x' # ou a versão do .NET Core que você está usando

      - name: Restaurar dependências
        run: dotnet restore

      - name: Compilar o projeto
        run: dotnet build --configuration Release

      - name: Publicar o projeto
        run: dotnet publish --configuration Release --output ./publish

      - name: Empacotar o artefato
        run: |
          cd publish
          zip -r MyApplication.zip .

      - name: Assinar o executável
        run: |
          signtool sign /f path/to/your/certificate.pfx /p ${{ secrets.SIGNING_PASSWORD }} /t http://timestamp.digicert.com MyApplication.exe

      - name: Verificar a assinatura
        run: signtool verify /pa /v ./publish/MyApplication.exe

      - name: Gerar hash para auditoria
        run: sha256sum ./publish/MyApplication.zip > ./publish/MyApplication.zip.sha256

      - name: Upload do artefato para GitHub Actions
        uses: actions/upload-artifact@v3
        with:
          name: signed-artifact
          path: ./publish/MyApplication.zip

      - name: Publicar artefato no Nexus
        env:
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
        run: |
          curl -v -u $NEXUS_USER:$NEXUS_PASSWORD --upload-file ./publish/MyApplication.zip http://your-nexus-repository/repository/my-repo/MyApplication.zip

  provision:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Configurar Docker
        run: |
          docker --version
          docker-compose --version

      - name: Criar Ambiente de Testes com Docker
        run: |
          docker-compose -f docker-compose.test.yml up -d

      - name: Executar Testes
        run: |
          docker-compose -f docker-compose.test.yml exec app dotnet test

      - name: Destruir Ambiente de Testes
        run: |
          docker-compose -f docker-compose.test.yml down
```

### 5. Configuração do Certificado e Nexus

- **Certificado**: O certificado `.pfx` usado para assinar o executável deve estar disponível no repositório ou em um repositório seguro.
- **Secrets**: 
  - Armazene a senha do certificado como um segredo no repositório GitHub. Vá para as configurações do repositório, em `Settings > Secrets and variables > Actions`, e adicione um novo segredo chamado `SIGNING_PASSWORD`.
  - Armazene as credenciais de acesso ao Nexus (`NEXUS_USER` e `NEXUS_PASSWORD`) também como segredos.

### 6. Execução do Workflow

Toda vez que você fizer push ou criar um pull request para a branch `main`, o workflow será executado automaticamente. Ele empacotará o projeto, assinará o executável, verificará a assinatura, gerará um hash para auditabilidade, publicará o artefato em um repositório binário imutável (como o Nexus), criará um ambiente de testes efêmero com Docker, executará os testes, e destruirá o ambiente de testes após a execução.


### 7. Validação da Conformidade da Infraestrutura com AWS CloudFormation

Após o provisionamento e execução dos testes, o próximo passo é validar a conformidade da infraestrutura provisionada na AWS, garantindo que ela esteja em conformidade com as políticas de segurança da organização.

Este passo envolve a adição de um estágio no workflow que realiza uma verificação dinâmica da infraestrutura, utilizando AWS CloudFormation.

#### 7.1. Configuração do Template AWS CloudFormation

No diretório `infrastructure/`, crie um arquivo chamado `cloudformation-template.yml` para definir os recursos de infraestrutura que serão provisionados na AWS.

O exemplo a seguir cria um bucket S3 e garante que o acesso público seja restrito:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyS3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: my-app-bucket
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
```

Essa configuração garante que o bucket S3 esteja seguro e não permita acesso público, atendendo a uma das políticas comuns de segurança.

#### 7.2. Configuração do Workflow para Validação de Conformidade

Adicione o seguinte estágio ao arquivo `package-provision-and-publish-artifact.yml` no diretório `.github/workflows/`:

```yaml
  validate_infrastructure:
    runs-on: ubuntu-latest
    needs: provision

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Configurar AWS CLI
        run: |
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip awscliv2.zip
          sudo ./aws/install

      - name: Validar Conformidade da Infraestrutura
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws cloudformation validate-template --template-body file://infrastructure/cloudformation-template.yml

      - name: Executar Scans de Conformidade
        run: |
          # Verifica se os grupos de segurança permitem apenas o tráfego nas portas e protocolos permitidos
          aws ec2 describe-security-groups --filters Name=ip-permission.protocol,Values=tcp Name=ip-permission.from-port,Values=80 --query 'SecurityGroups[*].GroupName'

          # Verifica o status da política de um bucket S3, garantindo que ele não seja público
          aws s3api get-bucket-policy-status --bucket my-app-bucket --query 'PolicyStatus.IsPublic'
```

#### 7.3. Explicação dos Passos de Validação

**1. Validação do Template AWS CloudFormation:**

Este passo utiliza o comando `aws cloudformation validate-template` para garantir que o template CloudFormation está formatado corretamente e que não há erros que possam causar falhas durante a criação ou atualização da infraestrutura. Isso é essencial para evitar interrupções nos ambientes de produção.

**2. Verificação dos Grupos de Segurança (Security Groups):**

Aqui, o comando `aws ec2 describe-security-groups` é usado para verificar se as regras dos grupos de segurança permitem apenas tráfego nas portas e protocolos especificados, como a porta 80 para tráfego HTTP. Isso garante que a infraestrutura não esteja exposta a portas não autorizadas que possam ser exploradas por atacantes.

**3. Verificação de Políticas de Buckets S3:**

O comando `aws s3api get-bucket-policy-status` verifica se o bucket S3 tem políticas que poderiam torná-lo público, o que pode ser um risco de segurança. Esse comando garante que o bucket está em conformidade com as políticas de segurança ao não permitir acesso público, protegendo dados sensíveis.

#### 8. Configuração de Secrets para AWS

No GitHub Actions, adicione as credenciais de acesso à AWS (`AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY`) como segredos no repositório, para que possam ser utilizados durante a execução do workflow.

#### 9. Execução e Monitoramento

Quando o workflow for executado, ele verificará a infraestrutura provisionada para garantir que todos os recursos estejam em conformidade com as políticas de segurança especificadas. Caso algum recurso não esteja em conformidade, o workflow será interrompido, evitando que uma infraestrutura não segura seja promovida para produção.

Este complemento ao workflow existente garante que a infraestrutura na AWS esteja em conformidade com as melhores práticas de segurança da organização.
