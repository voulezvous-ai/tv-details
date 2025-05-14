Okay, let's proceed to **Section 6: SCRIPTS AUXILIARES**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 6. SCRIPTS AUXILIARES

*   **orchestrator.py**: orquestra todo o pipeline.
*   **self\_monitor.py**: self-healing, reinicia contêineres com erros.
*   **logline\_verifier.py**: verifica integridade de LogLines via HMAC.
*   **log\_exporter.py**: empacota logs em tar.gz e gera manifesto JSON.
*   **cleanup\_old\_files.py**: limpa arquivos antigos, suportando `--days`, `--dirs`, `--dry-run`.
```

This section lists key utility scripts. Section 3 already detailed `orchestrator.py`. We can enrich this by:

1.  **For each remaining script (`self_monitor.py`, `logline_verifier.py`, `log_exporter.py`, `cleanup_old_files.py`):**
    *   **Responsabilidade Primária:** Clarify its main goal.
    *   **Modo de Execução:** How is it typically run (cron, manually, as a service)?
    *   **Inputs/Arguments:** Key command-line arguments or configuration it uses.
    *   **Core Logic:** Briefly describe its operational steps.
    *   **Outputs/Efeitos:** What does it produce or what system state does it change?
    *   **Error Handling & Logging:** How it reports issues.
    *   **Idempotency:** Is the script idempotent?
    *   **Interactions with `12.x` sections:** (e.g., `cleanup_old_files.py` with `12.3`, `self_monitor.py` with `12.2` and `12.9`).

Let's create a more detailed version of Section 6, skipping `orchestrator.py` as it's covered in Section 3.

---

## 6. SCRIPTS AUXILIARES E DE MANUTENÇÃO

Além dos módulos principais do pipeline de processamento, o `VV-Video-AI-System` inclui diversos scripts auxiliares que desempenham funções críticas de orquestração, monitoramento, auto-recuperação, verificação de integridade e manutenção.

---

### (6.0. `orchestrator.py` - Detalhado na Seção 3)

*   **Sumário:** Orquestra o pipeline de processamento de vídeo ponta-a-ponta, desde a detecção de novos arquivos até a geração de LogLines.

---

### 6.1. `self_monitor.py`

*   **Responsabilidade Primária:** Monitorar a saúde dos containers Docker definidos no `docker-compose.yml` e executar ações de auto-recuperação (self-healing) básicas, como o reinício de containers que falharam ou estão em estado "unhealthy".
*   **Modo de Execução:**
    *   Pode ser executado como um cronjob (e.g., a cada 1-5 minutos).
    *   Alternativamente, pode ser um serviço de longa duração que monitora eventos Docker ou periodicamente verifica o estado dos containers.
*   **Inputs/Arguments (Configuração via variáveis de ambiente ou args):**
    *   `CONTAINER_MAX_RESTART_ATTEMPTS`: Número máximo de tentativas de reinício para um container antes de desistir e/ou escalar um alerta mais grave (e.g., 5).
    *   `HEALTHCHECK_FAILURE_THRESHOLD`: Número de falhas de healthcheck consecutivas antes de considerar um container problemático.
    *   (Opcional) Lista de containers a serem monitorados explicitamente.
*   **Core Logic:**
    1.  Conecta-se ao Docker daemon (via socket Docker).
    2.  Lista os containers gerenciados pelo `docker-compose.yml` do projeto.
    3.  Para cada container:
        *   Verifica seu estado (`running`, `exited`, `restarting`).
        *   Verifica seu status de healthcheck (`healthy`, `unhealthy`, `starting`).
    4.  **Ações de Recuperação:**
        *   Se um container estiver parado (`exited` com erro) e sua política de reinício não o recuperou, ou se estiver `unhealthy` por um tempo/tentativas configuráveis, `self_monitor.py` tenta reiniciá-lo (e.g., `docker restart <container_name>`).
        *   Mantém um registro (em memória ou em um arquivo de estado simples) do número de reinícios tentados para cada container.
        *   Se o `CONTAINER_MAX_RESTART_ATTEMPTS` for atingido para um container, o script para de tentar reiniciá-lo e envia um alerta `CRITICAL` (conforme Seção `12.9`).
*   **Outputs/Efeitos:**
    *   Reinício de containers Docker.
    *   Logs de suas ações em `logs/app.log` ou `logs/self_monitor.log`.
    *   Geração de alertas para o sistema de notificação (Seção `12.9`).
*   **Error Handling & Logging:**
    *   Loga todas as verificações e ações.
    *   Trata erros de comunicação com o Docker API.
*   **Idempotency:** As verificações são baseadas no estado atual, e as ações de reinício são geralmente idempotentes.
*   **Interações com `12.x`:**
    *   **`12.2 (Self-Monitor LLM)`:** O `self_monitor.py` representa a primeira camada de auto-recuperação. O LLM monitor seria uma camada mais avançada para diagnóstico e remediação.
    *   **`12.9 (Alertas)`:** É um gerador chave de alertas sobre a saúde dos containers.

---

### 6.2. `logline_verifier.py`

*   **Responsabilidade Primária:** Fornecer uma ferramenta de linha de comando (CLI) para verificar a integridade e autenticidade de um ou mais arquivos `.logline.json` através da validação de sua assinatura HMAC.
*   **Modo de Execução:** Manualmente, via CLI, por um operador ou script de auditoria.
*   **Inputs/Arguments:**
    *   Caminho para um arquivo `.logline.json` específico ou um diretório contendo múltiplos arquivos LogLine.
    *   `--key-file` (ou similar): Caminho para um arquivo contendo o `HMAC_SECRET_KEY` e `KEY_ID` (ou busca em variáveis de ambiente, conforme `vv-core/utils.py load_key_store()`).
*   **Core Logic:**
    1.  Lê o arquivo `.logline.json`.
    2.  Extrai o payload original (todos os campos exceto `signature_details`) e a assinatura armazenada (`signature_details.signature`, `signature_details.key_id`).
    3.  Recupera a chave HMAC secreta correspondente ao `key_id` da LogLine.
    4.  Recalcula a assinatura HMAC do payload original usando a chave recuperada.
    5.  Compara a assinatura recalculada com a assinatura armazenada na LogLine.
    6.  Verifica se o `key_id` na LogLine é um `key_id` conhecido/confiável.
*   **Outputs/Efeitos:**
    *   Saída no console indicando se a assinatura de cada LogLine verificada é VÁLIDA ou INVÁLIDA.
    *   Em caso de falha, pode fornecer detalhes sobre a discrepância.
    *   Pode ter uma opção para output em formato JSON para fácil parsing por outros scripts.
*   **Error Handling & Logging:**
    *   Reporta erros como arquivo LogLine malformado, `key_id` desconhecido, ou chave HMAC não encontrada.
    *   Loga suas operações se uma opção de log for fornecida.
*   **Idempotency:** É uma operação de leitura e verificação, portanto, idempotente.
*   **Interações com `12.x`:**
    *   **`12.7 (Gestão de Segredos)`:** Depende de um método seguro para acessar as chaves HMAC.

---

### 6.3. `log_exporter.py`

*   **Responsabilidade Primária:** Coletar, empacotar (em formato `tar.gz`) e gerar um manifesto para os logs da aplicação e, opcionalmente, logs de containers, para fins de arquivamento, auditoria ou análise off-site.
*   **Modo de Execução:**
    *   Tipicamente executado como um cronjob diário (configurado pelo `install.sh`).
    *   Pode também ser executado manualmente.
*   **Inputs/Arguments (Configuração via variáveis de ambiente ou args):**
    *   `LOG_SOURCE_DIRECTORIES`: Lista de diretórios a serem incluídos no backup (e.g., `/opt/vv-video-ai-system/logs/`).
    *   `LOG_EXPORT_DESTINATION`: Diretório onde os arquivos `tar.gz` e manifestos serão salvos.
    *   `RETENTION_DAYS_EXPORTED_LOGS`: (Opcional) Por quantos dias manter os arquivos exportados no destino antes de limpá-los (ou isso é gerenciado por outra ferramenta).
    *   (Opcional) Opção para incluir logs de containers Docker específicos.
*   **Core Logic:**
    1.  Identifica os arquivos de log a serem exportados com base na data (e.g., logs do dia anterior). Inclui logs rotacionados e comprimidos (e.g., `app.log.1.gz`).
    2.  Cria um arquivo `tar.gz` contendo os arquivos de log selecionados. O nome do arquivo `tar.gz` deve incluir um timestamp (e.g., `vv_ai_logs_YYYYMMDD.tar.gz`).
    3.  Gera um arquivo de manifesto JSON correspondente (e.g., `vv_ai_logs_YYYYMMDD.manifest.json`). Este manifesto contém:
        *   Nome do arquivo `tar.gz`.
        *   Timestamp da exportação.
        *   Lista de arquivos incluídos no tarball.
        *   Hashes (e.g., SHA256) de cada arquivo individual e do tarball completo para verificação de integridade.
    4.  Salva o `tar.gz` e o manifesto no `LOG_EXPORT_DESTINATION`.
    5.  (Opcional) Se `RETENTION_DAYS_EXPORTED_LOGS` estiver configurado, remove arquivos de exportação mais antigos que o período de retenção do `LOG_EXPORT_DESTINATION`.
*   **Outputs/Efeitos:**
    *   Criação de arquivos `tar.gz` e `.manifest.json` no diretório de destino.
    *   Logs de sua operação.
*   **Error Handling & Logging:**
    *   Loga o processo de exportação, incluindo quaisquer arquivos que não puderam ser lidos ou adicionados.
    *   Reporta erros se o diretório de destino não for gravável.
*   **Idempotency:** Se executado múltiplas vezes para o mesmo período, pode recriar o mesmo arquivo de exportação, o que é aceitável. A lógica de nomeação baseada em data ajuda a evitar conflitos.

---

### 6.4. `cleanup_old_files.py`

*   **Responsabilidade Primária:** Gerenciar o espaço em disco removendo arquivos antigos de diretórios especificados, com base em critérios de idade e outras opções.
*   **Modo de Execução:**
    *   Executado como um cronjob (ver Seção `12.3` para configuração efetiva).
    *   Pode ser executado manualmente para limpezas ad-hoc ou testes.
*   **Inputs/Arguments (CLI):**
    *   `--days N`: Especifica a idade mínima (em dias) dos arquivos a serem considerados para exclusão (baseado no tempo de modificação).
    *   `--dirs /path/to/dir1,/path/to/dir2,...`: Lista de diretórios onde a limpeza será aplicada.
    *   `--dry-run`: Simula a execução, listando os arquivos que seriam excluídos, mas sem excluí-los. Essencial para testes.
    *   `--log-file /path/to/logfile.log`: (Recomendado) Especifica um arquivo para registrar as ações de limpeza detalhadas.
    *   `--min-free-space GB_OR_%`: (Avançado) Limpeza condicional baseada no espaço livre.
    *   `--exclude-patterns "*.important,temp_*,*.lock"`: Padrões de arquivos a serem excluídos da limpeza.
*   **Core Logic:**
    1.  Parseia os argumentos da linha de comando.
    2.  Para cada diretório especificado em `--dirs`:
        *   Percorre recursivamente os arquivos.
        *   Para cada arquivo, verifica sua idade (tempo de modificação).
        *   Verifica se corresponde a algum `--exclude-patterns`.
        *   Se o arquivo atender aos critérios de idade e não for excluído por padrão:
            *   Se `--dry-run` estiver ativo, loga que o arquivo *seria* excluído.
            *   Se `--dry-run` não estiver ativo, exclui o arquivo e loga a ação.
    3.  Se `--log-file` for especificado, todas as ações (ou simulações) são registradas nele.
    4.  Se `--min-free-space` for usado, a lógica de exclusão só é ativada se o espaço livre estiver abaixo do limite.
*   **Outputs/Efeitos:**
    *   Exclusão de arquivos dos diretórios especificados (se não em `--dry-run`).
    *   Saída no console (resumo) e/ou arquivo de log detalhado.
*   **Error Handling & Logging:**
    *   Loga erros (e.g., falha ao excluir um arquivo devido a permissões).
    *   Trata caminhos de diretório inválidos.
*   **Idempotency:** Sim, executar novamente com os mesmos parâmetros não causa problemas; apenas arquivos que ainda atendem aos critérios seriam (re)processados (ou listados em dry-run).
*   **Interações com `12.x`:**
    *   **`12.3 (Configuração Efetiva)`:** Esta seção detalha como este script é configurado para limpeza real em produção.

---

Esta versão expandida da Seção 6 fornece uma compreensão mais clara das responsabilidades e do funcionamento de cada script auxiliar, como eles são executados e como interagem com outras partes do sistema e as melhorias propostas.
