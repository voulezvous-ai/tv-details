```markdown name=VV-Video-AI-System-Master-Blueprint-v2.2.md
# VV-Video-AI-System — Master Blueprint v2.2

> **“Um só repositório. Um só comando. Operação 24/7. Hands-off. Edge devices. Robusto. Auditável.”**

---

## 📂 Estrutura Completa do Repositório

```
vv-video-ai-system/
├── README.md                     ← Blueprint completo com detalhes de implantação
├── CHANGELOG.md                  ← Histórico de versões e notas de release
├── vv-signature.json             ← Manifesto de hashes de integridade
├── install.sh                    ← Script de instalação idempotente
├── bootstrap.py                  ← Setup inicial do ambiente e assets
├── vv-video-ai.service           ← Unit systemd para iniciar no boot
├── vv-video-ai.timer             ← Timer systemd para delay de boot opcional
├── docker-compose.yml            ← Orquestração de containers com limites e healthchecks
├── Makefile                      ← Comandos convenientes (status, logs, test, simulate)
├── .env.example                  ← Template de variáveis de ambiente
│
├── agent_dvr/                    ← Container "video_capture" (DVR agent)
│   └── Dockerfile
│
├── scan_video/                   ← Container "scene_scanner"
│   ├── Dockerfile
│   └── scan_video.py             ← Script de detecção (YOLO, faces)
│
├── summarize_scene/              ← Container "scene_summarizer"
│   ├── Dockerfile
│   ├── summarize_scene.py        ← Script de IA local (BitNet)
│   └── prompts/                  ← Prompt externo versionado
│       └── scene_summarizer.prompt
│
├── logline_generator/            ← Container "logline_writer"
│   ├── Dockerfile
│   └── logline_generator.py      ← Script de geração e assinatura de LogLines
│
├── tv_scheduler/                 ← Container "playlist_manager"
│   ├── Dockerfile
│   ├── tv_scheduler.py           ← Script de montagem de playlists cronológicas
│   └── config.yaml               ← Regras para cada TV (curado, alerta, experimental)
│
├── tv_ui/                        ← Container "tv_api" + UI Curado
│   └── tv.html                   ← Frontend React/HTML para exibição
│
├── tv_mirror_ui/                 ← Container para modo espelho (MJPEG)
│   └── tv_mirror.html            ← Frontend espelho sem gravar
│
├── vv_ingest/                    ← Pipeline de ingestão de conteúdo adicional
│   ├── process_video.py          ← Conversão, padronização, thumb
│   ├── catalog_videos.py         ← Geração de manifesto JSON
│   ├── auto_curate.py            ← Seleção rotativa de acervo
│   └── config.yaml               ← Parâmetros de curadoria
│
├── fallback/                     ← Assets de fallback garantido
│   ├── loop_silencioso.mp4       ← Vídeo silencioso em loop
│   └── placeholder.png           ← Imagem para modo espelho
│
├── schemas/                      ← Contratos JSON para validação
│   ├── analytic.schema.json      ← Estrutura do JSON gerado em scan
│   ├── summary.schema.json       ← Estrutura dos summaries da IA
│   ├── logline.schema.json       ← Estrutura das LogLines
│   └── contract.signature.json   ← Formato de assinatura HMAC
│
├── vv-core/                      ← Módulo de utilitários compartilháveis
│   ├── __init__.py
│   └── utils.py                  ← hashing, validação de schema, HMAC
│
├── scripts/                      ← Scripts auxiliares e de orquestração
│   ├── orchestrator.py           ← Orquestração ponta-a-ponta (cron a cada 5 min)
│   ├── self_monitor.py           ← Self-healing: reinício de services com erros ≥ 3
│   ├── logline_verifier.py       ← CLI para validar assinatura de LogLines
│   ├── log_exporter.py           ← Exportação tar.gz + manifesto de logs
│   └── cleanup_old_files.py      ← Cleanup seguro de arquivos antigos (CLI)
│
└── tests/                        ← Cobertura de testes automatizados
    ├── test_self_monitor.py
    ├── test_logline_verifier.py
    ├── test_log_exporter.py
    ├── test_cleanup_old_files.py
    └── test_orchestrator.py
```

---

## 1. INSTALAÇÃO & BOOTSTRAP 100% AUTOMÁTICOS

### 1.1 One-liner de instalação

```bash
curl -fsSL https://raw.githubusercontent.com/voulezvous-ai/vv-video-ai-system/main/install.sh | sudo bash
```

### 1.2 Descrição de `install.sh`

*   **Auto-elevação**: re-execução com sudo se necessário.
*   **Instalação silenciosa** de dependências (docker, compose, git, python3, ffmpeg, jq).
*   **Clone/Recreate** idempotente do repo em `/opt/vv-video-ai-system`.
*   Execução de `python3 bootstrap.py --skip-existing`.
*   **Validações pós-bootstrap**: existência de `loop_silencioso.mp4` e configuração de `DVR_IP`.
*   `docker-compose pull` e `docker-compose up -d --build`.
*   Geração de **systemd unit** (`vv-video-ai.service`) e opcional **timer** (`vv-video-ai.timer`).
*   **Cronjobs idempotentes**:
    *   `orchestrator.py` a cada 5 minutos.
    *   `cleanup_old_files.py --dry-run` a cada 10 minutos (Ver Seção 12.3 para configuração efetiva).
    *   `log_exporter.py` diariamente às 02:00.

---

## 2. BOOTSTRAP (`bootstrap.py`)

1.  Criação de diretórios em `${VIDEO_STORAGE_ROOT}`:
    `input`, `raw`, `processed`, `tv_curado`, `analytic`, `summaries`, `loglines`, `playlists`, `fallback`, `corrupted`.
2.  Clonagem do modelo BitNet local (quantização 4-bit) se ausente (Ver Seção 12.6 para gerenciamento do modelo).
3.  Cópia de assets de **fallback**.
4.  Geração de `.env` a partir de `.env.example` (Ver Seção 12.7 para gestão de segredos).

---

## 3. ORQUESTRAÇÃO (`scripts/orchestrator.py`)

*   Loop contínuo que monitora **pasta de vídeos** (`*.mp4`).
*   Para cada vídeo novo:
    1.  `scan_video.py` dentro do container `scene_scanner`.
    2.  `summarize_scene.py` dentro de `scene_summarizer`.
    3.  `logline_generator.py` dentro de `logline_writer`.
*   Evita reprocessamento usando set interno de nomes (Ver Seção 12.4 para persistência de estado).
*   Intervalo configurável via `ORCHESTRATOR_INTERVAL`.

---

## 4. DOCKER-COMPOSE & LIMITES

*   Serviços com **mem\_limit** e **cpus** definidos.
*   **Healthchecks** para `scene_scanner`, `playlist_manager` e `tv_api`.
*   `restart: on-failure` e `restart: always` onde aplicável.
*   `watchtower` para atualizações automáticas de imagens internas (Ver Seção 12.5 para gerenciamento avançado).

---

## 5. MÓDULOS PRINCIPAIS

### 5.1 `scan_video.py`

*   Captura frames a cada X segundos.
*   Detecta objetos (YOLOv8) e rostos (face\_recognition).
*   Adiciona `timestamp`, `type`, `label`.
*   Gera JSON atômico com `source_hash` (SHA256 do vídeo).
*   Valida com `analytic.schema.json`.

### 5.2 `summarize_scene.py`

*   Carrega BitNet local (usa CPU/GPU automaticamente).
*   Prompt externo para interpretação semântica.
*   Gera `*.summary.json` atômico, inclui `source_hash`.
*   Valida com `summary.schema.json`.

### 5.3 `logline_generator.py`

*   Input: `.summary.json`.
*   Gera LogLine com campos: `who`, `did`, `this`, `when` (UTC Z), `status`, `source`.
*   Assina HMAC-SHA256 + `key_id`.
*   Output: `.logline.json` e correspondente `.md` simplificado.
*   Valida com `logline.schema.json`.

### 5.4 `tv_scheduler.py`

*   Lê `loglines/` e filtra eventos das últimas 24h.
*   Regras definidas em `config.yaml`:
    *   `tv1`: `confirmado`, `curado`
    *   `tv2`: `alerta`
    *   `tv3`: `experimental`, `curado` (com shuffle).
*   Gera `playlist.json` atômico.
*   Roda a cada 60s (Ver Seção 12.11 para otimização de frequência).

### 5.5 Frontends

*   **tv.html**: UI curado, cross-fade, fallback silencioso.
*   **tv\_mirror.html**: Modo espelho MJPEG, retry e fallback de imagem.

---

## 6. SCRIPTS AUXILIARES

*   **orchestrator.py**: orquestra todo o pipeline.
*   **self\_monitor.py**: self-healing, reinicia contêineres com erros.
*   **logline\_verifier.py**: verifica integridade de LogLines via HMAC.
*   **log\_exporter.py**: empacota logs em tar.gz e gera manifesto JSON.
*   **cleanup\_old\_files.py**: limpa arquivos antigos, suportando `--days`, `--dirs`, `--dry-run`.

---

## 7. VV-CORE (Utilitários)

*   `compute_sha256(path: Path) -> str`
*   `load_schema(path: Path) -> dict`
*   `validate_schema(data: dict, schema_path: Path) -> None`
*   `sign_payload(payload: bytes, secret: bytes) -> str`
*   `verify_signature(payload: bytes, secret: bytes, signature: str) -> bool`
*   `load_key_store(path: Path) -> dict` (Ver Seção 12.7 para gestão de segredos)

---

## 8. TESTES & CI

*   **pytest** cobrindo scripts e orquestração.
*   **Cobertura ≥ 90%**.
*   **Testes E2E** via Selenium headless (opcional).

---

## 9. OBSERVABILIDADE & STATUS

*   Logs estruturados em `logs/app.log`.
*   Endpoint `/status` em FastAPI.
*   Métricas compatíveis com Prometheus/Grafana (Ver Seção 12.10 para métricas detalhadas).

---

## 10. SEGURANÇA & INTEGRIDADE

*   **HMAC** para LogLines, verificado por `scripts/logline_verifier.py`.
*   **Manifesto** de integridade em `vv-signature.json`.
*   **Proteção de acesso** (NGINX + JWT) configurável externamente (Ver Seção 12.1).

---

## 11. AUTOMAÇÃO & AUTORECUPERAÇÃO

*   **Watchtower**: atualizações automáticas de imagens (Ver Seção 12.5).
*   **Self-monitor LLM**: serviço de autodiagnóstico e remediação (Ver Seção 12.2).
*   **Cronjobs** para manutenção: rebuilds semanais e testes diários (`run_tests.sh`).
*   **Alertas** via webhook (Slack/Telegram) para incidentes (Ver Seção 12.9 para detalhamento).

---

## 12. MELHORIAS PROPOSTAS & EVOLUÇÃO (Pós v2.1)

Esta seção detalha aprimoramentos e tarefas de desenvolvimento futuras identificadas para o `VV-Video-AI-System`, com base na revisão do Master Blueprint v2.1. O objetivo é incrementar ainda mais a robustez, segurança e eficiência operacional do sistema.

### 12.1 🔐 Implementação de Proteção de Acesso (NGINX + JWT)

*   **Objetivo**: Concluir a tarefa pendente `Proteção de acesso (NGINX + JWT)` para assegurar todos os endpoints e UIs expostos.
*   **Tarefas**:
    1.  **Definir Escopo**: Identificar todos os serviços que requerem proteção (e.g., `tv_ui/tv.html`, `tv_mirror_ui/tv_mirror.html`, endpoint `/status` da FastAPI).
    2.  **Configurar NGINX**: Implementar NGINX como reverse proxy.
        *   Configurar SSL/TLS para HTTPS.
        *   Gerenciar redirecionamentos e cabeçalhos de segurança.
    3.  **Integração JWT**:
        *   Estabelecer um serviço ou mecanismo para geração e validação de JSON Web Tokens (JWTs).
        *   Proteger rotas no NGINX exigindo um JWT válido.
        *   Definir estratégia para refresh de tokens e logout.
    4.  **Gerenciamento de Chaves**: Assegurar o armazenamento e rotação seguros das chaves secretas do JWT.
*   **Impacto Esperado**: Aumento significativo da segurança da aplicação, prevenindo acesso não autorizado.
*   **Checklist Item Referência**: `| Proteção de acesso (NGINX/JWT) | ⬜️ → Em Andamento |`

### 12.2 🧠 Desenvolvimento do Self-Monitor LLM (Opcional)

*   **Objetivo**: Explorar e, se viável, implementar o `Self-monitor LLM` para diagnósticos avançados e remediação autônoma.
*   **Tarefas**:
    1.  **Definir Escopo e Capacidades**:
        *   **Diagnóstico**: Análise de logs (`logs/app.log`) para padrões de erro complexos, verificação de integridade do pipeline (e.g., arquivos órfãos, falhas de processamento sequenciais), monitoramento de utilização de recursos vs. benchmarks.
        *   **Remediação**: Ações além de reinícios simples (e.g., ajuste dinâmico de `cpus`/`mem_limit` em `docker-compose.yml` se um container estiver sobrecarregado, limpeza seletiva de caches, rollback para uma versão estável de um componente se aplicável, escalonamento de alertas com diagnósticos detalhados).
    2.  **Seleção do LLM**: Avaliar modelos (locais ou API) considerando custo, performance, e requisitos de privacidade/segurança.
    3.  **Integração**:
        *   Como o LLM acessará os dados necessários (logs, status dos containers, métricas)?
        *   Como as ações de remediação serão executadas de forma segura?
    4.  **Recursos**: Estimar os recursos computacionais adicionais necessários para o LLM.
*   **Impacto Esperado**: Maior autonomia do sistema, redução do tempo de inatividade e intervenção manual.
*   **Checklist Item Referência**: `| Self-monitor LLM (opcional) | ⬜️ → Em Avaliação |`

### 12.3 🧹 Configuração Efetiva do Cronjob `cleanup_old_files.py`

*   **Objetivo**: Assegurar que a limpeza de arquivos antigos ocorra efetivamente, e não apenas em modo de simulação.
*   **Contexto**: O `install.sh` configura `cleanup_old_files.py --dry-run`.
*   **Ação Requerida**:
    1.  Revisar a política de retenção de dados para cada diretório em `${VIDEO_STORAGE_ROOT}` e `logs/`.
    2.  Modificar ou adicionar um cronjob para executar `cleanup_old_files.py` *sem* a flag `--dry-run` com a frequência e parâmetros desejados.
        ```bash
        # Exemplo: Limpeza diária de arquivos com mais de 30 dias nos diretórios especificados
        0 3 * * * /usr/bin/python3 /opt/vv-video-ai-system/scripts/cleanup_old_files.py --days 30 --dirs "/opt/vv-video-ai-system/processed,/opt/vv-video-ai-system/logs/app.log" >> /opt/vv-video-ai-system/logs/cleanup.log 2>&1
        ```
*   **Impacto Esperado**: Gerenciamento eficiente do espaço em disco, prevenindo esgotamento.

### 12.4 💾 Persistência de Estado do `orchestrator.py`

*   **Objetivo**: Garantir a idempotência do `orchestrator.py` mesmo após reinícios do container.
*   **Contexto**: Atualmente, o `orchestrator.py` "evita reprocessamento usando set interno de nomes", que é volátil.
*   **Sugestões de Implementação**:
    1.  **Arquivo de Estado**: Salvar o set de hashes/nomes de arquivos processados em um arquivo simples (e.g., `processed_videos.log`) no volume persistente.
    2.  **Banco de Dados Leve**: Utilizar um banco de dados SQLite armazenado em volume persistente para maior robustez.
    3.  **Movimentação de Arquivos**: Mover arquivos processados para um subdiretório específico (e.g., `input/processed/`) e o orquestrador apenas monitoraria `input/new/`.
*   **Impacto Esperado**: Prevenção de reprocessamento desnecessário, economizando recursos e garantindo consistência.

### 12.5 🔄 Gerenciamento Avançado de Imagens com Watchtower e CI/CD

*   **Objetivo**: Otimizar o processo de atualização de imagens Docker, especialmente as construídas localmente.
*   **Contexto**: `watchtower` é ideal para imagens de registros externos. Para imagens internas (`agent_dvr`, `scan_video`, etc.), o "rebuild semanal" pode não ser o ideal.
*   **Sugestões de Implementação**:
    1.  **Registro Privado**: Configurar um registro Docker privado (e.g., Harbor, Docker Hub privado, GitHub Container Registry).
    2.  **Pipeline CI/CD**:
        *   Utilizar GitHub Actions (ou similar) para construir as imagens Docker a cada push/merge na `main` branch.
        *   Versionar e enviar as imagens construídas para o registro privado.
        *   `watchtower` pode então monitorar o registro privado para atualizações.
    3.  **Alternativa**: Se um CI/CD completo for excessivo, aprimorar o script de "rebuild semanal" para:
        *   Detectar mudanças nos `Dockerfile`s ou código fonte relevante.
        *   Reconstruir (`docker-compose build --no-cache <service_name>`) apenas as imagens necessárias.
        *   Forçar a recriação dos containers (`docker-compose up -d --force-recreate <service_name>`).
*   **Impacto Esperado**: Atualizações mais rápidas e confiáveis, melhor versionamento de imagens, e infraestrutura como código mais robusta.

### 12.6 🤖 Estratégia de Gerenciamento do Modelo BitNet

*   **Objetivo**: Definir um ciclo de vida claro para o modelo BitNet local.
*   **Tarefas**:
    1.  **Versionamento**: O modelo clonado pelo `bootstrap.py` deve ser versionado. Considerar Git LFS para o repositório do modelo ou armazenar checksums do modelo esperado.
    2.  **Atualizações**: Como novas versões do modelo BitNet serão testadas e implantadas? O `bootstrap.py` deve ser capaz de buscar uma versão específica.
    3.  **Rollback**: Definir um processo para reverter para uma versão anterior do modelo em caso de problemas.
    4.  **Monitoramento**: Coletar métricas sobre a performance (acurácia, tempo de inferência) do modelo em produção.
*   **Impacto Esperado**: Maior controle sobre o componente de IA, facilitando atualizações e troubleshooting.

### 12.7 🔑 Aprimoramento da Gestão de Segredos

*   **Objetivo**: Implementar práticas mais seguras para o gerenciamento de segredos (HMAC keys, JWT secrets, etc.).
*   **Contexto**: O uso de `.env` é bom para desenvolvimento, mas menos ideal para produção.
*   **Sugestões de Implementação**:
    1.  **Docker Secrets**: Para segredos usados por containers Docker.
    2.  **HashiCorp Vault**: Solução dedicada para gerenciamento centralizado de segredos.
    3.  **Variáveis de Ambiente Injetadas**: Injetar segredos como variáveis de ambiente no momento da implantação através de um sistema CI/CD seguro ou ferramentas de orquestração.
    4.  **`vv-core/utils.py` `load_key_store`**: Assegurar que o `key_store` referenciado seja armazenado de forma segura e com permissões restritas.
*   **Impacto Esperado**: Redução do risco de exposição de segredos, conformidade com melhores práticas de segurança.

### 12.8 🛡️ Estratégia de Backup e Recuperação de Dados

*   **Objetivo**: Definir e documentar um plano robusto de backup e recuperação para dados críticos.
*   **Dados Críticos a Considerar**:
    1.  `${VIDEO_STORAGE_ROOT}`: `input/`, `raw/`, `processed/`, `tv_curado/`, `analytic/`, `summaries/`, `loglines/`, `playlists/`.
    2.  Logs: `logs/app.log`, `logs/cleanup.log`, logs de containers (se persistidos).
    3.  Estado Persistente: Arquivo de estado do `orchestrator.py` (ver 12.4), banco de dados SQLite se usado.
    4.  Configurações: `.env`, `config.yaml` dos módulos, `vv-signature.json`.
*   **Tarefas**:
    1.  **Ferramentas**: Selecionar ferramentas de backup (e.g., `rsync`, `restic`, soluções de cloud storage).
    2.  **Frequência e Retenção**: Definir RPO (Recovery Point Objective) e RTO (Recovery Time Objective).
    3.  **Teste de Recuperação**: Agendar testes periódicos do processo de restauração.
*   **Impacto Esperado**: Capacidade de recuperar o sistema e seus dados em caso de falha catastrófica, corrupção de dados ou desastre.

### 12.9 🚨 Detalhamento do Tratamento de Erros e Alertas

*   **Objetivo**: Expandir a granularidade e a utilidade do sistema de tratamento de erros e alertas.
*   **Contexto**: `self_monitor.py` e webhooks (Slack/Telegram) já existem.
*   **Sugestões de Aprimoramento**:
    1.  **Gatilhos de Alerta Específicos**:
        *   Falha persistente de um container específico (após N tentativas de `self_monitor.py`).
        *   Erro de validação de schema (`analytic.schema.json`, `summary.schema.json`, `logline.schema.json`).
        *   Falha na assinatura/verificação HMAC.
        *   Espaço em disco em `${VIDEO_STORAGE_ROOT}` abaixo de um limiar crítico.
        *   Processamento de um vídeo excedendo um tempo máximo esperado.
        *   Falhas de conexão com `DVR_IP`.
    2.  **Alertas Acionáveis**: Garantir que cada alerta contenha contexto suficiente (e.g., nome do arquivo, container, timestamp, trecho do log relevante) para diagnóstico rápido.
    3.  **Tratamento de Falhas em Scripts Python**: Implementar blocos `try-except` mais robustos em todos os scripts (`scan_video.py`, `summarize_scene.py`, etc.) para capturar exceções específicas, logar detalhadamente e, se necessário, mover arquivos problemáticos para o diretório `corrupted/` com um arquivo de metadados do erro.
*   **Impacto Esperado**: Resolução de problemas mais rápida, melhor entendimento da saúde do sistema, e maior resiliência a dados malformados ou falhas inesperadas.

### 12.10 📊 Métricas Detalhadas para Monitoramento de Recursos (Prometheus/Grafana)

*   **Objetivo**: Enriquecer o conjunto de métricas para uma observabilidade mais profunda.
*   **Contexto**: "Métricas compatíveis com Prometheus/Grafana" já planejado.
*   **Métricas Adicionais Sugeridas**:
    *   **Pipeline de Vídeo**:
        *   Número de vídeos na fila de processamento (aguardando `scan_video.py`).
        *   Tempo médio de processamento por estágio (`scan_video`, `summarize_scene`, `logline_generator`).
        *   Taxa de erro por estágio do pipeline.
        *   Número de arquivos em `corrupted/`.
    *   **Recursos de IA**:
        *   Tempo de inferência do modelo BitNet.
        *   Taxa de sucesso/falha das inferências.
        *   Uso de CPU/GPU pelo container `scene_summarizer`.
    *   **Armazenamento**:
        *   Espaço em disco utilizado/livre em `${VIDEO_STORAGE_ROOT}` e seus subdiretórios.
        *   Taxa de crescimento do armazenamento.
    *   **Rede**:
        *   Latência e taxa de erro na comunicação com `DVR_IP`.
    *   **`tv_scheduler.py`**:
        *   Número de playlists geradas.
        *   Tempo para gerar playlists.
*   **Impacto Esperado**: Visão detalhada da performance e gargalos do sistema, permitindo otimizações proativas.

### 12.11 ⏱️ Otimização da Frequência do `tv_scheduler.py`

*   **Objetivo**: Avaliar a necessidade da execução do `tv_scheduler.py` a cada 60 segundos.
*   **Contexto**: Lê `loglines/` das últimas 24h e gera `playlist.json`.
*   **Considerações**:
    1.  **Event-Driven**: Poderia o `tv_scheduler.py` ser acionado por um evento quando um novo `.logline.json` é criado? Isso reduziria a carga de verificações constantes.
    2.  **Frequência Reduzida**: Se a atualização da playlist não precisa ser em tempo real (minuto a minuto), uma frequência menor (e.g., a cada 5 ou 15 minutos) poderia economizar recursos, especialmente se a leitura e filtragem dos loglines for intensiva.
    3.  **Impacto da Mudança**: Avaliar o quão "fresca" a playlist precisa ser para os frontends `tv.html`.
*   **Impacto Esperado**: Potencial redução no uso de CPU e I/O, otimizando recursos.

### 12.12 📖 Documentação Operacional e de Usuário

*   **Objetivo**: Criar documentação para operadores do sistema e, se aplicável, usuários finais.
*   **Conteúdo Sugerido**:
    1.  **Guia do Operador**:
        *   Troubleshooting de problemas comuns (e.g., containers não iniciam, vídeos não são processados, alertas comuns).
        *   Interpretação de logs (`app.log`, logs de containers).
        *   Procedimentos para intervenções manuais (e.g., forçar reprocessamento, limpar estado).
        *   Como verificar a integridade do sistema (`vv-signature.json`, `logline_verifier.py`).
        *   Procedimento de backup e restauração.
    2.  **Guia de Configuração Avançada**:
        *   Detalhes sobre `config.yaml` para `tv_scheduler` e `vv_ingest`.
        *   Como adicionar/modificar prompts para `summarize_scene.py`.
    3.  **(Opcional) Guia do Usuário**: Se houver usuários interagindo com as UIs de forma mais complexa do que apenas visualização.
*   **Formato**: Markdown no repositório (e.g., em uma pasta `docs/`), Wiki do GitHub.
*   **Impacto Esperado**: Facilita a manutenção, onboarding de novos membros da equipe, e reduz a dependência de conhecimento tácito.

---

## 🗸 CHECKLIST FINAL (Atualizado v2.2)

| Item                                                            | Status          | Notas                                     |
| --------------------------------------------------------------- | :-------------: | ----------------------------------------- |
| Instalação automática + Systemd + Timer                         |        ✅        |                                           |
| Bootstrap idempotente                                           |        ✅        |                                           |
| Containers configurados (limites & healthchecks)                |        ✅        |                                           |
| Pipeline ponta-a-ponta (Orchestrator)                           |        ✅        | Ver 12.4 para persistência de estado      |
| Módulos principais (scan, summary, logline)                     |        ✅        |                                           |
| Frontends UI (curado & espelho)                                 |        ✅        |                                           |
| Scripts auxiliares resilientes                                  |        ✅        |                                           |
| VV-Core utils                                                   |        ✅        | Ver 12.7 para gestão de segredos          |
| Testes automatizados                                            |        ✅        |                                           |
| Observabilidade & Status API                                    |        ✅        | Ver 12.10 para métricas detalhadas        |
| Segurança: HMAC & manifesto                                     |        ✅        |                                           |
| Cronjobs & Watchtower                                           |        ✅        | Ver 12.3 (cleanup), 12.5 (watchtower)    |
| Proteção de acesso (NGINX/JWT)                                  | 📝 Em Andamento | Prioridade alta (Seção 12.1)              |
| Self-monitor LLM (opcional)                                     | 🤔 Em Avaliação | Avaliar viabilidade (Seção 12.2)          |
| **Novos Itens de Melhoria (v2.2):**                             |                 |                                           |
| Configuração Efetiva do `cleanup_old_files.py`                  |       📝        | Ação requerida (Seção 12.3)               |
| Persistência de Estado do `orchestrator.py`                     |       📝        | Ação requerida (Seção 12.4)               |
| Gerenciamento Avançado de Imagens (CI/CD)                       | 💡 Sugestão     | Para evolução (Seção 12.5)                |
| Estratégia de Gerenciamento do Modelo BitNet                    |       📝        | Ação requerida (Seção 12.6)               |
| Aprimoramento da Gestão de Segredos                             | 💡 Sugestão     | Para evolução (Seção 12.7)                |
| Estratégia de Backup e Recuperação de Dados                     |       📝        | Ação requerida (Seção 12.8)               |
| Detalhamento do Tratamento de Erros e Alertas                   | 💡 Sugestão     | Para evolução (Seção 12.9)                |
| Otimização da Frequência do `tv_scheduler.py`                   | 🤔 Em Avaliação | Avaliar necessidade (Seção 12.11)         |
| Documentação Operacional e de Usuário                           |       📝        | Ação requerida (Seção 12.12)              |

**Legenda do Status:**
*   ✅: Concluído
*   📝: Em Andamento / Ação Requerida
*   🤔: Em Avaliação / Consideração
*   💡: Sugestão / Para Evolução Futura

> **Versão**: v2.2 — 2025-05-14 (com propostas de evolução)
> **Pronto para produção**: auditável, resiliente e hands-off, com um roadmap claro para melhorias contínuas.

```