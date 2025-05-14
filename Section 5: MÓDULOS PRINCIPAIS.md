Okay, let's dive into **Section 5: MÓDULOS PRINCIPAIS**. This section is a core part of the blueprint, describing the key processing scripts.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
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
```

This provides a good summary of each module. We can enrich this by:

1.  **For each Python script (`5.1` to `5.4`):**
    *   **Inputs:** Be more specific about input arguments (e.g., input video path, output JSON path).
    *   **Outputs:** Detail the structure or key fields of the output JSONs/files more, beyond just the filename.
    *   **Core Logic:** Briefly expand on the main processing steps within the script.
    *   **Dependencies:** Mention key Python libraries used (beyond the obvious like `json`, `os`).
    *   **Error Handling:** How errors are handled internally (e.g., logging, moving to `corrupted/` as managed by orchestrator, specific exceptions raised). Refer to `12.9`.
    *   **Configuration:** How is it configured (env vars, command-line args, config files)?
    *   **Performance Considerations:** Any notes on performance or resource usage.
    *   **Metrics Exposition:** How it might contribute metrics (ref `12.10`).
2.  **For Frontends (`5.5`):**
    *   **Technology Stack:** Briefly mention (e.g., "Plain HTML/JS/CSS", "React", "Vue").
    *   **Data Source:** How they fetch data (e.g., `playlist.json` for `tv.html`, MJPEG stream URL for `tv_mirror.html`).
    *   **Key Features:** Expand slightly on "cross-fade," "fallback," "retry."
    *   **Configuration:** Any client-side or server-side (NGINX) configurations needed.
    *   **Security:** Briefly mention JWT protection as per `12.1` for accessing these UIs.

Let's start with `5.1 scan_video.py` and go through each one.

---

## 5. MÓDULOS PRINCIPAIS

Esta seção descreve os componentes de software fundamentais que formam o pipeline de processamento e exibição do `VV-Video-AI-System`. Cada módulo é encapsulado em seu próprio container Docker para isolamento e gerenciamento.

---

### 5.1. `scan_video.py` (Serviço: `scene_scanner`)

Este script é a primeira etapa de análise do pipeline, responsável por processar um arquivo de vídeo para extrair informações visuais básicas como objetos e faces.

*   **Responsabilidade Primária:** Analisar frames de um vídeo para detectar objetos e faces, gerando um arquivo JSON estruturado com esses eventos e seus timestamps.
*   **Inputs:**
    *   Argumento de linha de comando: Caminho para o arquivo de vídeo de entrada (e.g., `/app/input/video_a.mp4`).
    *   Argumento de linha de comando: Caminho para o arquivo JSON de saída (e.g., `/app/analytic/video_a.analytic.json`).
    *   Configuração (via variáveis de ambiente ou arquivo de config):
        *   `FRAME_CAPTURE_INTERVAL_SECONDS`: Intervalo em segundos para captura de frames (e.g., `1` para 1 frame por segundo).
        *   `YOLO_MODEL_VERSION`: Versão/tipo do modelo YOLO a ser usado.
        *   `FACE_RECOGNITION_THRESHOLD`: Limiar de confiança para detecção de faces.
*   **Outputs:**
    *   Um único arquivo JSON (`<video_nome_base>.analytic.json`) no diretório de saída especificado.
    *   **Estrutura do JSON de Saída (Exemplo de `analytic.schema.json`):**
        ```json
        {
          "source_video_path": "/app/input/video_a.mp4",
          "source_video_hash_sha256": "abcdef123...", // SHA256 do arquivo de vídeo original
          "processing_timestamp_utc": "2025-05-15T12:00:00Z",
          "duration_seconds": 120.5,
          "frames_analyzed": 120,
          "events": [
            {
              "frame_timestamp_seconds": 10.2, // Timestamp dentro do vídeo
              "event_type": "object_detection", // "object_detection" ou "face_detection"
              "label": "car", // Label do objeto (YOLO) ou "face"
              "confidence": 0.85, // Confiança da detecção
              "bounding_box": [100, 150, 50, 70] // [x, y, width, height]
              // "face_embedding": [0.1, ..., 0.9] // (Opcional) Se face embeddings forem extraídos
            },
            // ... mais eventos
          ]
        }
        ```
*   **Core Logic:**
    1.  Calcula o hash SHA256 do arquivo de vídeo de entrada.
    2.  Abre o arquivo de vídeo usando uma biblioteca como OpenCV (`cv2`).
    3.  Itera sobre os frames do vídeo no intervalo especificado (`FRAME_CAPTURE_INTERVAL_SECONDS`).
    4.  Para cada frame capturado:
        *   Realiza detecção de objetos usando um modelo YOLO (e.g., YOLOv8).
        *   Realiza detecção de faces usando uma biblioteca como `face_recognition` ou um detector de faces do OpenCV.
        *   Para cada detecção válida (acima de um limiar de confiança), registra um evento com `timestamp` do frame, `type` (`object` ou `face`), `label` (e.g., "carro", "pessoa", "face"), `confidence` e `bounding_box`.
    5.  Agrega todos os eventos detectados em uma estrutura JSON.
    6.  Grava o JSON de forma atômica (escreve em um arquivo temporário e depois renomeia) no caminho de saída especificado.
    7.  Valida o JSON gerado contra o `schemas/analytic.schema.json` usando `vv-core/utils.py`. Se a validação falhar, loga um erro e potencialmente sai com um código de erro.
*   **Dependências Chave (Exemplos):** `opencv-python`, `ultralytics` (para YOLOv8), `face_recognition`, `numpy`.
*   **Error Handling (referência `12.9`):**
    *   Captura exceções durante o processamento do vídeo (e.g., arquivo de vídeo corrompido, falha na carga do modelo).
    *   Loga erros detalhados.
    *   Se ocorrer um erro irrecuperável, o script sai com um código de erro não zero, sinalizando falha para o `orchestrator.py`.
*   **Performance Considerations:**
    *   A análise de vídeo é CPU-intensiva (e potencialmente GPU-intensiva se YOLO/face_recognition usarem GPU).
    *   A frequência de captura de frames impacta diretamente o tempo de processamento.
    *   Os limites de `cpus` e `mem_limit` no `docker-compose.yml` são cruciais.
*   **Metrics Exposition (referência `12.10`):**
    *   Se executado como um serviço de longa duração (menos provável para esta tarefa), poderia expor métricas como `vv_scan_video_processed_total`, `vv_scan_video_duration_seconds`.
    *   Se chamado pelo orquestrador, o orquestrador é quem mais provavelmente registraria essas métricas.

---

### 5.2. `summarize_scene.py` (Serviço: `scene_summarizer`)

Este script utiliza o modelo BitNet local para realizar uma interpretação semântica dos eventos detectados por `scan_video.py`, gerando um sumário textual.

*   **Responsabilidade Primária:** Processar o JSON de análise de cena e, usando um modelo de linguagem (BitNet), gerar um sumário textual descritivo e estruturado da cena ou dos eventos significativos.
*   **Inputs:**
    *   Argumento de linha de comando: Caminho para o arquivo JSON de análise de entrada (e.g., `/app/analytic/video_a.analytic.json`).
    *   Argumento de linha de comando: Caminho para o arquivo JSON de sumário de saída (e.g., `/app/summaries/video_a.summary.json`).
    *   Arquivo de prompt: Um arquivo de texto externo (e.g., `prompts/scene_summarizer.prompt`) que guia o modelo LLM.
    *   Modelo BitNet: Carregado do sistema de arquivos local (e.g., de `/app/models/bitnet_summarizer/`, conforme Seção `12.6`).
    *   Configuração (via variáveis de ambiente ou arquivo de config):
        *   `BITNET_MODEL_PATH`: Caminho para o diretório do modelo BitNet.
        *   `PROMPT_FILE_PATH`: Caminho para o arquivo de prompt.
        *   `MAX_TOKENS_SUMMARY`: Limite de tokens para o sumário gerado.
*   **Outputs:**
    *   Um único arquivo JSON (`<video_nome_base>.summary.json`) no diretório de saída especificado.
    *   **Estrutura do JSON de Saída (Exemplo de `summary.schema.json`):**
        ```json
        {
          "source_analytic_json_path": "/app/analytic/video_a.analytic.json",
          "source_video_hash_sha256": "abcdef123...", // Copiado do analytic.json
          "processing_timestamp_utc": "2025-05-15T12:05:00Z",
          "model_name": "BitNet-4bit-Summarizer",
          "model_version": "1.0.0", // Obtido do gerenciamento de modelos (12.6)
          "prompt_used_hash_sha256": "fedcba987...", // Hash do conteúdo do prompt usado
          "summary_text": "Uma pessoa caminhou em direção a um carro vermelho às 10:02. O carro partiu às 10:05.",
          "keywords": ["pessoa", "carro", "movimento"],
          "sentiment": "neutro", // (Opcional) Análise de sentimento do sumário
          "confidence_score": 0.92 // (Opcional) Confiança do modelo no sumário gerado
        }
        ```
*   **Core Logic:**
    1.  Lê o arquivo `analytic.json` de entrada.
    2.  Carrega o modelo BitNet quantizado de 4 bits do caminho especificado. O script deve ser capaz de detectar e utilizar CPU ou GPU disponível automaticamente, se o backend do modelo permitir.
    3.  Lê o arquivo de prompt externo.
    4.  Formata os dados do `analytic.json` (e.g., uma lista de eventos chave) para serem incluídos no prompt.
    5.  Envia o prompt formatado para o modelo BitNet para inferência.
    6.  Recebe a resposta do modelo (o sumário textual).
    7.  (Opcional) Realiza pós-processamento no sumário (e.g., extração de keywords, análise de sentimento).
    8.  Constrói a estrutura JSON de saída, incluindo o hash do vídeo original, informações do modelo e do prompt.
    9.  Grava o JSON de forma atômica no caminho de saída.
    10. Valida o JSON gerado contra o `schemas/summary.schema.json`.
*   **Dependências Chave (Exemplos):** Biblioteca para carregar e executar o modelo BitNet (e.g., `transformers`, `bitsandbytes`, custom C++ bindings), `json`.
*   **Error Handling:**
    *   Falha ao carregar o modelo BitNet ou o arquivo de prompt.
    *   Erro durante a inferência do modelo.
    *   Loga erros e sai com código de erro não zero para o `orchestrator.py`.
*   **Performance Considerations:**
    *   A inferência LLM pode ser a etapa mais demorada e consumir mais recursos (CPU/GPU e memória).
    *   O tamanho do modelo BitNet e a complexidade do prompt afetam a performance.
    *   Limites de `mem_limit` e `cpus` (e configuração de GPU) são críticos.
*   **Metrics Exposition:**
    *   O orquestrador ou o próprio script (se serviço) pode expor `vv_summarize_scene_duration_seconds`, `vv_summarize_scene_bitnet_inference_duration_seconds`.

---

### 5.3. `logline_generator.py` (Serviço: `logline_writer`)

Este script transforma o sumário da IA em uma LogLine estruturada e assinada, criando um registro auditável.

*   **Responsabilidade Primária:** Gerar uma LogLine concisa e estruturada a partir do sumário da IA, assinar digitalmente essa LogLine para garantir integridade e autenticidade, e salvar tanto a LogLine JSON quanto uma versão Markdown legível.
*   **Inputs:**
    *   Argumento de linha de comando: Caminho para o arquivo JSON de sumário de entrada (e.g., `/app/summaries/video_a.summary.json`).
    *   Argumento de linha de comando: Caminho base para os arquivos LogLine de saída (e.g., `/app/loglines/video_a`). O script adicionará `.logline.json` e `.logline.md`.
    *   Chave HMAC: Acessada de forma segura (e.g., variável de ambiente `HMAC_SECRET_KEY`, ou via `vv-core/utils.py load_key_store()` conforme Seção `12.7`).
    *   `KEY_ID`: Identificador da chave HMAC usada (via variável de ambiente).
*   **Outputs:**
    *   Arquivo JSON (`<video_nome_base>.logline.json`) contendo a LogLine estruturada e a assinatura.
    *   Arquivo Markdown (`<video_nome_base>.logline.md`) contendo uma representação simplificada e legível da LogLine.
    *   **Estrutura do JSON de Saída (Exemplo de `logline.schema.json`):**
        ```json
        {
          "log_id": "uuid-v4-random", // UUID único para esta LogLine
          "timestamp_event_utc": "2025-05-15T12:02:30Z", // Timestamp principal do evento sumarizado (extraído do summary)
          "timestamp_generation_utc": "2025-05-15T12:06:00Z", // Quando a LogLine foi gerada
          "source_video_hash_sha256": "abcdef123...", // Copiado do summary.json
          "source_summary_hash_sha256": "ghijkl456...", // Hash do summary.json usado como entrada
          "who": "Pessoa", // Sujeito principal
          "did_what": "Caminhou em direção a um carro", // Ação principal
          "did_to_whom_or_what": "Carro vermelho", // Objeto da ação
          "where": "Local X", // (Opcional, se inferível)
          "status_outcome": "Confirmado", // (e.g., Confirmado, Alerta, Experimental) - pode ser inferido ou fixo
          "details_summary_ref": "video_a.summary.json", // Referência ao sumário
          "signature_details": {
            "key_id": "hmac_key_prod_v1",
            "algorithm": "HMAC-SHA256",
            "signature": "a1b2c3d4e5f6..." // Assinatura HMAC do payload da LogLine
          }
        }
        ```
*   **Core Logic:**
    1.  Lê o arquivo `summary.json` de entrada.
    2.  Calcula o hash SHA256 do conteúdo do `summary.json`.
    3.  Extrai ou infere os campos da LogLine (`who`, `did_what`, `did_to_whom_or_what`, `timestamp_event_utc`, `status_outcome`) a partir do `summary_text` ou outros campos do sumário. Pode usar regex, parsing simples, ou até um prompt LLM mais simples se necessário para essa estruturação.
    4.  Constrói o payload da LogLine (todos os campos exceto `signature_details`).
    5.  Assina o payload da LogLine serializado (e.g., JSON canônico) usando HMAC-SHA256 com a chave secreta e o `key_id` fornecidos.
    6.  Adiciona os `signature_details` (incluindo `key_id`, `algorithm`, e a `signature`) ao objeto LogLine.
    7.  Grava o objeto LogLine completo de forma atômica como `<base_path>.logline.json`.
    8.  Valida o LogLine JSON gerado contra `schemas/logline.schema.json`.
    9.  Gera uma versão Markdown simplificada da LogLine e a salva como `<base_path>.logline.md`.
*   **Dependências Chave:** `json`, `hashlib`, `hmac`, `datetime`.
*   **Error Handling:**
    *   Falha ao carregar a chave HMAC ou `key_id`.
    *   Erro na geração da assinatura.
    *   Loga erros e sai com código de erro não zero para o `orchestrator.py`.
*   **Metrics Exposition:**
    *   O orquestrador pode expor `vv_logline_generated_total`.

---

### 5.4. `tv_scheduler.py` (Serviço: `playlist_manager`)

Este script é responsável por criar playlists dinâmicas para as diferentes interfaces de TV com base nas LogLines geradas.

*   **Responsabilidade Primária:** Ler LogLines recentes, aplicar regras de filtragem e curadoria definidas em um arquivo de configuração, e gerar arquivos `playlist.json` para cada "TV" configurada.
*   **Modo de Execução:** Tipicamente executado como um cronjob (e.g., a cada 60 segundos, mas ver Seção `12.11` para otimização da frequência) ou como um serviço de longa duração reagindo a eventos.
*   **Inputs:**
    *   Diretório `loglines/` (contendo os arquivos `.logline.json`).
    *   Arquivo de configuração `tv_scheduler/config.yaml`.
    *   (Implícito) Vídeos referenciados nas LogLines, localizados em `${VIDEO_STORAGE_ROOT}/processed/` ou `tv_curado/`.
*   **Outputs:**
    *   Um ou mais arquivos `playlist.json` (e.g., `tv1_playlist.json`, `tv2_playlist.json`) no diretório `${VIDEO_STORAGE_ROOT}/playlists/`.
    *   **Estrutura do `playlist.json` (Exemplo):**
        ```json
        {
          "tv_id": "tv1_curated",
          "generation_timestamp_utc": "2025-05-15T13:00:00Z",
          "items": [
            {
              "video_path": "/opt/vv-video-ai-system/storage/processed/video_a.mp4", // Caminho acessível pela UI ou NGINX
              "logline_ref": "video_a.logline.json",
              "summary_text": "Uma pessoa caminhou...",
              "duration_seconds": 60,
              "event_timestamp_utc": "2025-05-15T12:02:30Z"
            },
            // ... mais itens da playlist
          ],
          "fallback_video": "/opt/vv-video-ai-system/storage/fallback/loop_silencioso.mp4"
        }
        ```
*   **Core Logic:**
    1.  Lê o arquivo de configuração `tv_scheduler/config.yaml` para obter as definições de cada TV (filtros, ordenação, etc.).
    2.  Lista e lê os arquivos `.logline.json` do diretório `loglines/`. Filtra aqueles dentro da janela de tempo relevante (e.g., últimas 24 horas).
    3.  Para cada TV definida no `config.yaml`:
        *   Aplica as regras de filtragem às LogLines (e.g., por `status_outcome` como "confirmado", "alerta", ou tags customizadas se adicionadas às LogLines).
        *   Ordena os itens selecionados (e.g., cronologicamente).
        *   Aplica lógica de shuffle se configurado.
        *   Constrói a lista de itens da playlist, incluindo caminhos para os arquivos de vídeo e informações relevantes da LogLine/sumário.
        *   Gera o arquivo `playlist.json` para aquela TV de forma atômica.
*   **`tv_scheduler/config.yaml` (Exemplo):**
    ```yaml
    tvs:
      - id: "tv1_curated"
        output_file: "tv1_playlist.json"
        rules:
          - filter_by_status: ["confirmado", "curado"]
          - sort_by: "event_timestamp_utc"
            order: "desc"
          - max_items: 20
      - id: "tv2_alerts"
        output_file: "tv2_playlist.json"
        rules:
          - filter_by_status: ["alerta"]
          - sort_by: "event_timestamp_utc"
            order: "desc"
          - max_items: 10
      - id: "tv3_experimental"
        output_file: "tv3_playlist.json"
        rules:
          - filter_by_status: ["experimental", "curado"]
          - shuffle: true
          - max_duration_minutes: 60 # Limite total da playlist
    ```
*   **Dependências Chave:** `PyYAML`, `json`, `datetime`.
*   **Error Handling:**
    *   Falha ao ler `config.yaml` ou arquivos LogLine.
    *   Loga erros e continua para a próxima TV se possível, ou sai se for um erro fatal.
*   **Metrics Exposition (referência `12.10`):**
    *   `vv_tv_scheduler_playlists_generated_total`, `vv_tv_scheduler_generation_duration_seconds`.

---

### 5.5. Frontends (`tv_ui/` e `tv_mirror_ui/`)

Estes módulos fornecem as interfaces de usuário para visualização do conteúdo processado.

*   **`tv.html` (Serviço: `tv_api` ou servido por NGINX)**
    *   **Responsabilidade:** Exibir uma playlist de vídeos curados de forma contínua, com transições suaves e um vídeo de fallback.
    *   **Tecnologia (Exemplo):** HTML5, JavaScript (pode ser vanilla JS ou um framework leve como Vue.js/React se a complexidade aumentar), CSS.
    *   **Data Source:** Lê o arquivo `playlist.json` correspondente (e.g., `tv1_playlist.json`) via requisição HTTP GET (servido pelo NGINX ou uma API Python simples no container `tv_api`). Atualiza a playlist periodicamente (e.g., a cada X minutos).
    *   **Key Features:**
        *   **Reprodução Contínua:** Toca vídeos da playlist em sequência.
        *   **Cross-fade:** Transições suaves entre vídeos (se suportado pelo player de vídeo ou implementado via JS).
        *   **Fallback Silencioso:** Se um vídeo da playlist falhar ao carregar ou a playlist estiver vazia, reproduz o `fallback_video` especificado na playlist (e.g., `loop_silencioso.mp4`) para evitar tela preta.
        *   **Atualização Dinâmica:** Recarrega a playlist periodicamente para refletir atualizações do `tv_scheduler.py`.
    *   **Configuração:** URL para o `playlist.json`, URL do vídeo de fallback.
    *   **Segurança (referência `12.1`):** O acesso à página `tv.html` deve ser protegido por NGINX+JWT.

*   **`tv_mirror.html` (Serviço: `tv_mirror_ui` ou servido por NGINX)**
    *   **Responsabilidade:** Fornecer um "espelho" de baixa latência de uma fonte de vídeo, tipicamente para monitoramento em tempo real, sem gravar.
    *   **Tecnologia (Exemplo):** HTML5, JavaScript.
    *   **Data Source:** Exibe um stream MJPEG (Motion JPEG) proveniente de uma URL (e.g., fornecida pelo `agent_dvr` se ele expuser tal stream, ou de outra fonte de vídeo).
        *   A URL do stream MJPEG é configurável.
    *   **Key Features:**
        *   **Exibição de Stream MJPEG:** Usa `<img>` tag com `src` apontando para o stream.
        *   **Retry Logic:** Tenta reconectar ao stream MJPEG se a conexão for perdida.
        *   **Fallback de Imagem:** Se o stream estiver indisponível, exibe uma imagem de placeholder (e.g., `placeholder.png` de `${VIDEO_STORAGE_ROOT}/fallback/`).
    *   **Configuração:** URL do stream MJPEG, URL da imagem de fallback.
    *   **Segurança (referência `12.1`):** O acesso à página `tv_mirror.html` deve ser protegido por NGINX+JWT.

---

Esta versão expandida da Seção 5 detalha mais profundamente as responsabilidades, entradas, saídas, lógica interna e considerações para cada módulo principal, conectando-os com as melhorias e estratégias definidas nas seções `12.x`.
