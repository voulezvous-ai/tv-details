Okay, let's proceed to **Section 3: ORQUESTRAÇÃO (`scripts/orchestrator.py`)**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 3. ORQUESTRAÇÃO (`scripts/orchestrator.py`)

*   Loop contínuo que monitora **pasta de vídeos** (`*.mp4`).
*   Para cada vídeo novo:
    1.  `scan_video.py` dentro do container `scene_scanner`.
    2.  `summarize_scene.py` dentro de `scene_summarizer`.
    3.  `logline_generator.py` dentro de `logline_writer`.
*   Evita reprocessamento usando set interno de nomes (Ver Seção 12.4 para persistência de estado).
*   Intervalo configurável via `ORCHESTRATOR_INTERVAL`.
```

This section describes the core pipeline controller. We can enrich this by:

1.  **Detailing the "Loop contínuo":**
    *   Is it run by cron (as implied in Sec 1) or is it a long-running script itself? The blueprint says "cron a cada 5 min" for `orchestrator.py` in section 1. We should clarify this flow. If cron, it's not exactly a "loop contínuo" in the script itself, but a periodic execution.
    *   How it identifies "novos" vídeos (e.g., by comparing list of files with a persisted set of processed files, as per `12.4`).
2.  **Elaborating on how it calls scripts in other containers:**
    *   `docker-compose run --rm <service_name> python <script.py> <args>`?
    *   How are video paths and output paths passed between containers/stages? (e.g., shared volumes, specific output conventions).
    *   Error handling for each stage: What happens if `scan_video.py` fails for a video? Does it stop, retry, or move to `corrupted/`?
3.  **Expanding on "Evita reprocessamento":**
    *   Clearly state the chosen method from `12.4` (e.g., "using a persisted state file located at `${VIDEO_STORAGE_ROOT}/orchestrator_data/processed_videos.log` which stores SHA256 hashes of processed videos").
    *   How is the "nome" ou identificador do vídeo gerado para este set (filename, hash do conteúdo)?
4.  **Detailing `ORCHESTRATOR_INTERVAL`:**
    *   Default value.
    *   Impact of this interval.
5.  **Adding more on logging and status reporting from the orchestrator.**
6.  **Considering parallel processing:** Can the orchestrator handle multiple videos in parallel if resources allow, or is it sequential?
7.  **Interaction with `corrupted/` directory:** When and how does the orchestrator decide to move a file there?
8.  **File locking/stability:** How does it ensure a video file is fully written/copied to the `input/` directory before processing starts?

Let's create a more detailed version of Section 3.

---

## 3. ORQUESTRAÇÃO (`scripts/orchestrator.py`)

O `scripts/orchestrator.py` é o componente central que gerencia o pipeline de processamento de vídeo ponta-a-ponta. Ele é responsável por detectar novos arquivos de vídeo, invocar as diferentes etapas de análise e geração de metadados em seus respectivos containers, e garantir que os vídeos não sejam reprocessados desnecessariamente.

**Modo de Execução:**
O `orchestrator.py` é tipicamente executado periodicamente via um cronjob (configurado pelo `install.sh` para rodar, por exemplo, a cada 5 minutos). A cada execução, ele escaneia por novos trabalhos.

---

### 3.1. Fluxo de Trabalho Principal

A cada execução, o `orchestrator.py` realiza as seguintes etapas:

1.  **Carregar Estado de Processamento:**
    *   Conforme detalhado na Seção `12.4`, o orquestrador carrega o conjunto de identificadores de vídeos já processados de um local persistente (e.g., `"${VIDEO_STORAGE_ROOT}/orchestrator_data/processed_videos.log"` ou um banco de dados SQLite em `"${VIDEO_STORAGE_ROOT}/orchestrator_data/orchestrator_db.sqlite"`). Este conjunto é mantido em memória durante a execução do script.

2.  **Monitorar Pasta de Entrada (`${VIDEO_STORAGE_ROOT}/input/`):**
    *   Lista todos os arquivos de vídeo (e.g., `*.mp4`, `*.avi`, configurável) presentes no diretório `input/`.
    *   **Detecção de Arquivos Estáveis:** Para evitar processar arquivos que ainda estão sendo copiados/gravados, o orquestrador pode implementar uma verificação de estabilidade:
        *   Comparar o tamanho do arquivo em duas leituras consecutivas com um pequeno intervalo (e.g., alguns segundos).
        *   Verificar a idade de modificação do arquivo (e.g., processar apenas arquivos com mais de X segundos/minutos de idade de modificação).

3.  **Identificar Novos Vídeos:**
    *   Para cada arquivo de vídeo encontrado e considerado estável:
        *   Calcula um identificador único para o vídeo. A estratégia preferencial é o **hash SHA256 do conteúdo do arquivo de vídeo**, pois é robusto contra renomeações. Alternativamente, o nome do arquivo pode ser usado se houver garantia de unicidade e imutabilidade.
        *   Verifica se este identificador já existe no conjunto de vídeos processados carregado na etapa 1.

4.  **Processar Novos Vídeos (Sequencialmente por Padrão):**
    *   Para cada vídeo identificado como novo e ainda não processado, o orquestrador executa o seguinte pipeline:

    *   **a. Etapa 1: Análise de Cena (`scan_video.py`)**
        *   **Invocação:** O orquestrador executa o script `scan_video.py` dentro do container `scene_scanner`.
            ```bash
            # Exemplo conceitual de comando
            docker-compose run --rm \
              -v "${VIDEO_STORAGE_ROOT}/input:/app/input:ro" \
              -v "${VIDEO_STORAGE_ROOT}/analytic:/app/analytic:rw" \
              scene_scanner python scan_video.py /app/input/video_novo.mp4 /app/analytic/video_novo.analytic.json
            ```
        *   **Entrada:** Caminho para o arquivo de vídeo original.
        *   **Saída Esperada:** Um arquivo JSON (`<video_nome>.analytic.json`) no diretório `${VIDEO_STORAGE_ROOT}/analytic/`, contendo os resultados da detecção de objetos, faces, etc.
        *   **Tratamento de Erro:** Se `scan_video.py` falhar (e.g., retornar um código de saída não zero, ou não produzir o arquivo de saída esperado):
            *   O orquestrador loga o erro detalhadamente.
            *   O vídeo problemático é movido para `${VIDEO_STORAGE_ROOT}/corrupted/` com um arquivo `.meta.json` associado descrevendo a falha (ver Seção `12.9`).
            *   O processamento para este vídeo é interrompido.

    *   **b. Etapa 2: Sumarização por IA (`summarize_scene.py`)**
        *   **Invocação:** Executado somente se a Etapa 1 for bem-sucedida. O orquestrador executa `summarize_scene.py` dentro do container `scene_summarizer`.
            ```bash
            # Exemplo conceitual de comando
            docker-compose run --rm \
              -v "${VIDEO_STORAGE_ROOT}/analytic:/app/analytic:ro" \
              -v "${VIDEO_STORAGE_ROOT}/summaries:/app/summaries:rw" \
              -v "${VIDEO_STORAGE_ROOT}/models:/app/models:ro" # Acesso ao modelo BitNet
              scene_summarizer python summarize_scene.py /app/analytic/video_novo.analytic.json /app/summaries/video_novo.summary.json
            ```
        *   **Entrada:** Caminho para o arquivo `<video_nome>.analytic.json` gerado na etapa anterior.
        *   **Saída Esperada:** Um arquivo JSON (`<video_nome>.summary.json`) no diretório `${VIDEO_STORAGE_ROOT}/summaries/`.
        *   **Tratamento de Erro:** Similar à Etapa 1. Se falhar, loga, move o vídeo original para `corrupted/` (ou apenas os metadados, se o vídeo já foi movido), e interrompe o processamento para este vídeo.

    *   **c. Etapa 3: Geração de LogLine (`logline_generator.py`)**
        *   **Invocação:** Executado somente se a Etapa 2 for bem-sucedida. O orquestrador executa `logline_generator.py` dentro do container `logline_writer`.
            ```bash
            # Exemplo conceitual de comando
            docker-compose run --rm \
              -v "${VIDEO_STORAGE_ROOT}/summaries:/app/summaries:ro" \
              -v "${VIDEO_STORAGE_ROOT}/loglines:/app/loglines:rw" \
              logline_writer python logline_generator.py /app/summaries/video_novo.summary.json /app/loglines/video_novo.logline.json
            ```
        *   **Entrada:** Caminho para o arquivo `<video_nome>.summary.json`.
        *   **Saída Esperada:** Um arquivo JSON (`<video_nome>.logline.json`) e um arquivo Markdown (`<video_nome>.logline.md`) no diretório `${VIDEO_STORAGE_ROOT}/loglines/`.
        *   **Tratamento de Erro:** Similar às etapas anteriores.

5.  **Atualizar Estado de Processamento:**
    *   Se todas as etapas do pipeline (a, b, c) forem concluídas com sucesso para um vídeo:
        *   O identificador único do vídeo (e.g., hash SHA256) é adicionado ao set em memória de vídeos processados.
        *   Este novo identificador é anexado ao arquivo de estado persistente (e.g., `processed_videos.log`) ou inserido no banco de dados SQLite.
        *   Opcional: O vídeo original pode ser movido de `${VIDEO_STORAGE_ROOT}/input/` para `${VIDEO_STORAGE_ROOT}/processed/` para indicar visualmente que foi concluído e para limpar a pasta de entrada.

6.  **Logging:**
    *   O orquestrador loga suas ações principais em `logs/app.log` (ou `logs/orchestrator.log`):
        *   Início e fim de cada ciclo de verificação.
        *   Número de novos vídeos encontrados.
        *   Início e fim do processamento de cada vídeo.
        *   Sucesso ou falha de cada etapa do pipeline para cada vídeo.
        *   Quaisquer erros encontrados.
        *   Vídeos movidos para `corrupted/`.

### 3.2. Configuração

*   **`ORCHESTRATOR_INTERVAL`:** Variável de ambiente que define a frequência (em segundos) com que o cronjob executa o `orchestrator.py`. Um valor padrão (e.g., 300 segundos = 5 minutos) é usado se não definido.
*   **Padrão de Arquivos de Vídeo:** O padrão de arquivos a serem procurados (e.g., `*.mp4`) pode ser configurável via variável de ambiente.
*   **Caminhos de Diretório:** Os caminhos para `input`, `analytic`, `summaries`, `loglines`, `corrupted`, `processed`, e `orchestrator_data` são derivados de `VIDEO_STORAGE_ROOT`.

### 3.3. Considerações de Performance e Robustez

*   **Processamento Sequencial vs. Paralelo:**
    *   Por padrão, o orquestrador processa um vídeo por vez para simplificar e evitar sobrecarga de recursos em dispositivos de borda.
    *   Para sistemas com mais recursos, o `orchestrator.py` poderia ser estendido para processar múltiplos vídeos em paralelo (e.g., usando `multiprocessing` ou submetendo múltiplos `docker-compose run` jobs, até um limite configurável). Isso exigiria um gerenciamento de estado mais complexo.
*   **Comunicação com Containers:** A utilização de `docker-compose run --rm` para cada etapa é robusta, pois cada tarefa é executada em um container limpo. Volumes Docker são usados para passar dados entre as etapas.
*   **Idempotência das Etapas:** Embora o orquestrador evite reprocessar vídeos inteiros, as etapas individuais (`scan_video.py`, etc.) também devem ser projetadas para serem idempotentes se possível (e.g., se um arquivo de saída já existe e é válido, não o recriar). Isso pode ajudar na recuperação de falhas parciais.
*   **Limpeza de Recursos:** O uso de `--rm` em `docker-compose run` garante que os containers sejam removidos após a conclusão da tarefa, evitando o acúmulo de containers parados.

### 3.4. Interação com Outros Componentes

*   **`self_monitor.py`:** Se um dos containers do pipeline (`scene_scanner`, `scene_summarizer`, `logline_writer`) falhar consistentemente ao ser invocado pelo orquestrador, o `self_monitor.py` pode detectar isso se essas falhas fizerem o container principal do serviço (se houver um de longa duração) ficar "unhealthy" ou se o `docker-compose run` falhar de forma que afete o status geral do serviço Docker.
*   **`cleanup_old_files.py`:** O orquestrador não lida diretamente com a limpeza de arquivos antigos; isso é delegado ao `cleanup_old_files.py`.

---

Esta versão expandida da Seção 3 detalha melhor o funcionamento interno do `orchestrator.py`, seu modo de execução, tratamento de erros, e como ele interage com as definições de persistência de estado da Seção `12.4` e as estratégias de tratamento de erro da Seção `12.9`.
