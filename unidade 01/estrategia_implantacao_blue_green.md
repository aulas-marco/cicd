
# Blue-Green Deployment com GitHub Actions e AWS Elastic Beanstalk

Este guia descreve como configurar um fluxo de trabalho no GitHub Actions que realiza o Blue-Green Deployment de uma aplicação .NET Core na AWS usando o Elastic Beanstalk.

## O que é Blue-Green Deployment?

A estratégia de Blue-Green Deployment é uma técnica de implantação que minimiza o tempo de inatividade e reduz os riscos associados ao lançamento de novas versões de uma aplicação. O conceito básico envolve ter dois ambientes idênticos, chamados "Blue" e "Green". 

### Como Funciona?

1. **Ambiente Blue:** Este é o ambiente de produção atual, onde a versão existente da aplicação está em execução. O tráfego de usuários é direcionado para este ambiente.

2. **Ambiente Green:** Este é o ambiente que contém a nova versão da aplicação. Enquanto o ambiente Green está sendo preparado, ele não recebe tráfego dos usuários.

3. **Troca de Ambientes (Swap):** Uma vez que a nova versão no ambiente Green é validada e está pronta para produção, o tráfego é redirecionado do ambiente Blue para o ambiente Green. Esse processo é chamado de "swap" dos ambientes.

4. **Descarte do Ambiente Antigo:** Após a troca, o antigo ambiente Blue pode ser mantido como backup temporário, ou pode ser descartado. Em caso de problemas com a nova versão, a reversão é simples: basta redirecionar o tráfego de volta para o ambiente Blue.

```
Usuários  --->  [Ambiente Blue]
                 | Versão Antiga
                 |
            +---------------+
            | Load Balancer  |
            +---------------+
                 | Em Preparação
                 |
                 |
                 |
                 v
+---------------+     +---------------+
| Servidor 1    |     | Servidor 2    |
| Aplicação     |     | Aplicação     |
+---------------+     +---------------+

  [Após os testes de fumaça em producao]

Usuários  --->  [Ambiente Green]
                 | Versão Nova
                 |
            +---------------+
            | Load Balancer  |
            +---------------+
                 |
                 |
                 v
+---------------+     +---------------+
| Servidor 1    |     | Servidor 2    |
| Aplicação     |     | Aplicação     |
+---------------+     +---------------+
```


### Integração com Banco de Dados

Na estratégia de Blue-Green Deployment, o ponto de partida é uma infraestrutura com a versão antiga (versão A, a versão azul) da aplicação e do banco de dados. Em paralelo, é construída uma nova infraestrutura, que possui a nova versão (versão B, a versão verde) instalada. O balanceador de carga alterna instantaneamente da infraestrutura A para B, direcionando o tráfego para a nova versão. Se o sistema tiver um banco de dados, duas opções são possíveis:

1. **A nova versão da aplicação pode funcionar com a versão antiga do banco de dados.**
2. **A versão antiga da aplicação pode funcionar com a nova versão do banco de dados.**

Freqüentemente, a primeira opção não é possível, pois uma nova versão da aplicação geralmente requer uma mudança no banco de dados, específica para a nova versão da aplicação.  O ponto de partida neste exemplo é um pool de servidores (pool de servidores A) contendo ambos os servidores 1 e 2, cada um executando a versão A da aplicação. O balanceador de carga distribui todas as solicitações entre ambos os servidores no pool.

### Benefícios

- **Minimiza o Tempo de Inatividade:** Como a troca de ambientes é instantânea, os usuários experimentam praticamente nenhum tempo de inatividade.
- **Facilita a Reversão:** Se algo der errado com a nova versão, é fácil reverter para o ambiente antigo.
- **Testes em Produção:** O ambiente Green pode ser testado com dados reais antes de ser lançado ao público.

## Passo 1: Estrutura do Repositório

Certifique-se de que o repositório tenha a seguinte estrutura básica:

```
your-repo/
├── .github/
│   └── workflows/
│       └── blue-green-deployment.yml
├── src/
│   ├── MyApplication/
│   │   ├── Dockerfile
│   │   ├── MyApplication.csproj
│   │   └── Program.cs
└── deploy/
    ├── Dockerrun.aws.json
    ├── appspec.yaml
```

### Descrição dos Arquivos e Diretórios

- **.github/workflows/blue-green-deployment.yml**: Contém o workflow do GitHub Actions que automatiza o processo de construção, teste e implantação da aplicação .NET Core usando Blue-Green Deployment na AWS.
- **src/MyApplication/**: Diretório contendo o código-fonte da aplicação .NET Core.
- **deploy/Dockerrun.aws.json**: Arquivo de configuração para o AWS Elastic Beanstalk, especificando como o Docker deve ser executado.
- **deploy/appspec.yaml**: Arquivo de configuração do AWS CodeDeploy para gerenciar a estratégia de Blue-Green Deployment.

## Passo 2: Criar o Workflow no GitHub Actions

```yaml
name: Blue-Green Deployment

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Baixar código
        uses: actions/checkout@v2
        # Baixa o código-fonte do repositório GitHub

      - name: Configurar .NET Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: '7.x'
        # Configura o ambiente .NET Core para a versão especificada (7.x)

      - name: Restaurar dependências
        run: dotnet restore src/MyApplication/MyApplication.csproj
        # Restaura as dependências do projeto .NET Core

      - name: Compilar o projeto
        run: dotnet build --configuration Release \
          src/MyApplication/MyApplication.csproj
        # Compila o projeto com a configuração de Release

      - name: Publicar o projeto
        run: dotnet publish --configuration Release \
          --output ./publish src/MyApplication/MyApplication.csproj
        # Publica o projeto no diretório de saída especificado

      - name: Criar imagem Docker
        run: |
          docker build -t myapplication:latest \
            -f src/MyApplication/Dockerfile .
        # Cria uma imagem Docker usando o Dockerfile da aplicação

      - name: Logar no AWS ECR
        uses: aws-actions/amazon-ecr-login@v1
        # Faz login no Amazon ECR (Elastic Container Registry)

      - name: Enviar imagem para o ECR
        run: |
          docker tag myapplication:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.\
          ${{ secrets.AWS_REGION }}.amazonaws.com/myapplication:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.\
          ${{ secrets.AWS_REGION }}.amazonaws.com/myapplication:latest
        # Marca a imagem Docker com a tag apropriada e a envia para o Amazon ECR

  deploy-green:
    runs-on: ubuntu-latest
    needs: build  # Este job depende do sucesso do job "build"

    steps:
      - name: Logar no AWS CLI
        run: |
          aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws configure set region ${{ secrets.AWS_REGION }}
        # Configura as credenciais AWS para uso do CLI

      - name: Criar novo ambiente no Elastic Beanstalk (Green)
        run: |
          aws elasticbeanstalk create-environment \
            --application-name "MyApplication" \
            --environment-name "MyApp-Green" \
            --solution-stack-name "64bit Amazon Linux 2 v3.2.6 running Docker" \
            --version-label "v1" \
            --option-settings Namespace=aws:elasticbeanstalk:environment:process,\
            OptionName=Dockerrun.aws.json,Value=file://deploy/Dockerrun.aws.json
        # Cria um novo ambiente no Elastic Beanstalk chamado "MyApp-Green",
        # usando a imagem Docker publicada

      - name: Publicar a aplicação no ambiente Green
        run: |
          aws elasticbeanstalk update-environment \
            --environment-name "MyApp-Green" \
            --version-label "v1"
        # Publica a nova versão da aplicação no ambiente Green do Elastic Beanstalk

      - name: Validar ambiente Green
        run: |
          # Execute seus testes de validação aqui
          # Exemplo: Testes de integração, testes de fumaça, etc.
          curl -f https://myapp-green.elasticbeanstalk.com/health || exit 1
        # Valida se o ambiente Green está funcionando corretamente usando um teste
        # simples de saúde

  swap-environments:
    runs-on: ubuntu-latest
    needs: deploy-green  # Este job depende do sucesso do job "deploy-green"

    steps:
      - name: Logar no AWS CLI
        run: |
          aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws configure set region ${{ secrets.AWS_REGION }}
        # Configura as credenciais AWS para uso do CLI

      - name: Swap do ambiente Green para Blue
        run: |
          aws elasticbeanstalk swap-environment-cnames \
            --source-environment-name "MyApp-Green" \
            --destination-environment-name "MyApp-Blue"
        # Troca os CNAMEs do ambiente Green e Blue, fazendo o ambiente Green se
        # tornar o ambiente de produção

      - name: Excluir o antigo ambiente Blue
        run: |
          aws elasticbeanstalk terminate-environment \
            --environment-name "MyApp-Blue"
        # Exclui o antigo ambiente Blue após a validação do ambiente Green
```

## Passo 3: Atualização do Arquivo `Dockerrun.aws.json`

```json
{
  "AWSEBDockerrunVersion": 2,
  "containerDefinitions": [
    {
      "name": "myapplication",
      "image": "<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/myapplication:latest",
      "essential": true,
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ]
    }
  ]
}
```

Explicações Detalhadas

	•	“AWSEBDockerrunVersion”: 2:
	•	  Esta linha define a versão do formato Dockerrun.aws.json que está sendo usada. A versão 2 é necessária para configurar aplicativos Docker no AWS Elastic Beanstalk.
	•	“containerDefinitions”:
	•	  Esta seção define uma lista de contêineres Docker que o Elastic Beanstalk deve iniciar. Mesmo que você esteja usando um único contêiner, ele deve ser definido dentro de containerDefinitions.
	•	“name”: “myapplication”:
	•	  Este campo define o nome do contêiner. Neste exemplo, o nome do contêiner é "myapplication". Esse nome pode ser usado para identificar o contêiner em logs e outros lugares no Elastic Beanstalk.
	•	“image”: “<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/myapplication:latest”:
	•	  Este campo especifica a imagem Docker que será usada para iniciar o contêiner. O valor <AWS_ACCOUNT_ID> deve ser substituído pelo ID da sua conta AWS, e <AWS_REGION> deve ser substituído pela região onde o ECR (Elastic Container Registry) está localizado.
	•	O :latest indica que a versão mais recente da imagem será utilizada. Se você tiver versões específicas, pode alterá-la conforme necessário.
	•	“essential”: true:
	•	  Define se o contêiner é essencial para o funcionamento do ambiente. Se for definido como true (verdadeiro), o Elastic Beanstalk garantirá que esse contêiner esteja sempre em execução. Se o contêiner parar ou falhar, o ambiente será considerado com falha e tentará reiniciar o contêiner.
	•	“memory”: 512:
	•	  Este campo especifica a quantidade de memória (em MB) que será atribuída ao contêiner. Neste exemplo, o contêiner receberá 512 MB de memória.
	•	“portMappings”:
	•	  Esta seção mapeia as portas do contêiner Docker para as portas do host no Elastic Beanstalk.
	•	“containerPort”: 80:
	•	  A porta 80 dentro do contêiner está sendo exposta. Essa é a porta padrão para tráfego HTTP.
	•	“hostPort”: 80:
	•	  A porta 80 do host (máquina virtual no Elastic Beanstalk) será mapeada para a porta 80 do contêiner. Isso permite que o tráfego HTTP que chega ao host seja redirecionado para a porta 80 do contêiner, onde sua aplicação está em execução.

## Passo 4: Configurar Secrets no GitHub

Para garantir que o fluxo de trabalho funcione corretamente, você precisará configurar algumas variáveis de ambiente e segredos no repositório do GitHub. Essas variáveis são essenciais para permitir que o GitHub Actions se conecte à sua conta AWS e provisione a infraestrutura necessária. Siga os passos abaixo para configurar os segredos e variáveis:

Passo 4.11: Acessar a Configuração de Segredos

	1.	No seu repositório GitHub, clique em Settings (Configurações) no menu superior.
	2.	No menu lateral, clique em Secrets and variables e, em seguida, em Actions.

Passo 4.2: Adicionar Segredos

Adicione os seguintes segredos:

	•	AWS_ACCESS_KEY_ID: A chave de acesso da sua conta AWS. Esta chave é usada para autenticar o GitHub Actions na AWS.
	•	Exemplo: AKIAIOSFODNN7EXAMPLE
	•	AWS_SECRET_ACCESS_KEY: A chave secreta associada à sua chave de acesso AWS. Esta chave também é usada para autenticar o GitHub Actions na AWS.
	•	Exemplo: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
	•	AWS_REGION: A região da AWS onde você deseja provisionar os recursos. Isso é importante para garantir que todos os recursos sejam criados na região correta.
	•	Exemplo: us-east-1
	•	AWS_ACCOUNT_ID: O ID da sua conta AWS. Esse ID é usado para construir o caminho para o Amazon Elastic Container Registry (ECR) onde as imagens Docker serão armazenadas.
	•	Exemplo: 123456789012

Passo 4.3: Adicionar Variáveis de Ambiente Específicas da Infraestrutura

Além dos segredos mencionados acima, você também precisará configurar variáveis específicas da infraestrutura que serão usadas nos jobs de provisionamento. Adicione os seguintes segredos:

	•	KEY_NAME: O nome do par de chaves SSH usado para acessar suas instâncias EC2. Esse par de chaves deve estar previamente criado na AWS.
	•	Exemplo: my-ec2-keypair
	•	SECURITY_GROUP_IDS: Os IDs dos grupos de segurança associados às instâncias EC2. Os grupos de segurança controlam o tráfego de rede permitido para suas instâncias.
	•	Exemplo: sg-0a1b2c3d4e5f6g7h8
	•	SUBNET_ID: O ID da sub-rede onde as instâncias EC2 serão lançadas. A sub-rede deve estar associada ao VPC onde seus recursos estão configurados.
	•	Exemplo: subnet-123abc45
	•	VPC_ID: O ID do Virtual Private Cloud (VPC) onde os recursos de rede, como sub-redes e grupos de segurança, estão configurados. É essencial para garantir que todos os recursos sejam criados no ambiente correto.
	•	Exemplo: vpc-0a1b2c3d4e5f6g7h8
	•	LOAD_BALANCER_ARN: O ARN (Amazon Resource Name) do Load Balancer que será usado para distribuir o tráfego entre as versões antiga e nova da aplicação durante o Canary Deployment.
	•	Exemplo: arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188
	•	LISTENER_ARN: O ARN do Listener associado ao Load Balancer, que gerencia o tráfego de entrada e o distribui para os Target Groups. Este ARN é necessário para modificar a distribuição de tráfego durante o Canary Deployment.
	•	Exemplo: arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-load-balancer/50dc6c495c0c9188/abcd1234efgh5678
	•	TARGET_GROUP_ARN_A: O ARN do Target Group A, que representa a versão antiga da aplicação.
	•	Exemplo: arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-target-group-a/50dc6c495c0c9188
	•	TARGET_GROUP_ARN_B: O ARN do Target Group B, que representa a nova versão da aplicação.
	•	Exemplo: arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-target-group-b/60ef7g8h9i0jklmnop


