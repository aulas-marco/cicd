# Etapa 1: Descrição do Pipeline Complexo como um Fluxo

## Contexto

### Você é o DevOps/SRE responsável por configurar um pipeline de CI/CD para a empresa fictícia ACME, uma plataforma de e-commerce altamente escalável. A ACME possui uma arquitetura de microserviços, todos gerenciados em um monorepo no GitHub. A empresa adota a estratégia GitFlow para ramificação, com as branches principais main, develop, e release, e utiliza tags de versão para realizar releases. O ambiente de produção está hospedado na AWS, e a infraestrutura é provisionada utilizando Terraform. O pipeline deve garantir segurança robusta, incluindo a rotação automática de segredos e a publicação de artefatos com controle de versão.

Requisitos do Pipeline

# Guia de Implementação de Pipeline CI/CD - ACME

## 1. Branches Diferentes, Ações Diferentes

- **Branch `develop`:** 
  - Executa testes unitários, de integração e performance. 
  - Se todos forem bem-sucedidos, constrói os artefatos Docker para cada microserviço.

- **Branch `release`:** 
  - Executa todos os testes.
  - Faz o provisionamento da infraestrutura usando Terraform.
  - Publica os artefatos Docker em um registro privado na AWS ECR.

- **Branch `main`:** 
  - Executa todos os testes e o build.
  - Realiza o deploy automatizado para o ambiente de produção, utilizando segredos seguros.
  - Envia notificações aos stakeholders.

## 2. Tags de Versão

- **Exemplo: `v1.0.0`:**
  - Executa o build e o deploy para produção.
  - Publica a documentação.
  - Roda testes de performance usando Apache Bench (ab) e arquiva os resultados.
  - Envia notificações automatizadas para stakeholders e atualiza dashboards de monitoramento.

## 3. Gerenciamento de Falhas

- Se qualquer teste falhar, o pipeline deve parar imediatamente e notificar a equipe.
- Se os testes forem bem-sucedidos, o build e os passos seguintes serão realizados.
- Se o provisionamento ou deploy falhar, o pipeline deve reverter as mudanças aplicadas e notificar a equipe.
- Sempre executar uma etapa de limpeza após o deploy, independentemente do sucesso ou falha das etapas anteriores.

## 4. Segurança

- Chaves de API, tokens de acesso e outros segredos utilizados no deploy e no provisionamento devem ser rotacionados automaticamente a cada 30 dias.
- Evitar a exposição de segredos em logs, utilizando variáveis temporárias e criptografia.
- Acesso controlado e monitoramento contínuo durante o pipeline.

## 5. Modularidade e Reutilização

- O pipeline deve ser modular, com workflows reutilizáveis para testes, build, provisionamento, e deploy, que podem ser compartilhados entre múltiplos microserviços.
- Snippets de código YAML devem ser organizados e reutilizáveis para facilitar a manutenção e escalabilidade.

## 6. Infraestrutura como Código (IaC)

- O provisionamento de ambientes de teste, staging e produção deve ser automatizado usando Terraform, garantindo consistência e repetibilidade.
- Todos os ambientes devem ser idempotentes, com validação de conformidade antes da aplicação de mudanças.

## 7. Performance e Monitoramento

- Após o deploy, rodar testes de performance com Apache Bench (ab) para validar a escalabilidade sob carga. 
- Resultados devem ser registrados e analisados.
- Atualizar dashboards de monitoramento automaticamente após cada release para refletir o estado atual do sistema.

## 8. Documentação e Comentários

- O YAML deve ser bem documentado, com comentários explicando as decisões e os propósitos dos diferentes trechos de código.
- As práticas de versionamento e geração de documentação devem ser seguidas rigorosamente.

## 9. Condicionais e Flexibilidade

- Utilizar condicionais (`if`) para gerenciar a execução de steps com base no status de passos anteriores, variáveis de ambiente, e outras condições dinâmicas, garantindo que o pipeline seja adaptável a diferentes contextos.


Este pipeline deve ser robusto, seguro e eficiente, e deve lidar com todas as complexidades de um sistema de CI/CD moderno. Os alunos precisarão aplicar suas habilidades em GitFlow, segurança, Terraform, Docker, e testes de performance para construir um pipeline que seja capaz de escalar com a organização.

# Uma implementação frágil é mostrada abaixo:

```yaml
name: Pipeline CI/CD Desorganizado - ACME

on:
  push:
    branches:
      - develop
      - release
      - main

  pull_request:
    branches:
      - develop
      - release
      - main

jobs:
  ci-cd:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Código
        uses: actions/checkout@v2

      - name: Instalar Dependências
        run: npm install

      - name: Executar Testes Unitários
        run: npm run test:unit

      - name: Executar Testes de Integração
        run: npm run test:integration

      - name: Executar Testes de Performance
        run: ab -n 1000 -c 10 http://localhost:3000/

      - name: Verificação de Código com SonarQube
        run: |
          sonar-scanner \
          -Dsonar.projectKey=acme_project \
          -Dsonar.sources=. \
          -Dsonar.host.url=http://localhost:9000 \
          -Dsonar.login=your_sonarqube_token

      - name: Construir Docker Image Microserviço 1
        run: docker build -t microservice1:latest ./microservice1

      - name: Construir Docker Image Microserviço 2
        run: docker build -t microservice2:latest ./microservice2

      - name: Testes de Segurança com Trivy no Microserviço 1
        run: trivy image --severity HIGH,CRITICAL microservice1:latest

      - name: Testes de Segurança com Trivy no Microserviço 2
        run: trivy image --severity HIGH,CRITICAL microservice2:latest

      - name: Provisionar Infraestrutura Staging
        run: |
          cd terraform/staging
          terraform init
          terraform apply -auto-approve

      - name: Publicar Docker Image para ECR
        run: |
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
          docker tag microservice1:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservice1:latest
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservice1:latest
          docker tag microservice2:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservice2:latest
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservice2:latest

      - name: Deploy para Produção
        run: |
          cd terraform/production
          terraform init
          terraform apply -auto-approve

      - name: Enviar Notificação
        run: echo "Deploy realizado com sucesso!" | mail -s "Deploy Completo" team@acme.com

      - name: Limpeza
        run: |
          docker rmi $(docker images -q)
          cd terraform/staging
          terraform destroy -auto-approve
```