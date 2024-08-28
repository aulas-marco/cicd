Vocabulário CI/CD
    
•	Alvo/ambiente-alvo: Ambiente onde um artefato é implantado. Alvo e ambiente-alvo não são usados no contexto de uma plataforma ALM e/ou ferramentas de CI/CD relacionadas. O ambiente-alvo pode ser um servidor de testes ou de produção. Também pode ser uma conta/assinatura em um provedor de serviços em nuvem.
•	Ambiente: Na maioria dos casos, refere-se à plataforma/infraestrutura onde o artefato é implantado. Em alguns casos, é usado no contexto de uma plataforma ALM e/ou ferramentas de CI/CD relacionadas.
•	Análise de código: Esta é uma subárea de garantia de qualidade que inclui varreduras de código estático para determinar a qualidade do código e detectar vulnerabilidades no código da aplicação e suas dependências.
•	Artefato (Build): Refere-se à combinação de código-fonte e suas dependências para criar um produto executável.
•	Artefato de lançamento - Criação de um lançamento candidato.
•	Artefato: Um artefato é um pacote armazenado em um repositório binário e usado para implantação em um ambiente-alvo.
•	Ciclo de vida de desenvolvimento de software (SDLC): Processo de desenvolvimento de software, build, teste e implantação.
•	Controle duplo: Aplicação do princípio dos quatro olhos, onde uma pessoa realiza uma tarefa e outra deve aprovar a execução dessa tarefa.
•	Empacotar (verbo): Após a construção de uma aplicação, ela é empacotada de forma a torná-la facilmente transportável, como um arquivo .zip ou .jar. No caso de produtos comerciais prontos (COTS), esta etapa geralmente é pulada do ponto de vista do consumidor, pois o produto já está em um formato transportável. Empacotamento também implica em definir uma linha de base do artefato, garantindo que o que é implantado em produção seja o artefato correto (testado e sem compromissos).
•	Estágio: Grupo de atividades relacionadas em uma pipeline de CI/CD. Exemplos de estágios são Executar build, Analisar código e Realizar teste.
•	Garantia de qualidade: Processo que assegura que a qualidade do seu produto atinge um determinado nível.
•	Gerenciamento de controle de versão (SCM): Sistema para realizar controle de versão. Exemplos de sistemas SCM são Git, Mercurial e Subversion.
•	Gerenciamento do ciclo de vida de aplicações (ALM): Este conjunto de ferramentas integradas abrange os principais aspectos da cadeia de suprimentos de software.
•	Implantação contínua: Processo no qual o artefato é construído, testado e implantado em produção de forma automatizada. A pipeline valida se o artefato atende a todos os critérios de qualidade, diferindo da entrega contínua, onde é necessário um passo manual adicional (controle duplo).
•	Implantar (Deploy): Significa instalar um artefato em um determinado ambiente-alvo, que pode ser um ambiente de teste ou de produção.
•	Lançamento (Release) ou versão de lançamento: Artefato que pode ser implantado em um ambiente de produção.
•	Lançamento candidato: Artefato ao qual não são mais adicionadas novas funcionalidades. Apenas correções de bugs são feitas em um lançamento candidato, e ele se torna um novo lançamento candidato, mas com uma versão diferente. Se todos os bugs forem corrigidos e o lançamento candidato passar em todos os testes, ele é promovido a um lançamento(substantivo).
•	Lançar (verbo): Ações realizadas para implantar um artefato em um ambiente de produção.
•	Pacote (substantivo): É o artefato, construído pelo servidor de integração ou um arquivo baixado de um fornecedor no caso de uma aplicação COTS.
•	Pipeline: Design e implementação de todos os passos que definem a automação do processo de entrega de software. Pode ser uma pipeline de CI/CD real ou uma pipeline com um caráter menos contínuo.
•	Plataforma ALM: Um servidor de integração ou um conjunto de ferramentas de CI e CD individuais, mas integradas. A plataforma ALM cobre toda a cadeia de ferramentas de CI/CD.
•	Promover: Esta atividade “promove” um lançamento candidato para um lançamento (release). O lançamento pode ser implantado em um ambiente de produção.
•	Publicar: Após a aplicação ser empacotada, ela é armazenada em um repositório binário imutável, como Artifactory ou Nexus. Pacotes baixados de fornecedores (COTS) ainda precisam ser publicados em um local seguro dentro da organização para garantir a integridade e rastreabilidade.
•	Ramo (Branch): Este é um ramo usado no gerenciamento de controle de versão.
•	Rótulo (Tag) é usada para identificar um grupo. Esse grupo pode consistir em código, artefatos, estágios, recursos-alvo, entre outros. Um rótulo é frequentemente usado para identificar um release (candidato), com todo o código relacionado, estágio(s) de CI/CD e recursos-alvo.
•	Tarefa: Atividade dentro de um estágio. Um estágio de testes, por exemplo, pode consistir em diferentes tipos de testes, como um teste de regressão ou um teste de pré-produção, cada um realizado como uma tarefa.
•	Teste: Conjunto de garantia de qualidade que envolve testes automatizados e testes manuais.
•	Tronco: Ramo principal em um sistema de gerenciamento de controle de versão. Também é chamado de linha principal (mainline) ou linha mestre (master).
•	Versionamento: A versão refere-se a uma instância específica do artefato, e a tag é aplicada a um grupo do qual o artefato faz parte. A versão e a tag podem ter o mesmo valor, mas isso não é obrigatório. Uma tag pode se referir a um recurso de produto associado a vários candidatos a release, cada um com sua versão.
