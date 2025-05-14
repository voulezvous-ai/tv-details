Okay, let's move to **Section 8: TESTES & CI**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 8. TESTES & CI

*   **pytest** cobrindo scripts e orquestração.
*   **Cobertura ≥ 90%**.
*   **Testes E2E** via Selenium headless (opcional).
```

This section outlines the testing strategy and Continuous Integration (CI) approach. We can enrich this by:

1.  **Expanding on the types of tests:**
    *   **Unit Tests:** More detail on what they cover (e.g., individual functions in `vv-core`, logic within each script).
    *   **Integration Tests:** How different components are tested together (e.g., `orchestrator.py`'s interaction with mock versions of `scan_video.py`, `summarize_scene.py`). Testing interactions between actual containers.
    *   **End-to-End (E2E) Tests:** Clarify the scope beyond just Selenium (e.g., simulating a video drop into `input/` and verifying outputs in `loglines/` and `playlists/`).
    *   **Contract Tests:** For schemas (`analytic.schema.json`, etc.) to ensure data producers and consumers agree.
2.  **Detailing the `pytest` usage:**
    *   Directory structure for tests (`tests/`).
    *   Use of fixtures, mocks (e.g., `pytest-mock`), and plugins.
3.  **Discussing "Cobertura ≥ 90%":**
    *   Tools used to measure coverage (e.g., `pytest-cov`).
    *   How this target is enforced/monitored.
4.  **Elaborating on "Testes E2E via Selenium headless (opcional)":**
    *   What specific UI functionalities would be tested?
    *   Alternative E2E testing tools if Selenium is too heavy for this project (e.g., Playwright, or script-based API/file verification).
5.  **Expanding on the CI (Continuous Integration) aspect:**
    *   Which CI server/service is used (e.g., GitHub Actions, GitLab CI, Jenkins)? This links to `12.5`.
    *   What triggers the CI pipeline (e.g., push to any branch, push to `main`, PR creation)?
    *   Key stages in the CI pipeline:
        *   Linting and static analysis.
        *   Running unit tests.
        *   Running integration tests.
        *   Building Docker images (as per `12.5`).
        *   (Optional) Pushing images to a staging registry.
        *   (Optional) Deploying to a staging environment.
        *   (Optional) Running E2E tests against a staging environment.
    *   Reporting of test results and build status.
6.  **Considering a "test data" strategy:** How are test videos and expected outputs managed?

Let's create a more detailed version of Section 8.

---

## 8. ESTRATÉGIA DE TESTES & INTEGRAÇÃO CONTÍNUA (CI)

Uma estratégia de testes robusta e um pipeline de Integração Contínua (CI) são fundamentais para garantir a qualidade, estabilidade e manutenibilidade do `VV-Video-AI-System`. O objetivo é detectar problemas o mais cedo possível no ciclo de desenvolvimento e automatizar o processo de verificação.

---

### 8.1. Tipos de Testes

O sistema emprega uma combinação de diferentes tipos de testes para cobrir vários aspectos:

1.  **Testes Unitários:**
    *   **Foco:** Verificar a menor unidade de código (funções, métodos, classes) de forma isolada.
    *   **Escopo:**
        *   Funções em `vv-core/utils.py` (hashing, validação de schema, assinatura HMAC, etc.).
        *   Lógica de negócio principal dentro de cada script Python (e.g., `scan_video.py`, `summarize_scene.py`, `tv_scheduler.py`), com dependências externas (como chamadas a modelos de IA ou I/O de arquivos) mockadas.
        *   Manipulação de dados, transformações e lógica condicional.
    *   **Ferramentas:** `pytest` framework, `pytest-mock` para mocking.
    *   **Localização:** `tests/unit/`.

2.  **Testes de Integração:**
    *   **Foco:** Verificar a interação entre diferentes componentes ou módulos do sistema.
    *   **Escopo:**
        *   Interação do `orchestrator.py` com os scripts do pipeline (`scan_video.py`, `summarize_scene.py`, `logline_generator.py`). Estes podem ser testados mockando a execução `docker-compose run` ou testando a lógica do orquestrador com "stubs" dos scripts que produzem saídas esperadas/errôneas.
        *   Interação entre serviços que se comunicam diretamente (e.g., se `tv_api` consumisse uma API de outro serviço).
        *   Teste da pipeline de processamento de um vídeo de teste através de múltiplos estágios, verificando a criação correta dos arquivos intermediários (`.analytic.json`, `.summary.json`, `.logline.json`). Pode envolver a execução real de `docker-compose run` para os containers de processamento com volumes de teste.
    *   **Ferramentas:** `pytest`, `docker-compose` para orquestrar containers de teste.
    *   **Localização:** `tests/integration/`.

3.  **Testes de Contrato (Schema Validation):**
    *   **Foco:** Garantir que os dados trocados entre os serviços (principalmente via arquivos JSON) aderem aos schemas definidos (`analytic.schema.json`, `summary.schema.json`, `logline.schema.json`, `playlist.json`).
    *   **Escopo:** Incluído como parte dos testes unitários dos módulos que produzem esses JSONs e como parte dos testes de integração onde esses JSONs são consumidos.
    *   **Ferramentas:** `jsonschema` biblioteca, integrada com `pytest`.

4.  **Testes End-to-End (E2E):**
    *   **Foco:** Validar o fluxo completo do sistema do ponto de vista do usuário ou de uma entrada de dados completa.
    *   **Escopo:**
        *   **Pipeline de Dados:** Simular a adição de um novo arquivo de vídeo ao diretório `input/`. Verificar se o `orchestrator.py` o processa corretamente, se todos os artefatos (`.analytic.json`, `.summary.json`, `.logline.json`) são gerados, se a LogLine é válida, e se o vídeo aparece na `playlist.json` apropriada.
        *   **(Opcional) Testes de UI:** Se a complexidade da UI justificar, usar ferramentas como Selenium, Playwright ou Cypress para testar funcionalidades chave das UIs (`tv.html`, `tv_mirror.html`). Isso pode incluir:
            *   Verificar se a página carrega.
            *   Verificar se a playlist é carregada e os vídeos são reproduzidos (pode ser desafiador com players de vídeo).
            *   Verificar o funcionamento do fallback.
            *   Para `tv_mirror.html`, verificar se a imagem placeholder aparece quando o stream MJPEG está indisponível.
    *   **Ferramentas:** Scripts `pytest` que orquestram as ações (colocar arquivo, verificar saídas), `docker-compose` para rodar o sistema completo em ambiente de teste. Para UI: Selenium, Playwright, Cypress.
    *   **Localização:** `tests/e2e/`.

### 8.2. Ferramentas e Práticas de Teste

*   **Framework de Teste:** `pytest` é o framework principal devido à sua simplicidade, flexibilidade e rico ecossistema de plugins.
*   **Estrutura de Testes:** Os testes são organizados no diretório `tests/` com subdiretórios para `unit/`, `integration/`, e `e2e/`.
*   **Fixtures (`pytest`):** Usadas extensivamente para configurar o estado de teste (e.g., criar arquivos temporários, mockar objetos, iniciar containers).
*   **Mocking:** `pytest-mock` (wrapper para `unittest.mock`) é usado para isolar unidades de código e simular dependências.
*   **Dados de Teste:**
    *   Um conjunto pequeno e gerenciável de arquivos de vídeo de teste.
    *   Arquivos JSON de exemplo (e.g., `analytic.json`, `summary.json`) para testar módulos consumidores.
    *   Os dados de teste podem ser armazenados no repositório (se pequenos) ou baixados durante a configuração do teste.
*   **Cobertura de Testes:**
    *   **Meta:** Cobertura de código de, no mínimo, **90%** para testes unitários e de integração.
    *   **Ferramenta:** `pytest-cov` para gerar relatórios de cobertura.
    *   **Monitoramento:** A cobertura é verificada no pipeline de CI, e PRs que diminuem a cobertura podem ser bloqueados ou sinalizados.

### 8.3. Integração Contínua (CI)

O pipeline de CI automatiza a construção, teste e verificação do código a cada alteração.

*   **Serviço de CI:** GitHub Actions é a escolha preferencial, dada a sua integração com o GitHub (onde o código é presumivelmente hospedado). Alternativas incluem GitLab CI, Jenkins, etc. (Conforme Seção `12.5` para o CI/CD de imagens Docker).
*   **Gatilhos do Pipeline de CI:**
    *   Push para qualquer branch.
    *   Criação/atualização de Pull Requests (PRs) para a branch `main` (ou `develop`).
*   **Principais Etapas do Pipeline de CI (Exemplo com GitHub Actions):**
    1.  **Checkout do Código:** Obtém a última versão do código da branch/PR.
    2.  **Configuração do Ambiente:**
        *   Configura a versão correta do Python.
        *   Instala dependências da aplicação (e.g., via `pip install -r requirements.txt`).
        *   Instala dependências de teste (e.g., `pip install -r requirements-dev.txt`).
        *   Configura o Docker e Docker Compose.
    3.  **Análise Estática e Linting:**
        *   Executa linters como `flake8` ou `pylint`.
        *   Executa formatadores como `black --check` e `isort --check`.
        *   Falha o build se houver erros de linting/formatação.
    4.  **Execução de Testes Unitários e de Integração:**
        *   Executa `pytest tests/unit/` e `pytest tests/integration/`.
        *   Gera relatório de cobertura com `pytest-cov`.
        *   (Opcional) Faz upload do relatório de cobertura para um serviço como Codecov ou Coveralls.
        *   Falha o build se algum teste falhar ou se a cobertura cair abaixo do limiar.
    5.  **Construção de Imagens Docker (Conforme Seção `12.5`):**
        *   `docker-compose build` para todas as imagens customizadas.
        *   Esta etapa verifica se os Dockerfiles estão corretos e se as imagens podem ser construídas.
    6.  **(Opcional, para PRs ou merges na `main`) Push de Imagens para Registro:**
        *   Se todos os passos anteriores forem bem-sucedidos, as imagens Docker construídas são tagueadas e enviadas para um registro privado (e.g., GHCR), conforme detalhado em `12.5`.
    7.  **(Opcional) Execução de Testes E2E:**
        *   Pode requerer a inicialização do sistema completo (ou um subconjunto) usando `docker-compose up -d` em um ambiente de CI (ou um ambiente de staging dedicado).
        *   Executa os scripts de teste E2E.
        *   Este passo pode ser mais demorado e pode ser executado condicionalmente (e.g., apenas em PRs para `main` ou em um cronograma noturno).
    8.  **Notificação de Status:** O resultado do pipeline de CI (sucesso/falha) é reportado no PR, no commit, e/ou via notificações (Slack, email).

### 8.4. Execução Local de Testes

*   Os desenvolvedores devem ser capazes de executar todos os testes (ou subconjuntos) localmente antes de fazer push do código.
*   O `Makefile` (Seção de Estrutura do Repositório) deve incluir comandos convenientes para isso:
    *   `make test`: Roda todos os testes.
    *   `make test-unit`: Roda apenas testes unitários.
    *   `make test-integration`: Roda apenas testes de integração.
    *   `make lint`: Roda linters.
    *   `make coverage`: Roda testes e gera relatório de cobertura.

---

Esta estratégia de testes e CI visa criar um ciclo de desenvolvimento robusto, onde a qualidade é verificada continuamente, facilitando a entrega de um `VV-Video-AI-System` confiável e estável.
