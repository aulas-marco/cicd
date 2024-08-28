# Como fazer testes de pipelines

Aqui temos um roteiro de um teste de unidade genérico que poderia ser aplicada a uma coleção de pipelines para garantir a conformidade dele a melhores práticas.

Assuma que você tenha a seguinte estrutura de arquivos

```
my_project/
│
├── .github/
│   └── workflows/
│       └── deploy.yml  # Seu pipeline YAML
│
└── tests/
    └── test_pipeline.py  # Testes de unidade para o pipeline
```

## Arquito deploy.yml

```yaml
name: Pipeline de Deploy .NET na AWS

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  DOTNET_VERSION: '7.x'
  AWS_REGION: 'us-west-2'
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPOSITORY: 'minhaaplicacao'
  IMAGE_TAG: ${{ github.sha }}

jobs:
  preparar-ambiente:
    runs-on: ubuntu-latest

    steps:
      - name: Baixar código
        uses: actions/checkout@v2
        # Baixa o código do repositório

      - name: Configurar .NET Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
        # Configura o ambiente .NET Core

      - name: Cache de pacotes NuGet
        uses: actions/cache@v2
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-
        # Utiliza cache para melhorar a performance do pipeline

  compilar:
    runs-on: ubuntu-latest
    needs: preparar-ambiente

    steps:
      - name: Restaurar dependências
        run: dotnet restore
        # Restaura as dependências do projeto

      - name: Compilar o projeto
        run: dotnet build --configuration Release --no-restore
        # Compila o projeto .NET

      - name: Publicar o projeto
        run: dotnet publish --configuration Release --output ./publish --no-build
        # Publica o projeto no diretório especificado

      - name: Enviar artefatos de compilação
        uses: actions/upload-artifact@v2
        with:
          name: app-publicada
          path: ./publish
        # Faz upload dos artefatos gerados

  testar:
    runs-on: ubuntu-latest
    needs: compilar

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Configurar .NET Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restaurar dependências
        run: dotnet restore
        # Restaura as dependências para os testes

      - name: Executar testes de unidade
        run: dotnet test --no-build --verbosity normal --configuration Release
        # Executa os testes de unidade

      - name: Enviar resultados dos testes
        uses: actions/upload-artifact@v2
        with:
          name: resultados-testes
          path: '**/TestResults/*.trx'
        # Faz upload dos resultados dos testes

  construir-imagem-docker:
    runs-on: ubuntu-latest
    needs: testar

    steps:
      - name: Baixar código
        uses: actions/checkout@v2

      - name: Fazer login no Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1
        with:
          registry: ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com
        # Faz login no Amazon ECR (Elastic Container Registry)

      - name: Construir imagem Docker
        run: |
          docker build -t ${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} .
        # Constrói a imagem Docker

      - name: Enviar imagem Docker para o ECR
        run: |
          docker tag ${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} \
            ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker push ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
        # Envia a imagem Docker para o Amazon ECR

  deploy:
    runs-on: ubuntu-latest
    needs: construir-imagem-docker

    steps:
      - name: Realizar deploy no AWS Elastic Beanstalk
        run: |
          aws elasticbeanstalk create-environment \
            --application-name "MinhaAplicacao" \
            --environment-name "MeuApp-Ambiente" \
            --solution-stack-name "64bit Amazon Linux 2 v3.2.6 running Docker" \
            --version-label ${{ env.IMAGE_TAG }} \
            --option-settings Namespace=aws:elasticbeanstalk:environment:process,OptionName=Dockerrun.aws.json,Value=file://deploy/Dockerrun.aws.json
        # Implanta a aplicação no AWS Elastic Beanstalk usando a imagem Docker enviada ao ECR

```

Explicação do Pipeline em Português

	1.	Preparar Ambiente:
	•	Baixa o código do repositório.
	•	Configura o ambiente .NET Core na versão especificada.
	•	Utiliza cache para melhorar a performance na restauração de pacotes NuGet.
	2.	Compilar:
	•	Restaura as dependências do projeto.
	•	Compila o projeto com a configuração de Release.
	•	Publica o projeto no diretório de saída.
	•	Faz upload dos artefatos gerados.
	3.	Testar:
	•	Executa testes de unidade no projeto compilado.
	•	Faz upload dos resultados dos testes como artefatos para análise posterior.
	4.	Construir Imagem Docker:
	•	Constrói a imagem Docker da aplicação usando o Dockerfile no repositório.
	•	Faz login no Amazon ECR e envia a imagem Docker para o repositório do ECR.
	5.	Deploy:
	•	Implanta a aplicação no AWS Elastic Beanstalk usando a imagem Docker enviada ao ECR.


## Teste de Unidade

Agora vamos criar um exemplo de teste de unidade de Pipeline em Python

```py
import pytest
import yaml

@pytest.fixture
def carregar_pipeline():
    # Carrega o pipeline YAML como um dicionário Python
    with open(".github/workflows/deploy.yml", 'r') as stream:
        return yaml.safe_load(stream)

def test_variaveis_ambiente(carregar_pipeline):
    # Verifica se as variáveis de ambiente estão definidas corretamente
    env_vars = carregar_pipeline['env']
    
    assert 'DOTNET_VERSION' in env_vars
    assert env_vars['DOTNET_VERSION'] == '7.x'

    assert 'AWS_REGION' in env_vars
    assert env_vars['AWS_REGION'] == 'us-west-2'

def test_jobs_existem(carregar_pipeline):
    # Verifica se os jobs esperados estão presentes no pipeline
    jobs = carregar_pipeline['jobs']
    
    assert 'preparar-ambiente' in jobs
    assert 'compilar' in jobs
    assert 'testar' in jobs
    assert 'construir-imagem-docker' in jobs
    assert 'deploy' in jobs

def test_sequencia_jobs(carregar_pipeline):
    # Verifica se a sequência e dependências dos jobs estão configuradas corretamente
    jobs = carregar_pipeline['jobs']
    
    assert jobs['compilar']['needs'] == 'preparar-ambiente'
    assert jobs['testar']['needs'] == 'compilar'
    assert jobs['construir-imagem-docker']['needs'] == 'testar'
    assert jobs['deploy']['needs'] == 'construir-imagem-docker'

def test_cache_nuget(carregar_pipeline):
    # Verifica se o cache de pacotes NuGet está configurado corretamente
    steps = carregar_pipeline['jobs']['preparar-ambiente']['steps']
    
    found_cache = False
    for step in steps:
        if step.get('uses') == 'actions/cache@v2':
            found_cache = True
            assert step['with']['path'] == '~/.nuget/packages'
            assert 'key' in step['with']
            break
    
    assert found_cache, "Step de cache NuGet não encontrado no job preparar-ambiente"

def test_variaveis_segredos(carregar_pipeline):
    # Verifica se os secrets necessários estão sendo usados no job de deploy
    deploy_steps = carregar_pipeline['jobs']['deploy']['steps']
    
    for step in deploy_steps:
        if 'uses' in step and 'aws-actions/configure-aws-credentials' in step['uses']:
            assert '${{ secrets.AWS_ACCESS_KEY_ID }}' in step['with'].values()
            assert '${{ secrets.AWS_SECRET_ACCESS_KEY }}' in step['with'].values()
            assert '${{ secrets.AWS_REGION }}' in step['with'].values()
            break
    else:
        assert False, "Credenciais AWS não configuradas corretamente no job de deploy"

def test_deploy_beanstalk(carregar_pipeline):
    # Verifica se o deploy no AWS Elastic Beanstalk está configurado corretamente
    steps = carregar_pipeline['jobs']['deploy']['steps']
    
    found_deploy = False
    for step in steps:
        if 'run' in step and 'aws elasticbeanstalk create-environment' in step['run']:
            found_deploy = True
            assert '--application-name "MinhaAplicacao"' in step['run']
            assert '--environment-name "MeuApp-Ambiente"' in step['run']
            assert '--solution-stack-name "64bit Amazon Linux 2 v3.2.6 running Docker"' in step['run']
            break
    
    assert found_deploy, "Step de deploy no Elastic Beanstalk não encontrado"

def test_substituicao_variaveis(carregar_pipeline):
    # Verifica se as variáveis de ambiente são substituídas corretamente no YAML
    deploy_steps = carregar_pipeline['jobs']['deploy']['steps']
    
    for step in deploy_steps:
        if 'run' in step and '${{ env.IMAGE_TAG }}' in step['run']:
            assert '${{ env.AWS_ACCOUNT_ID }}' in step['run']
            assert '${{ env.AWS_REGION }}' in step['run']
            break
    else:
        assert False, "Substituição de variáveis de ambiente falhou no step de deploy"
```

Para rodar esse codigo, utilize o comando pytest

```sh
pytest tests/
```

Explicação dos Testes

	•	test_variaveis_ambiente: Valida se as variáveis de ambiente, como DOTNET_VERSION e AWS_REGION, estão configuradas corretamente no pipeline.
	•	test_jobs_existem: Confirma que todos os jobs esperados (preparar-ambiente, compilar, testar, etc.) estão presentes no pipeline.
	•	test_sequencia_jobs: Verifica se a sequência dos jobs e suas dependências (needs) estão configuradas corretamente.
	•	test_cache_nuget: Checa se o cache de pacotes NuGet está configurado, o que ajuda a melhorar a performance do pipeline.
	•	test_variaveis_segredos: Garante que os secrets necessários (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY) estão sendo utilizados corretamente no job de deploy.
	•	test_deploy_beanstalk: Verifica se o comando de deploy para o AWS Elastic Beanstalk está configurado corretamente.
	•	test_substituicao_variaveis: Verifica se as variáveis de ambiente, como IMAGE_TAG, AWS_ACCOUNT_ID, e AWS_REGION, estão sendo substituídas corretamente nos comandos.