
# Empacotamento, Assinatura e Publicação de Artefatos .NET Core com GitHub Actions

Este documento descreve como criar um workflow do GitHub Actions que automatiza o processo de empacotamento, assinatura, verificação de integridade e publicação de um artefato .NET Core. O workflow incluirá a criação de um executável (.exe), empacotamento em um arquivo .zip, assinatura digital para garantir a integridade antes de implantar o artefato em ambientes de teste ou produção, e a publicação em um repositório binário imutável.

## 1. Criar o Workflow no GitHub Actions

No seu repositório GitHub, crie um arquivo chamado `package-and-publish-artifact.yml` na pasta `.github/workflows`.

## 2. Configuração do Workflow

Aqui está um exemplo de configuração do workflow:

```yaml
name: Empacotar, Assinar e Publicar Artefato .NET Core

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
```

## 3. Explicação do Workflow

- **name**: Nome do workflow (`Empacotar, Assinar e Publicar Artefato .NET Core`).
- **on**: Define quando o workflow será acionado. Neste exemplo, o workflow é acionado em pushs ou pull requests na branch `main`.
- **jobs**: Define o trabalho (`build`) que será executado.
  - **runs-on**: Especifica o ambiente onde o trabalho será executado (`ubuntu-latest`).
  - **steps**: Lista os passos que serão executados dentro do trabalho:
    - **Baixar código**: Usa a ação `checkout` para fazer o download do código do repositório.
    - **Configurar .NET Core**: Configura o ambiente para usar o .NET Core, instalando a versão especificada.
    - **Restaurar dependências**: Restaura as dependências do projeto usando `dotnet restore`.
    - **Compilar o projeto**: Compila o projeto em modo Release.
    - **Publicar o projeto**: Publica o projeto para a pasta `./publish`.
    - **Empacotar o artefato**: Empacota os arquivos publicados em um arquivo `.zip`.
    - **Assinar o executável**: Usa a ferramenta `signtool` para assinar o arquivo `.exe`. O caminho para o certificado `.pfx` deve ser fornecido, e a senha do certificado deve ser armazenada como um segredo no repositório GitHub (`SIGNING_PASSWORD`).
    - **Verificar a assinatura**: Verifica se o arquivo foi assinado corretamente.
    - **Gerar hash para auditoria**: Gera um hash SHA-256 do arquivo `.zip` para garantir a integridade e auditabilidade do pacote.
    - **Upload do artefato para GitHub Actions**: Faz o upload do artefato assinado para o GitHub Actions como um artefato de build.
    - **Publicar artefato no Nexus**: Publica o artefato no repositório binário imutável (por exemplo, Nexus). As credenciais são armazenadas como segredos no GitHub (`NEXUS_USER` e `NEXUS_PASSWORD`).

## 4. Configuração do Certificado e Nexus

- **Certificado**: O certificado `.pfx` usado para assinar o executável deve estar disponível no repositório ou em um repositório seguro.
- **Secrets**: 
  - Armazene a senha do certificado como um segredo no repositório GitHub. Vá para as configurações do repositório, em `Settings > Secrets and variables > Actions`, e adicione um novo segredo chamado `SIGNING_PASSWORD`.
  - Armazene as credenciais de acesso ao Nexus (`NEXUS_USER` e `NEXUS_PASSWORD`) também como segredos.

## 5. Execução do Workflow

Toda vez que você fizer push ou criar um pull request para a branch `main`, o workflow será executado automaticamente. Ele empacotará o projeto, assinará o executável, verificará a assinatura, gerará um hash para auditabilidade, e publicará o artefato em um repositório binário imutável (como o Nexus). O artefato final, incluindo o arquivo `.zip` assinado, estará disponível para download no GitHub Actions e no Nexus.

Este processo automatiza completamente o empacotamento, a garantia de integridade e a publicação do artefato .NET Core usando GitHub Actions, facilitando a integração contínua do código.
