Okay, let's move to **Section 4: DOCKER-COMPOSE & LIMITES**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 4. DOCKER-COMPOSE & LIMITES

*   Serviços com **mem\_limit** e **cpus** definidos.
*   **Healthchecks** para `scene_scanner`, `playlist_manager` e `tv_api`.
*   `restart: on-failure` e `restart: always` onde aplicável.
*   `watchtower` para atualizações automáticas de imagens internas (Ver Seção 12.5 para gerenciamento avançado).
```

This section outlines key aspects of the `docker-compose.yml` file. We can enrich this by:

1.  **Providing a more structured overview of the `docker-compose.yml` file's purpose and general structure.**
2.  **Elaborating on `mem_limit` and `cpus`:**
    *   Why they are important, especially for edge devices.
    *   General guidance or examples of how these limits might be set for different types of services (e.g., CPU-intensive vs. I/O-intensive).
3.  **Detailing `healthchecks`:**
    *   The purpose of healthchecks.
    *   Examples of healthcheck commands for different services (e.g., a simple `curl` for a web service, a script execution for a worker).
    *   How healthchecks interact with `restart` policies and `self_monitor.py`.
4.  **Clarifying `restart` policies:**
    *   When to use `on-failure` (e.g., for task-based containers run by the orchestrator if they weren't `--rm`, or for services that should recover from crashes).
    *   When to use `always` or `unless-stopped` (e.g., for long-running services like UIs, `watchtower`, or a persistent `orchestrator` if it were a service).
5.  **Expanding on `watchtower`:**
    *   Its role and basic configuration within `docker-compose.yml`.
    *   Explicitly mention its interaction with the CI/CD pipeline and private registry as discussed in `12.5`.
    *   How to configure `watchtower` to only monitor specific containers or to use credentials for a private registry.
6.  **Mentioning other important `docker-compose.yml` features used:**
    *   **Volumes:** How data persistence is managed for services (e.g., mapping `${VIDEO_STORAGE_ROOT}` subdirectories). Named volumes vs. host mounts.
    *   **Networks:** Use of default bridge network vs. custom networks for better isolation if needed.
    *   **Environment Variables:** How services are configured using `.env` file and environment sections in `docker-compose.yml`.
    *   **`depends_on`:** If used, explain how it controls startup order (though it doesn't guarantee service readiness).
    *   **Logging drivers:** Configuration for Docker logging (e.g., json-file with size/rotation, or sending to a centralized logging system).

Let's create a more detailed version of Section 4.

---

## 4. DOCKER-COMPOSE & GERENCIAMENTO DE RECURSOS

O arquivo `docker-compose.yml` é o pilar da arquitetura de serviços do `VV-Video-AI-System`. Ele define como os diversos containers (serviços) são construídos, configurados, interconectados e gerenciados. Esta seção detalha as principais configurações e estratégias empregadas no `docker-compose.yml` para garantir a robustez, eficiência e controlabilidade do sistema, especialmente em ambientes de borda com recursos limitados.

---

### 4.1. Estrutura Geral e Propósito

*   **Definição de Serviços:** Cada componente principal do sistema (e.g., `agent_dvr`, `scene_scanner`, `scene_summarizer`, `logline_writer`, `playlist_manager`, `tv_api`, `watchtower`) é definido como um serviço no `docker-compose.yml`.
*   **Construção de Imagens:** Para serviços com código customizado, o `docker-compose.yml` especifica o contexto de build e o `Dockerfile` a ser usado (e.g., `build: ./agent_dvr/`). Para imagens públicas, ele especifica o nome da imagem (e.g., `image: containrrr/watchtower:latest`).
*   **Configuração Centralizada:** Permite a definição de volumes, redes, portas, variáveis de ambiente, limites de recursos e políticas de reinício de forma declarativa.

### 4.2. Limites de Recursos (`mem_limit` e `cpus`)

A imposição de limites de recursos é crucial para a estabilidade em dispositivos de borda, prevenindo que um serviço consuma todos os recursos disponíveis e afete outros serviços ou o sistema operacional.

*   **`mem_limit`:** Define a quantidade máxima de memória RAM que um container pode usar (e.g., `mem_limit: 512m`, `mem_limit: 2g`).
    *   **Importância:** Previne erros de "Out of Memory" (OOM) no nível do host e permite um compartilhamento mais justo da memória.
    *   **Definição:** Os limites são ajustados com base no consumo esperado de cada serviço. Serviços que manipulam dados grandes ou modelos de IA (como `scene_summarizer`) podem requerer limites mais altos. Testes de carga e monitoramento (Seção `12.10`) ajudam a refinar esses valores.
*   **`cpus`:** Define a fração de CPUs que um container pode usar (e.g., `cpus: '0.5'` para meio core, `cpus: '2.0'` para dois cores).
    *   **Importância:** Controla o impacto de serviços CPU-intensivos no desempenho geral do sistema.
    *   **Definição:** Alocado com base na carga de trabalho. `scan_video` e `scene_summarizer` podem ser mais CPU-intensivos durante o processamento.

*   **Exemplo no `docker-compose.yml`:**
    ```yaml
    services:
      scene_summarizer:
        # ...
        mem_limit: 2g
        cpus: '1.5'
        # Para GPUs, configurações adicionais em 'deploy: resources: reservations: devices' podem ser necessárias.
    ```

### 4.3. Verificações de Saúde (`healthcheck`)

Healthchecks permitem ao Docker monitorar o estado interno de um container, indo além de apenas verificar se o processo principal está em execução.

*   **Propósito:** Detectar containers que estão rodando mas não funcionam corretamente (e.g., um servidor web que não responde, um worker travado).
*   **Configuração:** Definido por serviço, especifica um comando (`test`), intervalo (`interval`), tempo limite (`timeout`), número de tentativas (`retries`) e tempo de início (`start_period`).
*   **Exemplos:**
    *   **`tv_api` (Serviço Web FastAPI):**
        ```yaml
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8000/status"] # Assumindo que tv_api roda na porta 8000 internamente
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s # Dar tempo para a aplicação iniciar
        ```
    *   **`playlist_manager` (Worker Script):**
        ```yaml
        healthcheck:
          # Verifica se um arquivo de 'heartbeat' foi atualizado recentemente pelo script
          test: ["CMD", "find", "/app/heartbeat", "-mmin", "-5"] # Exemplo: modificado nos últimos 5 minutos
          interval: 2m
          timeout: 10s
          retries: 3
        ```
    *   **`scene_scanner` (Pode ser similar ao `playlist_manager` ou verificar um status interno se for um serviço de longa duração).**
*   **Interação:** Um status `unhealthy` pode ser usado pelo `self_monitor.py` (Seção `11` e `12.2`) como um gatilho para reinício ou alerta. Ele também influencia como o Docker lida com reinícios e atualizações.

### 4.4. Políticas de Reinício (`restart`)

Definem o comportamento do Docker quando um container para.

*   **`no` (Padrão):** Não reinicia o container.
*   **`on-failure[:max_retries]`:** Reinicia o container apenas se ele sair com um código de erro diferente de zero. Pode-se especificar um número máximo de tentativas (e.g., `on-failure:3`).
    *   **Uso:** Adequado para serviços que devem se recuperar de falhas transitórias ou erros internos. `scene_scanner`, `scene_summarizer`, `logline_writer`, `playlist_manager` (se forem serviços de longa duração e não apenas tasks do `orchestrator.py`).
*   **`always`:** Reinicia o container sempre que ele parar, a menos que seja explicitamente parado pelo Docker (e.g., `docker stop`).
    *   **Uso:** Para serviços críticos que devem estar sempre em execução (e.g., `agent_dvr`, `tv_api`, `watchtower`).
*   **`unless-stopped`:** Similar a `always`, mas não reinicia o container se ele foi explicitamente parado (e.g., via `docker stop`) antes do daemon Docker ser reiniciado. É geralmente a política mais recomendada para serviços de longa duração.

### 4.5. `watchtower` e Atualizações Automáticas

`Watchtower` monitora imagens Docker em execução e as atualiza automaticamente para a versão mais recente encontrada no registro.

*   **Configuração Básica:**
    ```yaml
    services:
      watchtower:
        image: containrrr/watchtower:latest
        container_name: watchtower
        restart: unless-stopped
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock # Necessário para interagir com o Docker daemon
        environment:
          # WATCHTOWER_POLL_INTERVAL: 300 # Default é 5 minutos
          WATCHTOWER_CLEANUP: "true"      # Remove imagens antigas após a atualização
          WATCHTOWER_INCLUDE_STOPPED: "true" # Também atualiza containers parados
          # WATCHTOWER_DEBUG: "true"
        # Para monitorar apenas containers específicos (recomendado):
        command: --interval 300 agent_dvr scene_scanner scene_summarizer # Nomes dos containers a monitorar
    ```
*   **Gerenciamento Avançado (Conforme Seção `12.5`):**
    *   Quando imagens customizadas são publicadas em um **registro privado** (e.g., GHCR), `watchtower` precisa ser configurado com credenciais para acessar este registro.
    *   Isso pode ser feito montando o arquivo `~/.docker/config.json` do host (que contém as credenciais após um `docker login`) ou usando variáveis de ambiente como `REPO_USER` e `REPO_PASS`.
        ```yaml
        # Exemplo com variáveis de ambiente para registro privado
        # volumes:
        #   - /var/run/docker.sock:/var/run/docker.sock
        # environment:
        #   REPO_USER: "${GHCR_USER}" # Usuário/token para o registro
        #   REPO_PASS: "${GHCR_TOKEN_FOR_WATCHTOWER}" # Token de acesso
        ```
    *   `watchtower` então baixará as novas versões das imagens customizadas (`vv-agent_dvr`, `vv-scene_scanner`, etc.) e recriará os containers correspondentes.

### 4.6. Outras Configurações Importantes no `docker-compose.yml`

*   **Volumes:**
    *   **Host Mounts:** Usados extensivamente para mapear subdiretórios de `${VIDEO_STORAGE_ROOT}` para dentro dos containers (e.g., `- ./storage/input:/app/input`). Isso permite que os scripts Python acessem e gravem dados persistentes.
    *   **Named Volumes:** Podem ser usados para dados que não precisam ser diretamente acessíveis do host, mas precisam persistir entre recriações de containers (e.g., `prometheus_data`, `grafana_data`).
*   **Redes:**
    *   Por padrão, o Docker Compose cria uma rede bridge para todos os serviços no arquivo, permitindo que eles se comuniquem usando os nomes dos serviços como hostnames.
    *   Redes customizadas podem ser definidas para maior isolamento ou configurações de rede específicas, se necessário.
*   **Variáveis de Ambiente (`environment` e `.env` file):**
    *   O `docker-compose.yml` carrega automaticamente variáveis de um arquivo `.env` no mesmo diretório.
    *   Variáveis de ambiente específicas podem ser passadas para cada serviço usando a seção `environment:`.
        ```yaml
        services:
          orchestrator_service: # Se o orquestrador fosse um serviço de longa duração
            image: my_orchestrator_image
            environment:
              - ORCHESTRATOR_INTERVAL=${ORCHESTRATOR_INTERVAL_FROM_ENV}
              - LOG_LEVEL=INFO
        ```
*   **`depends_on`:**
    *   Usado para controlar a ordem de inicialização dos serviços. Por exemplo, um serviço que depende de um banco de dados pode ter `depends_on: [db_service]`.
    *   **Importante:** `depends_on` apenas garante que o container da dependência foi iniciado, não que o serviço dentro dele está pronto para aceitar conexões. Healthchecks ou scripts de entrypoint com lógica de espera são necessários para isso.
*   **Drivers de Logging:**
    *   Pode-se configurar o driver de logging para os containers. O padrão é `json-file`.
        ```yaml
        services:
          scene_scanner:
            # ...
            logging:
              driver: "json-file"
              options:
                max-size: "10m" # Rotaciona logs após 10MB
                max-file: "3"   # Mantém 3 arquivos de log rotacionados
        ```
    *   Isso ajuda a gerenciar o espaço em disco consumido pelos logs de container.

---

Esta versão expandida da Seção 4 fornece uma visão mais completa de como o `docker-compose.yml` é utilizado para orquestrar os serviços, gerenciar recursos, garantir a saúde dos containers e facilitar atualizações, integrando-se com as melhorias propostas nas seções `12.x`.
