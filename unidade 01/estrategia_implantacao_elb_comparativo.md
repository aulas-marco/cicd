
# Documentação: Uso do Elastic Load Balancer (ELB) em Blue-Green Deployment e Canary Deployment

## Introdução

O Elastic Load Balancer (ELB) é um serviço essencial da AWS que distribui automaticamente o tráfego de entrada de aplicativos em várias instâncias, aumentando a tolerância a falhas dos aplicativos implantados na AWS. Embora o ELB seja usado em várias estratégias de implantação, como **Blue-Green Deployment** e **Canary Deployment**, a maneira como ele é configurado e gerenciado varia dependendo da abordagem adotada.

Esta documentação explora como o ELB é utilizado e configurado em duas abordagens diferentes: **Blue-Green Deployment** e **Canary Deployment**, ambas utilizando o **Elastic Beanstalk** como plataforma.

## Blue-Green Deployment e ELB

### Visão Geral

O **Blue-Green Deployment** é uma técnica de implantação que minimiza o tempo de inatividade e reduz os riscos associados ao lançamento de novas versões de uma aplicação. O conceito envolve ter dois ambientes idênticos, chamados "Blue" e "Green". O **Elastic Beanstalk** é utilizado para gerenciar automaticamente esses ambientes, incluindo a configuração e o gerenciamento do Elastic Load Balancer (ELB).

### Como o ELB é Utilizado no Blue-Green Deployment

1. **Automação e Abstração**:
   - O Elastic Beanstalk automatiza a criação e configuração do ELB, incluindo Target Groups e Listeners.
   - Quando você realiza um **Blue-Green Deployment**, o Elastic Beanstalk gerencia a troca (swap) dos CNAMEs entre os ambientes Blue e Green. Isso permite que o tráfego de produção seja redirecionado para o novo ambiente sem tempo de inatividade, tudo isso sem a necessidade de intervenção manual.

2. **Configuração Simplificada**:
   - Como o Elastic Beanstalk gerencia o ELB automaticamente, os desenvolvedores podem se concentrar mais no desenvolvimento e menos na configuração da infraestrutura. 
   - A configuração do ELB, incluindo Target Groups e Listeners, é abstraída, o que simplifica o processo de implantação.

### Quando Usar Blue-Green Deployment

- **Automação e Simplicidade**:
  - Ideal para equipes que desejam implantar aplicações rapidamente sem se preocupar com a infraestrutura subjacente.
  - A escolha perfeita para Blue-Green Deployment, onde a troca entre os ambientes pode ser feita de forma rápida e automatizada.

### Exemplo de Uso no Blue-Green Deployment

```yaml
deploy-green:
  runs-on: ubuntu-latest
  needs: build

  steps:
    - name: Criar novo ambiente no Elastic Beanstalk (Green)
      run: |
        aws elasticbeanstalk create-environment \
          --application-name "MyApplication" \
          --environment-name "MyApp-Green" \
          --solution-stack-name "64bit Amazon Linux 2 v3.2.6 running Docker" \
          --version-label "v1" \
          --option-settings \
            Namespace=aws:elasticbeanstalk:environment:process, \
            OptionName=Dockerrun.aws.json,Value=file://deploy/Dockerrun.aws.json
```

- **Explicação**: Este exemplo mostra como um novo ambiente Green é criado no Elastic Beanstalk, utilizando uma imagem Docker. O ELB é automaticamente gerenciado pelo Elastic Beanstalk.

## Canary Deployment e ELB

### Visão Geral

O **Canary Deployment** é uma estratégia de implantação progressiva que permite que novas versões de uma aplicação sejam implantadas de forma gradual, redirecionando uma pequena porcentagem do tráfego de produção para a nova versão enquanto o restante continua a ser atendido pela versão antiga. Isso oferece a oportunidade de monitorar e validar a nova versão com um risco mínimo. Assim como no Blue-Green Deployment, o **Elastic Beanstalk** pode ser utilizado, mas o controle granular sobre o ELB é feito manualmente.

### Como o ELB é Utilizado no Canary Deployment

1. **Controle Granular**:
   - No Canary Deployment, embora você esteja utilizando o Elastic Beanstalk, precisa de controle manual sobre o ELB, incluindo a configuração de Target Groups e Listeners.
   - Você precisa criar e configurar manualmente Target Groups para diferentes versões da aplicação (A e B), e então usar Listeners no ELB para controlar a distribuição de tráfego entre esses grupos.

2. **Implementação Gradual**:
   - No Canary Deployment, o tráfego é gradualmente redirecionado para o novo Target Group (nova versão da aplicação). Esse redirecionamento é feito ajustando os pesos nos Listeners do ELB, permitindo que uma pequena porcentagem de tráfego seja enviada para a nova versão enquanto o restante continua a ser direcionado para a versão antiga.
   - À medida que a confiança na nova versão aumenta, o peso do tráfego pode ser gradualmente ajustado até que 100% do tráfego esteja direcionado para a nova versão.

### Quando Usar Canary Deployment

- **Controle e Flexibilidade**:
  - Ideal para implantações que exigem um alto grau de controle sobre como o tráfego é gerenciado e redirecionado.
  - Perfeito para situações em que você deseja monitorar a nova versão da aplicação em produção com um risco minimizado.

### Exemplo de Uso no Canary Deployment

```yaml
provision-infrastructure:
  runs-on: ubuntu-latest
  needs: build

  steps:
    - name: Criar e Configurar Instâncias EC2
      run: |
        INSTANCE_IDS_A=$(aws ec2 run-instances \
          --image-id ami-0c55b159cbfafe1f0 \
          --count 2 \
          --instance-type t2.micro \
          --key-name ${{ secrets.KEY_NAME }} \
          --security-group-ids ${{ secrets.SECURITY_GROUP_IDS }} \
          --subnet-id ${{ secrets.SUBNET_ID }} \
          --query 'Instances[*].InstanceId' \
          --output text)
        
        INSTANCE_IDS_B=$(aws ec2 run-instances \
          --image-id ami-0c55b159cbfafe1f0 \
          --count 2 \
          --instance-type t2.micro \
          --key-name ${{ secrets.KEY_NAME }} \
          --security-group-ids ${{ secrets.SECURITY_GROUP_IDS }} \
          --subnet-id ${{ secrets.SUBNET_ID }} \
          --query 'Instances[*].InstanceId' \
          --output text)
        
        TARGET_GROUP_ARN_A=$(aws elbv2 create-target-group \
          --name my-app-target-group-a \
          --protocol HTTP \
          --port 80 \
          --vpc-id ${{ secrets.VPC_ID }} \
          --target-type instance \
          --query 'TargetGroups[0].TargetGroupArn' \
          --output text)

        TARGET_GROUP_ARN_B=$(aws elbv2 create-target-group \
          --name my-app-target-group-b \
          --protocol HTTP \
          --port 80 \
          --vpc-id ${{ secrets.VPC_ID }} \
          --target-type instance \
          --query 'TargetGroups[0].TargetGroupArn' \
          --output text)

        aws elbv2 register-targets \
          --target-group-arn $TARGET_GROUP_ARN_A \
          --targets Id=${INSTANCE_IDS_A// / Id=}

        aws elbv2 register-targets \
          --target-group-arn $TARGET_GROUP_ARN_B \
          --targets Id=${INSTANCE_IDS_B// / Id=}

        aws elbv2 create-listener \
          --load-balancer-arn ${{ secrets.LOAD_BALANCER_ARN }} \
          --protocol HTTP \
          --port 80 \
          --default-actions \
            Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_A,Weight=100 \
            Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_B,Weight=0
```

- **Explicação**:  

* Este exemplo demonstra como configurar manualmente o ELB, Target Groups, e Listeners para realizar um Canary Deployment. O tráfego é inicialmente roteado 100% para o Target Group A (versão antiga), com o Target Group B (nova versão) aguardando para receber gradualmente mais tráfego.


* Embora o ELB seja utilizado em ambas as abordagens (Blue-Green Deployment e Canary Deployment), a principal diferença está no nível de controle e automação. No **Blue-Green Deployment** utilizando Elastic Beanstalk, o ELB é configurado e gerenciado automaticamente, proporcionando uma experiência simplificada. No **Canary Deployment**, mesmo utilizando o Elastic Beanstalk, você precisa configurar manualmente o ELB para permitir uma implantação gradual e controlada. 
