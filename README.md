# Teste Técnico Philips

## 1. Introdução

O objetivo desta solução é estabelecer um Processo de CI/CD resiliente e eficiente para a aplicação C++, cumprindo todas as diretrizes da Prova Técnica. A arquitetura foi desenvolvida utilizando Jenkins Master/Agent em contêineres Docker, com Agents de execução dedicados para o ambiente C++17, garantindo isolamento e reprodutibilidade do build. A Pipeline Declarativa implementada oferece rastreabilidade completa e permite builds otimizados através de parâmetros de execução condicional. O foco está em qualidade imediata, onde a falha em qualquer etapa (checa de código ou teste) interrompe o fluxo, assegurando que apenas artefatos validados sejam entregues.

## 2. Stack

| Categoria | Tecnologia | Justificativa |
| :--- | :--- | :--- |
| Orquestração de CI/CD | Jenkins Pipeline | Foi escolhido como o motor central. A sintaxe Declarative assegura que o fluxo de CI/CD seja versionado e rastreável (Pipeline-as-Code), focando na estabilidade e na manutenibilidade do processo. |
| Configuração de CI | Jenkins Configuration as Code (JCasC) | O JCasC elimina a intervenção manual no Controller, definindo usuários, segurança e nós Agents via código (jenkins.yml), garantindo que o setup seja repetível e audível. |
| Infraestrutura como Código (IaC) | Docker Compose | É o método empregado para provisionar toda a infraestrutura de CI/CD (Controller + 2 Agents). A utilização de Docker Compose permite que o ambiente seja inicializado de forma consistente com um único comando. |
| Ambiente de Build | Agentes Docker Customizados | Foi criada uma imagem customizada (Dockerfile.agent) que inclui todas as dependências de C++, CMake e ferramentas de teste. Isso isola o ambiente de build, garantindo a compatibilidade exigida com o C++17 e a reprodutibilidade. |
| Arquitetura de CI | Jenkins Master/Agent (2 Agents) | Foi implementada a separação de responsabilidades. O Controller apenas orquestra, enquanto os dois Agents (label cpp-agent) realizam a execução. Isso aumenta a segurança, estabilidade e oferece redundância para o processamento de builds. |
| Automação de Build | Makefile | É utilizado como a interface padronizada para interagir com a aplicação C++. A pipeline apenas invoca comandos genéricos (make check, make), mantendo a lógica de compilação fora do Jenkinsfile. |

## 3. Arquitetura da Solução e Fluxo de CI

A arquitetura de CI foi projetada com base nos princípios de **Infraestrutura como Código (IaC)** e **Configuração como Código (CaC)** para garantir **escalabilidade e rastreabilidade**.

### 3.1. Visão Geral da Infraestrutura de CI

A infraestrutura é provisionada via **Docker Compose** (`jenkins-setup/docker-compose.yml`) e consiste nos seguintes componentes:

1.  **Jenkins Controller:** O core da orquestração. Sua configuração inicial é feita de forma declarativa e automatizada pelo **JCasC** (`jenkins.yml`), que define *security realm*, usuários, e a configuração inicial dos agentes.
2.  **Agentes de Execução (2x):** Dois Agentes *Inbound* (nomeados `agent1` e `agent2`) são configurados. Ambos possuem a *label* **`cpp-agent`** para receber tarefas da pipeline. Estes Agents rodam uma imagem **Docker customizada** que contém todas as dependências de **C++17** necessárias para o build e teste.

A segregação entre Controller e Agents garante que o Controller permaneça leve e focado em orquestração, enquanto os Agents lidam com o processamento intensivo do build C++.

### 3.2. Fluxo da Pipeline (`Jenkinsfile`)

A Pipeline Declarativa é acionada por três mecanismos:

1.  **Polling SCM:** Dispara o build a cada nova alteração no código (*push*).
2.  **Agendamento (`cron`):** Executa a pipeline diariamente, garantindo um artefato de *release* diário.
3.  **Gatilho Manual:** Permite a execução sob demanda, controlada por parâmetros.

O fluxo de trabalho no Agent (`cpp-agent`) segue uma abordagem de **Qualidade em Primeiro Lugar**:

1.  **Checkout:** Obtém o código-fonte do repositório.
2.  **Code Check (Opcional):** Executa validações de código (`make check`). Esta etapa, assim como os testes, é **condicional** e pode ser pulada via parâmetro de build (`SKIP_CHECKS_AND_TESTS`) para builds rápidos de desenvolvimento.
3.  **Build:** Compila a aplicação C++ (`make`).
4.  **Test (Opcional):** Executa os testes unitários (`make unittest`).
5.  **Archive Artifacts:** O binário final (artefato `calculator`) é arquivado no Jenkins e associado ao build, **garantindo a rastreabilidade** do código-fonte, dos testes e do artefato final. Esta etapa também é **condicional**, sendo executada apenas em builds agendados ou se solicitada via parâmetro (`GENERATE_ARTIFACT`).

A falha em qualquer uma das etapas críticas do fluxo (Build, Check ou Test) resulta na **interrupção imediata da pipeline**, impedindo a geração de artefatos não validados.

## 4. Pipeline CI/CD: Detalhamento do `Jenkinsfile`

A pipeline é implementada de forma Declarativa e é executada inteiramente em um Agente dedicado (`cpp-agent`), garantindo um ambiente C++17 isolado e consistente. O fluxo é desenhado com uma abordagem de **"Fail Fast"**, onde a falha em qualquer etapa crítica interrompe a execução.

### 4.1. Recursos de Controle e Otimização (Visão Sênior)

O pipeline foi configurado com recursos que aprimoram a usabilidade e a eficiência:

* **Agente Dedicado (`agent { label 'cpp-agent' }`):** Garante que o build sempre ocorra em um dos Agentes provisionados, que possuem as ferramentas **C++17, clang-tidy, e gtest** instaladas (validando o isolamento).
* **Parâmetros de Build Condicionais:**
    * `SKIP_CHECKS_AND_TESTS`: Permite um **build rápido de desenvolvimento** que pula as verificações de qualidade e testes longos.
    * `GENERATE_ARTIFACT`: Permite que o artefato seja gerado e arquivado **sob demanda**, além do agendamento diário.
* **Triggers Múltiplos:** Configuração com **`cron` (diário)** e **`pollSCM`** (disparo via push) para cobrir releases agendados e CI em desenvolvimento.

### 4.2. Fluxo das Stages

| Stage | Comando Principal | Objetivo e Status (Validado pelo Log) |
| :--- | :--- | :--- |
| **Checkout** | `checkout scm` | Clona o código. *Observação:* A execução foi confirmada no `agent2`. |
| **Code Check** | `make check` | **Qualidade de Código:** Executa **clang-tidy** e **clang-format**. A pipeline identificou violações de formatação, provando que a ferramenta de qualidade está ativa e pronta para falhar em builds futuros. |
| **Build** | `make` | **Compilação:** Geração do binário com **g++** e padrão **C++17**, validando a compatibilidade de ambiente. |
| **Test** | `make unittest` | **Testes Unitários:** Executa o *suite* de testes GTest, que foi **aprovado** sem falhas, garantindo a funcionalidade. |
| **Archive Artifacts** | `archiveArtifacts` | **Entrega Condicional:** Esta etapa foi **skipada** no build de teste (devido à lógica `when`), demonstrando que o parâmetro `GENERATE_ARTIFACT` ou o `cron` são requisitos para a entrega final, otimizando builds de CI. |
| **Post Actions** | `always { echo ... }` | Bloco final de notificação. |

## 5. Guia de Configuração e Execução

Este guia fornece as instruções necessárias para inicializar a infraestrutura de CI/CD (Jenkins + Agents) e executar a pipeline, validando todos os requisitos da prova.

### 5.1. Pré-requisitos

Para rodar a solução de forma local e reprodutível, os seguintes softwares são obrigatórios:

* **Git:** Para clonar o repositório.
* **Docker:** Para rodar os contêineres.
* **Docker Compose:** Para gerenciar e orquestrar os serviços definidos em `jenkins-setup/docker-compose.yml`.

### 5.2. Inicialização da Infraestrutura de CI (IaC)

Toda a infraestrutura do Jenkins é inicializada via IaC, utilizando o Docker Compose e a configuração é automatizada via JCasC.

1.  **Clone o Repositório:**
    ```bash
    git clone [https://github.com/NavesEdu/Test_DevOps.git](https://github.com/NavesEdu/Test_DevOps.git)
    cd Test_DevOps
    ```

2.  **Suba os Serviços de CI:**
    Navegue até o diretório de IaC e use o Docker Compose.
    ```bash
    cd jenkins-setup
    docker compose up -d
    ```
    *Resultado:* Esta ação constrói as imagens customizadas do **Controller** e dos **dois Agents** (`agent1` e `agent2`) e inicia os três serviços em segundo plano.
<img width="1087" height="287" alt="Image" src="https://github.com/user-attachments/assets/703443db-9d55-4eb6-adee-daf7395eec6f" />

3.  **Acesso ao Jenkins UI:**
    Após a inicialização (que pode levar alguns minutos), o Jenkins estará acessível e já configurado pelo JCasC.
    * **URL:** `http://localhost:8080`
    * **Credenciais (Usuário Admin):**
        * Usuário: `admin`
        * Senha: `admin`
<img width="2559" height="1355" alt="Image" src="https://github.com/user-attachments/assets/a6d9056c-425c-4d84-888e-3b9e8882e72f" />
<img width="2557" height="1363" alt="Image" src="https://github.com/user-attachments/assets/62108e51-8963-4f7d-87e1-5e654cf8d1da" />

### 5.3. Execução e Validação da Pipeline

A pipeline está pronta para ser executada no ambiente `cpp-agent` assim que o Jenkins for inicializado.

1.  **Não é necessário criar o Job, já que o JCasC é configurado pra isso**

2.  **Execute o Build (Testando os Parâmetros):**
    Use a opção **"Build With Parameters"** para validar as funcionalidades condicionais:
    * **Teste de Qualidade Total:** Deixe ambos os parâmetros (`SKIP_CHECKS_AND_TESTS` e `GENERATE_ARTIFACT`) como `false`. A pipeline deve executar **Code Check, Build e Test**.
    <img width="2552" height="720" alt="Image" src="https://github.com/user-attachments/assets/699c95f0-9138-40f3-8f7b-09c55a220f56" />
    <img width="2545" height="1357" alt="Image" src="https://github.com/user-attachments/assets/4cdb15d7-9c1d-4ce2-90e3-c0093e06ee7a" />
    
    * **Teste de Geração de Artefato Sob Demanda:** Marque apenas `GENERATE_ARTIFACT`. A pipeline deve executar Build e **Archive Artifacts**, simulando o gatilho manual.
    <img width="2552" height="789" alt="Image" src="https://github.com/user-attachments/assets/58768c3a-23b5-4613-bf97-1792bc4dd52c" />
    <img width="2559" height="620" alt="Image" src="https://github.com/user-attachments/assets/60c9c7b4-b083-47ea-ba4a-dd358c5a0b5a" />
    
3.  **Validação dos Agentes:**
    Confirme na seção **Nodes** do Jenkins que os agentes **`agent1`** e **`agent2`** estão conectados e prontos para receber o *label* **`cpp-agent`**.
    <img width="2557" height="513" alt="Image" src="https://github.com/user-attachments/assets/73cbbc6b-9b8b-437e-905c-54accafbbcee" />

### 5.4. Limpeza (Cleanup)

Para derrubar e remover os contêineres e a rede após a avaliação:

```bash
cd jenkins-setup
docker compose down -v
```

## 6. Desafios, Reflexões e Próximos Passos

Esta seção detalha os desafios operacionais encontrados durante a implementação da arquitetura e traça o **roteiro de evolução** da solução para um ambiente de Continuous Delivery (CD).

### 6.1. Desafios Operacionais e Soluções Imediatas

Durante a validação da pipeline, alguns pontos operacionais foram identificados e endereçados:

| Desafio | Solução na Implementação | Próximo Passo para Produção |
| :--- | :--- | :--- |
| **Instabilidade Crítica do Ambiente de Teste** | A decisão foi migrar a infraestrutura do Jenkins para um ambiente de execução local (Localhost/Docker), usando a máquina do desenvolvedor como host. | O ambiente remoto fornecido estava caindo e apresentando latência excessiva, o que impedia a iteração e o debug eficiente. A migração foi feita para desbloquear o desenvolvimento e garantir que a solução de CI/CD (o foco do teste) pudesse ser entregue e validada de forma estável. |
| **Cache no Build do Docker** | Foi necessário utilizar a flag `--build --force-recreate` durante os testes para garantir que as alterações no JCasC fossem refletidas. | **Otimização do Dockerfile:** A ordem das instruções do `Dockerfile` deve ser otimizada para que o `COPY` dos arquivos de configuração seja uma das últimas camadas, maximizando o uso do cache nas camadas base. |
| **Status de Commit do GitHub** | O log de execução reportou `status: "401" (Requires authentication)`. | **Resolver Autenticação:** É mandatório criar uma **Credencial (PAT - Personal Access Token)** no GitHub e configurá-la no Jenkins. Esta credencial será usada para notificar o status do build (sucesso/falha) diretamente no Commit ou Pull Request. |

#### Detalhamento dos desafios

**6.1.1 Desafio:** Por conta da instabilidade crítica do ambiente de teste (servidor remoto caindo e latência excessiva), foi decidido migrar a infraestrutura do Jenkins para um ambiente de execução local. Essa decisão de gerenciamento de risco transformou o problema de infraestrutura em uma validação do princípio de portabilidade do projeto. O foco permaneceu na entrega da solução de CI/CD.

**6.1.2 Desafio:** Durante a execução, foi identificado que a pipeline não conseguia atualizar o status do commit no GitHub (erro **401 - Requires Authentication**), e também apresentou lentidões relacionadas ao rate limit da API do GitHub, conforme o log:
<img width="2538" height="1313" alt="Image" src="https://github.com/user-attachments/assets/535cb027-8e08-48a9-9ace-d5fcef2c768c" />
<img src="https://github.com/user-attachments/assets/77694e55-ade4-4303-b8f4-3b6d6f159962" alt="Log de erro 401" width="800"/>

**6.1.2 Solução:** Para resolver esses problemas, foi criada e configurada a credencial **GitHub Personal Access Token (PAT)** no Jenkins. A credencial precisa ser configurada no Job Multibranch, conforme capturado abaixo:

<img src="https://github.com/user-attachments/assets/11b2cf4d-61fe-41f4-8809-1dc61cbdcde9" alt="Configuração da credencial PAT no Jenkins" width="800"/>

Essa etapa garante que o Job tenha permissão para interagir com a API do GitHub, fornecendo *feedback* visual (checks verde/vermelho) no código-fonte

### 6.2. Roteiro de Evolução da Arquitetura (Continuous Delivery)

Para evoluir a solução de CI (Integração Contínua) para um modelo completo de **CI/CD (Continuous Delivery)**, as seguintes implementações seriam priorizadas:

#### 1. Containerização Final e Gerenciamento de Artefatos

A etapa de `Archive Artifacts` será substituída por:
* **Docker Build:** Construção de uma imagem final (utilizando o binário C++).
* **Docker Push:** Envio da imagem para um **Container Registry** (ex: Docker Hub ou Artifactory).
    * **Valor Agregado:** A imagem será **taggeada** com o `git commit hash` ou o `BUILD_NUMBER`, garantindo a **rastreabilidade** da versão do código até o contêiner final.

#### 2. Implementação de CD e Deploy Automatizado

* **Stage de Deploy:** Adicionar uma *Stage* de **`Deploy to Staging`** após o sucesso do *push* da imagem.
* **Tecnologia:** Utilizar uma ferramenta de orquestração moderna como **Kubernetes (via `kubectl` ou Helm Charts)** para gerenciar o ambiente de *staging* e produção, demonstrando familiaridade com a infraestrutura de *Cloud Native*.
