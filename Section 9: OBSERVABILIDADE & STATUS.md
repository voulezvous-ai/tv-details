Okay, let's move to **Section 9: OBSERVABILIDADE & STATUS**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 9. OBSERVABILIDADE & STATUS

*   Logs estruturados em `logs/app.log`.
*   Endpoint `/status` em FastAPI.
*   Métricas compatíveis com Prometheus/Grafana (Ver Seção 12.10 para métricas detalhadas).
```

This section covers how the system's health and behavior can be observed. We've already detailed metrics extensively in `12.10` and alerts in `12.9`. We can enrich this section by:

1.  **Elaborating on "Logs estruturados":**
    *   What format is used (e.g., JSON, key-value pairs)?
    *   Key fields included in each log entry (timestamp, level, service_name, message, correlation_id if applicable).
    *   How `logs/app.log` is managed (rotation via `logrotate` as per `12.3`).
    *   Centralized logging considerations (e.g., ELK stack, Grafana Loki) for larger deployments, even if not implemented initially.
2.  **Detailing the "Endpoint `/status` em FastAPI":**
    *   Which service hosts this endpoint (e.g., a dedicated API service, or a main component like `tv_api` or `orchestrator` if it were a long-running service)?
    *   What information does it return? (e.g., overall system status, status of individual components/containers, version, uptime, basic stats).
    *   How is this endpoint secured (ref `12.1`)?
    *   Example JSON response.
3.  **Reinforcing the link to Prometheus/Grafana (`12.10`) and Alerting (`12.9`):** Briefly summarize how these fit into the overall observability strategy.
4.  **Adding a subsection on "Health Checks"** as an observability tool, summarizing their role as defined in Section 4.
5.  **Considering "Distributed Tracing" (Optional/Advanced):** For very complex interactions, tools like Jaeger or Zipkin could be mentioned as future possibilities, though likely overkill for the current scope.

Let's create a more detailed version of Section 9.

---

## 9. OBSERVABILIDADE, STATUS & SAÚDE DO SISTEMA

A observabilidade é crucial para entender o comportamento do `VV-Video-AI-System` em tempo real, diagnosticar problemas, monitorar a performance e garantir a sua saúde operacional. Esta seção descreve as ferramentas e práticas empregadas para alcançar uma boa observabilidade.

---

### 9.1. Logging Estruturado

Logs são a base da observabilidade, fornecendo um registro cronológico de eventos, erros e informações de depuração.

*   **Formato dos Logs:**
    *   Os logs são predominantemente **estruturados**, preferencialmente em formato **JSON** ou um formato key-value de fácil parseamento.
    *   **Campos Padrão por Entrada de Log:**
        *   `timestamp`: Data e hora do evento em formato ISO 8601 UTC (e.g., `2025-05-15T14:30:00.123Z`).
        *   `level`: Nível de severidade do log (e.g., `INFO`, `WARNING`, `ERROR`, `CRITICAL`, `DEBUG`).
        *   `service_name`: Nome do container ou script que gerou o log (e.g., `scene_scanner`, `orchestrator.py`, `tv_api`).
        *   `hostname`: Identificador do host/edge device.
        *   `message`: A mensagem de log principal.
        *   **(Opcional) `correlation_id`:** Um ID para rastrear uma requisição ou fluxo de processamento de vídeo através de múltiplos serviços/logs.
        *   **(Opcional) `video_file_context`:** Nome ou hash do vídeo sendo processado, se aplicável.
        *   **(Opcional) `function_name`, `line_number`:** Para depuração.
        *   Quaisquer campos de contexto adicionais relevantes para o evento.
*   **Arquivo Principal de Log (`logs/app.log`):**
    *   A maioria dos scripts Python e serviços customizados direciona seus logs para um arquivo centralizado `/opt/vv-video-ai-system/logs/app.log`.
    *   A biblioteca `logging` do Python é configurada (potencialmente via `vv-core/utils.py setup_logging()`) para usar o formato estruturado.
*   **Gerenciamento de `logs/app.log`:**
    *   A rotação, compressão e limpeza de `logs/app.log` (e outros logs do sistema como `cleanup_*.log`, `cron_*.log`) são gerenciadas por `logrotate`, conforme detalhado na Seção `12.3`.
*   **Logs de Containers Docker:**
    *   Os containers Docker também geram logs para `stdout`/`stderr`, que são capturados pelo Docker daemon.
    *   Estes podem ser acessados via `docker logs <container_name>`.
    *   O driver de logging do Docker (`json-file` por padrão) é configurado com rotação (max-size, max-file) no `docker-compose.yml` (Seção 4) para evitar o consumo excessivo de disco.
*   **Considerações Futuras (Centralized Logging):**
    *   Para implantações em maior escala ou com múltiplos edge devices, a integração com um sistema de logging centralizado (e.g., ELK Stack - Elasticsearch, Logstash, Kibana; ou Grafana Loki com Promtail) seria benéfica. Os logs seriam enviados dos dispositivos para um servidor central para agregação, busca e análise.

---

### 9.2. Endpoint de Status (`/status` API)

Um endpoint HTTP dedicado fornece uma visão geral rápida do estado de saúde do sistema.

*   **Host do Endpoint:**
    *   Este endpoint é exposto por um dos serviços web, idealmente um serviço leve dedicado para API de status/gerenciamento, ou integrado a um serviço existente como o `tv_api` (se este for um container FastAPI/Python).
    *   Se o sistema não tiver um serviço Python de longa duração adequado para hospedar esta API, uma alternativa simples poderia ser um script que gera um arquivo JSON de status periodicamente, e o NGINX serve este arquivo estático. No entanto, uma API dinâmica é preferível.
*   **Informações Retornadas:**
    *   **`overall_status`**: (e.g., `HEALTHY`, `DEGRADED`, `UNHEALTHY`).
    *   **`system_version`**: Versão do `VV-Video-AI-System` (e.g., do `CHANGELOG.md` ou uma tag Git).
    *   **`uptime_seconds`**: Tempo de atividade do serviço que expõe o endpoint.
    *   **`timestamp_utc`**: Horário atual do servidor.
    *   **`components`**: Uma lista de componentes chave e seus status:
        *   Nome do componente/serviço Docker (e.g., `scene_scanner`, `orchestrator_status`).
        *   Status (e.g., `RUNNING`, `HEALTHY`, `UNHEALTHY`, `STOPPED`, `PROCESSING_OK`, `QUEUE_LENGTH_NORMAL`).
        *   Última atividade/heartbeat (timestamp).
        *   Métricas básicas (e.g., `videos_in_queue` para o orquestrador, `active_connections` para `tv_api`).
    *   **`checks`**: (Opcional) Resultados de verificações de dependências:
        *   `dvr_connection`: (e.g., `CONNECTED`, `DISCONNECTED`).
        *   `video_storage_writable`: (e.g., `OK`, `READ_ONLY`, `NOT_ACCESSIBLE`).
        *   `bitnet_model_loaded`: (e.g., `OK`, `NOT_FOUND`).
*   **Exemplo de Resposta JSON:**
    ```json
    {
      "overall_status": "HEALTHY",
      "system_version": "v2.2.0",
      "uptime_seconds": 36000,
      "timestamp_utc": "2025-05-15T15:00:00Z",
      "components": [
        {"name": "agent_dvr", "status": "RUNNING", "health": "HEALTHY", "last_seen_active": "2025-05-15T14:59:55Z"},
        {"name": "scene_scanner_service", "status": "IDLE", "health": "HEALTHY", "pending_tasks": 0},
        {"name": "scene_summarizer_service", "status": "IDLE", "health": "HEALTHY", "model_version": "1.0.0"},
        {"name": "orchestrator_process", "status": "LAST_RUN_OK", "last_run_utc": "2025-05-15T14:55:00Z", "videos_in_queue": 2},
        {"name": "tv_api", "status": "RUNNING", "health": "HEALTHY", "active_playlist": "tv1_curated"}
      ],
      "checks": {
        "dvr_connectivity": "CONNECTED",
        "video_storage_status": "OK_WRITABLE",
        "bitnet_model_status": "LOADED_VERSION_1.0.0"
      }
    }
    ```
*   **Segurança:** O acesso ao endpoint `/status` deve ser protegido, especialmente se expor informações detalhadas. A Seção `12.1` (NGINX+JWT) cobre essa proteção. Pode-se também restringir o acesso por IP.

---

### 9.3. Métricas e Dashboards (Prometheus/Grafana)

Conforme detalhado extensivamente na Seção `12.10`, o monitoramento baseado em métricas com Prometheus e Grafana é um pilar da estratégia de observabilidade.

*   **Prometheus:** Coleta métricas de performance, utilização de recursos e saúde de todos os componentes do sistema (scripts Python, containers, host).
*   **Grafana:** Fornece dashboards customizáveis para visualizar essas métricas, permitindo análises de tendências, identificação de gargalos e monitoramento em tempo real.
*   **Integração:** Os logs e o endpoint `/status` complementam as métricas, fornecendo diferentes perspectivas sobre a saúde e o comportamento do sistema.

---

### 9.4. Verificações de Saúde dos Containers (Healthchecks)

Conforme descrito na Seção 4 (Docker-Compose & Limites), os healthchecks configurados para os serviços Docker são uma forma proativa de observabilidade.

*   **Função:** Permitem ao Docker (e ao `self_monitor.py`) determinar se um container está funcional, não apenas em execução.
*   **Status:** O status de saúde (`healthy`, `unhealthy`) é visível via `docker ps` e pode ser usado como métrica ou gatilho para alertas.

---

### 9.5. Alertas e Notificações

Conforme detalhado na Seção `12.9`, um sistema de alertas robusto notifica os operadores sobre problemas significativos, erros ou condições anormais.

*   **Função:** Traduz eventos de log, métricas anômalas e falhas de healthcheck em notificações acionáveis.
*   **Canais:** Slack, Telegram, email, etc.
*   **Integração:** Os alertas são uma forma ativa de observabilidade, chamando a atenção para problemas que requerem intervenção ou conhecimento.

---

### 9.6. (Consideração Futura Avançada) Distributed Tracing

*   **Conceito:** Em sistemas distribuídos complexos, o rastreamento distribuído (e.g., com OpenTelemetry, Jaeger, Zipkin) permite seguir uma única requisição ou fluxo de trabalho através de múltiplos serviços, medindo latências em cada etapa e visualizando o caminho completo.
*   **Aplicabilidade ao VV-AI-System:** Para o fluxo de processamento de um vídeo (`input` -> `scan` -> `summarize` -> `logline`), o rastreamento distribuído poderia fornecer insights valiosos sobre gargalos.
*   **Status Atual:** Provavelmente um exagero para a escala atual, mas é uma técnica a ser considerada se o sistema crescer em número de microsserviços e complexidade de interações. A instrumentação do código seria necessária.

---

Ao combinar logs estruturados, um endpoint de status informativo, métricas detalhadas, healthchecks e um sistema de alertas eficaz, o `VV-Video-AI-System` pode alcançar um alto nível de observabilidade, facilitando a operação, manutenção e evolução contínua.
