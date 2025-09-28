# Teste Técnico Philips

- [Introdução](#-introdução)


## Introdução

O objetivo desta solução é estabelecer um Processo de CI/CD resiliente e eficiente para a aplicação C++, cumprindo todas as diretrizes da Prova Técnica. A arquitetura foi desenvolvida utilizando Jenkins Master/Agent em contêineres Docker, com Agents de execução dedicados para o ambiente C++17, garantindo isolamento e reprodutibilidade do build. A Pipeline Declarativa implementada oferece rastreabilidade completa e permite builds otimizados através de parâmetros de execução condicional. O foco está em qualidade imediata, onde a falha em qualquer etapa (checa de código ou teste) interrompe o fluxo, assegurando que apenas artefatos validados sejam entregues.

## Stack

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

## 4. Guia de Configuração e Execução

Este guia fornece as instruções necessárias para inicializar a infraestrutura de CI/CD (Jenkins + Agents) e executar a pipeline, validando todos os requisitos da prova.

### 4.1. Pré-requisitos

Para rodar a solução de forma local e reprodutível, os seguintes softwares são obrigatórios:

* **Git:** Para clonar o repositório.
* **Docker:** Para rodar os contêineres.
* **Docker Compose:** Para gerenciar e orquestrar os serviços definidos em `jenkins-setup/docker-compose.yml`.

### 4.2. Inicialização da Infraestrutura de CI (IaC)

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

### 4.3. Execução e Validação da Pipeline

A pipeline está pronta para ser executada no ambiente `cpp-agent` assim que o Jenkins for inicializado.

1.  **Crie o Job Pipeline:**
    * No Jenkins UI, crie um novo item do tipo **Pipeline**.
    * No campo **Definição**, selecione **"Pipeline script from SCM"**.
    * Configure o SCM para **Git** e aponte para o **URL do seu repositório**.
    * Defina o Script Path como `Jenkinsfile`.

2.  **Execute o Build (Testando os Parâmetros):**
    Use a opção **"Build With Parameters"** para validar as funcionalidades condicionais:
    * **Teste de Qualidade Total:** Deixe ambos os parâmetros (`SKIP_CHECKS_AND_TESTS` e `GENERATE_ARTIFACT`) como `false`. A pipeline deve executar **Code Check, Build e Test**.
    * **Teste de Geração de Artefato Sob Demanda:** Marque apenas `GENERATE_ARTIFACT`. A pipeline deve executar Build e **Archive Artifacts**, simulando o gatilho manual.

3.  **Validação dos Agentes:**
    Confirme na seção **Nodes** do Jenkins que os agentes **`agent1`** e **`agent2`** estão conectados e prontos para receber o *label* **`cpp-agent`**.

### 5.4. Limpeza (Cleanup)

Para derrubar e remover os contêineres e a rede após a avaliação:

```bash
cd jenkins-setup
docker-compose down -v
