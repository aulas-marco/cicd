
# Workflow Completo para Construção, Publicação e Provisionamento de Ambiente de Testes com Docker

Este documento descreve como criar um workflow do GitHub Actions que automatiza o processo de construção, assinatura, publicação e provisionamento de um ambiente de testes usando Docker. O workflow inclui a criação de um artefato .NET Core, publicação em um repositório binário imutável, e a configuração de um ambiente de testes com Docker.

## Estrutura do Repositório

Antes de criar o workflow, certifique-se de que o repositório tenha a seguinte estrutura:

```
your-repo/
├── .github/
│   └── workflows/
│       └── package-provision-and-publish-artifact.yml
├── db-init-scripts/
│   ├── entrypoint.sh
│   └── initialize.sql
├── src/
│   ├── MyApplication/
│   │   ├── Dockerfile
│   │   ├── MyApplication.csproj
│   │   └── Program.cs
└── docker-compose.test.yml
```

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
