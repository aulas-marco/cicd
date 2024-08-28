
# Rolling Update e Canary Deployment com GitHub Actions e AWS Elastic Beanstalk

Este guia descreve como configurar um fluxo de trabalho no GitHub Actions que realiza um Rolling Update ou Canary Deployment de uma aplicação .NET Core na AWS usando o Elastic Beanstalk. Além disso, automatiza o provisionamento das instâncias EC2 e a configuração dos Target Groups e Load Balancer.

## O que é Rolling Update e Canary Deployment?

### Rolling Update

O Rolling Update é uma estratégia de implantação que difere do Blue/Green Deployment na medida em que não requer duas infraestruturas idênticas. Em vez disso, o deployment para uma nova versão é feito dentro da infraestrutura atual clusterizada, onde a versão antiga ainda está sendo executada. 

### Como Funciona?

1. **Substituição Gradual:** Uma pequena porcentagem da versão da aplicação é substituída primeiro. Se tudo correr bem, essa porcentagem é gradualmente aumentada até que todas as instâncias estejam executando a nova versão.

2. **Infraestrutura Atual:** A nova versão é implantada diretamente na infraestrutura existente, sem necessidade de configurar uma nova infraestrutura em paralelo.

### Canary Deployment

O Canary Deployment é semelhante ao Rolling Update, com a diferença de que, no Canary Deployment, uma pequena porcentagem dos usuários é direcionada para a nova versão da aplicação, enquanto a maioria dos usuários continua a usar a versão antiga. 

### Como Funciona?

1. **Testes com Usuários Reais:** Um pequeno grupo de usuários é direcionado para a nova versão da aplicação, permitindo que a nova versão seja testada em um ambiente real antes de ser disponibilizada para todos os usuários.

2. **Aumento Gradual:** Se a nova versão se mostrar estável e confiável, mais usuários são direcionados para a nova versão até que ela substitua completamente a versão antiga.

Como ambas as estratégias são semelhantes e focadas principalmente em testar a estabilidade e confiabilidade de uma mudança, elas podem ser usadas de forma intercambiável.

```
Usuários  --->  [AmbienteOriginal]
                 |
                 |
            +---------------+
            | Load Balancer  |
            +---------------+
                 |
                 |
                 v
+---------------+
| Servidor 1    |
| Aplicação     |
+---------------+

Usuários  --->  [Acessos]
                 |
                 |
            +---------------+
            | Load Balancer  |
            +---------------+
                 |          |
                 v          v
+---------------+     +---------------+
| Servidor 1    |     | AmbienteCanario|
| Aplicação     |     | (2% do Tráfego)|
+---------------+     +---------------+
                            |
                            |
                            v
                     +---------------+
                     | Servidor 2    |
                     | Aplicação     |
                     +---------------+

Usuários  --->  [Acessos]
                 |
                 |
            +---------------+
            | Load Balancer  |
            +---------------+
                 |          |
                 v          v
+---------------+     +---------------+
| Servidor 1    |     | AmbienteCanario|
| Aplicação     |     | (100% do Tráfego)|
+---------------+     +---------------+
                            |
                            |
                            v
                     +---------------+
                     | Servidor 2    |
                     | Aplicação     |
                     +---------------+


```

## Passo 1: Estrutura do Repositório

Certifique-se de que o repositório tenha a seguinte estrutura básica:

```
your-repo/
├── .github/
│   └── workflows/
│       └── rolling-canary-deployment.yml
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

- **.github/workflows/rolling-canary-deployment.yml**: Contém o workflow do GitHub Actions que automatiza o processo de construção, teste e implantação da aplicação .NET Core usando Rolling Update ou Canary Deployment na AWS.
- **src/MyApplication/**: Diretório contendo o código-fonte da aplicação .NET Core.
- **deploy/Dockerrun.aws.json**: Arquivo de configuração para o AWS Elastic Beanstalk, especificando como o Docker deve ser executado.
- **deploy/appspec.yaml**: Arquivo de configuração do AWS CodeDeploy para gerenciar a estratégia de Rolling Update ou Canary Deployment.

## Passo 2: Automatizar o Provisionamento de Instâncias e Target Groups

### Provisionamento Automatizado

```yaml
name: Rolling Update/Canary Deployment

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
        run: dotnet build --configuration Release src/MyApplication/MyApplication.csproj
        # Compila o projeto com a configuração de Release

      - name: Publicar o projeto
        run: dotnet publish --configuration Release --output ./publish src/MyApplication/MyApplication.csproj
        # Publica o projeto no diretório de saída especificado

      - name: Criar imagem Docker
        run: |
          docker build -t myapplication:latest -f src/MyApplication/Dockerfile .
        # Cria uma imagem Docker usando o Dockerfile da aplicação

      - name: Logar no AWS ECR
        uses: aws-actions/amazon-ecr-login@v1
        # Faz login no Amazon ECR (Elastic Container Registry)

      - name: Enviar imagem para o ECR
        run: |
          docker tag myapplication:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/myapplication:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/myapplication:latest
        # Marca a imagem Docker com a tag apropriada e a envia para o Amazon ECR

# Job para provisionamento da infraestrutura na AWS
provision-infrastructure:
  runs-on: ubuntu-latest
  needs: build  # Este job depende do sucesso do job "build"

  steps:
    - name: Criar e Configurar Instâncias EC2
      run: |
        # Criar duas instâncias no Target Group A
        # As instâncias são baseadas na AMI "ami-0c55b159cbfafe1f0" (Amazon Linux 2)
        # Tipo de instância é "t2.micro" e outras configurações são obtidas das variáveis secretas
        INSTANCE_IDS_A=$(aws ec2 run-instances \
          --image-id ami-0c55b159cbfafe1f0 \
          --count 2 \
          --instance-type t2.micro \
          --key-name ${{ secrets.KEY_NAME }} \
          --security-group-ids ${{ secrets.SECURITY_GROUP_IDS }} \
          --subnet-id ${{ secrets.SUBNET_ID }} \
          --query 'Instances[*].InstanceId' \
          --output text)
        
        # Criar duas instâncias no Target Group B
        INSTANCE_IDS_B=$(aws ec2 run-instances \
          --image-id ami-0c55b159cbfafe1f0 \
          --count 2 \
          --instance-type t2.micro \
          --key-name ${{ secrets.KEY_NAME }} \
          --security-group-ids ${{ secrets.SECURITY_GROUP_IDS }} \
          --subnet-id ${{ secrets.SUBNET_ID }} \
          --query 'Instances[*].InstanceId' \
          --output text)
        
        # Criar Target Group A (versão antiga)
        # O Target Group A será usado para a versão antiga da aplicação
        TARGET_GROUP_ARN_A=$(aws elbv2 create-target-group \
          --name my-app-target-group-a \
          --protocol HTTP \
          --port 80 \
          --vpc-id ${{ secrets.VPC_ID }} \
          --target-type instance \
          --query 'TargetGroups[0].TargetGroupArn' \
          --output text)

        # Criar Target Group B (versão nova)
        # O Target Group B será usado para a nova versão da aplicação
        TARGET_GROUP_ARN_B=$(aws elbv2 create-target-group \
          --name my-app-target-group-b \
          --protocol HTTP \
          --port 80 \
          --vpc-id ${{ secrets.VPC_ID }} \
          --target-type instance \
          --query 'TargetGroups[0].TargetGroupArn' \
          --output text)

        # Registrar instâncias no Target Group A
        # As instâncias criadas anteriormente são registradas no Target Group A
        aws elbv2 register-targets \
          --target-group-arn $TARGET_GROUP_ARN_A \
          --targets Id=${INSTANCE_IDS_A// / Id=}

        # Registrar instâncias no Target Group B
        # As instâncias criadas anteriormente são registradas no Target Group B
        aws elbv2 register-targets \
          --target-group-arn $TARGET_GROUP_ARN_B \
          --targets Id=${INSTANCE_IDS_B// / Id=}

        # Criar Listener no ELB para distribuir tráfego entre A e B
        # Um Listener é criado para distribuir o tráfego HTTP na porta 80
        # Inicialmente, todo o tráfego é direcionado para o Target Group A
        aws elbv2 create-listener \
          --load-balancer-arn ${{ secrets.LOAD_BALANCER_ARN }} \
          --protocol HTTP \
          --port 80 \
          --default-actions \
            Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_A,Weight=100 \
            Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_B,Weight=0

# Job para implantação no Elastic Beanstalk
deploy-to-elastic-beanstalk:
  runs-on: ubuntu-latest
  needs: provision-infrastructure  # Este job depende do sucesso do job "provision-infrastructure"

  steps:
    - name: Configurar Elastic Beanstalk para usar a imagem Docker
      run: |
        aws elasticbeanstalk create-environment \
          --application-name "MyApplication" \
          --environment-name "MyApp-Environment" \
          --solution-stack-name "64bit Amazon Linux 2 v3.2.6 running Docker" \
          --version-label "v1" \
          --option-settings Namespace=aws:elasticbeanstalk:environment:process,OptionName=Dockerrun.aws.json,Value=file://deploy/Dockerrun.aws.json
      # Este comando cria um novo ambiente no AWS Elastic Beanstalk usando a imagem Docker
      # previamente enviada para o Amazon ECR. O ambiente é configurado com as opções
      # especificadas, incluindo o uso do arquivo Dockerrun.aws.json.

# Job para iniciar o Canary Deployment com 2% do tráfego
canary-deploy-2-percent:
  runs-on: ubuntu-latest
  needs: provision-infrastructure  # Este job depende do sucesso do job "provision-infrastructure"

  steps:
    - name: Iniciar Canary Deployment com 2% do Tráfego
      run: |
        aws elbv2 modify-listener --listener-arn ${{ secrets.LISTENER_ARN }} \
        --default-actions Type=forward,TargetGroupArn=${{ secrets.TARGET_GROUP_ARN_A }},Weight=98 \
                          Type=forward,TargetGroupArn=${{ secrets.TARGET_GROUP_ARN_B }},Weight=2 
      # Modifica o Listener no ELB para redirecionar 2% do tráfego para o Target Group B (nova versão)
      # e 98% do tráfego continua direcionado para o Target Group A (versão antiga).

# Job para aumentar o Canary Deployment para 50% do tráfego
canary-deploy-50-percent:
  runs-on: ubuntu-latest
  needs: canary-deploy-2-percent  # Este job depende do sucesso do job "canary-deploy-2-percent"

  steps:
    - name: Aumentar Canary Deployment para 50% do Tráfego
      run: |
        aws elbv2 modify-listener --listener-arn ${{ secrets.LISTENER_ARN }} \
        --default-actions Type=forward,TargetGroupArn=${{ secrets.TARGET_GROUP_ARN_A }},Weight=50 \
                          Type=forward,TargetGroupArn=${{ secrets.TARGET_GROUP_ARN_B }},Weight=50 
      # Modifica o Listener no ELB para redirecionar 50% do tráfego para o Target Group B (nova versão)
      # e 50% do tráfego para o Target Group A (versão antiga).

# Job para completar o Canary Deployment com 100% do tráfego
canary-deploy-100-percent:
  runs-on: ubuntu-latest
  needs: canary-deploy-50-percent  # Este job depende do sucesso do job "canary-deploy-50-percent"

  steps:
    - name: Completar Canary Deployment com 100% do Tráfego
      run: |
        aws elbv2 modify-listener --listener-arn ${{ secrets.LISTENER_ARN }} \
        --default-actions Type=forward,TargetGroupArn=${{ secrets.TARGET_GROUP_ARN_B }},Weight=100
      # Modifica o Listener no ELB para redirecionar 100% do tráfego para o Target Group B (nova versão),
      # completando assim o Canary Deployment.
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

