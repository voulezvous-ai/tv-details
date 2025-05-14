Alright, let's move on to **Section 2: BOOTSTRAP (`bootstrap.py`)**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 2. BOOTSTRAP (`bootstrap.py`)

1.  Criação de diretórios em `${VIDEO_STORAGE_ROOT}`:
    `input`, `raw`, `processed`, `tv_curado`, `analytic`, `summaries`, `loglines`, `playlists`, `fallback`, `corrupted`.
2.  Clonagem do modelo BitNet local (quantização 4-bit) se ausente (Ver Seção 12.6 para gerenciamento do modelo).
3.  Cópia de assets de **fallback**.
4.  Geração de `.env` a partir de `.env.example` (Ver Seção 12.7 para gestão de segredos).
```

This section describes the initial setup script `bootstrap.py`. We can enrich it by:

1.  **Elaborating on the `--skip-existing` flag mentioned in Section 1.**
2.  **Detailing the directory creation process:**
    *   Permissions for these directories.
    *   Handling of `VIDEO_STORAGE_ROOT` if not defined or accessible.
3.  **Expanding on "Clonagem do modelo BitNet":**
    *   Reference the `model_manifest.json` approach from `12.6` more explicitly if that's the chosen strategy.
    *   Specify the target location for the model.
    *   Error handling if the model download/clone fails.
4.  **Detailing "Cópia de assets de fallback":**
    *   Source and destination of these assets.
    *   What happens if source assets are missing?
5.  **Explaining "Geração de `.env`":**
    *   How it handles existing `.env` files (respects, overwrites, merges?).
    *   Prompting for essential undefined variables (like `DVR_IP`) if not set by `install.sh` or a previous run.
    *   Security of the generated `.env` file (permissions).
6.  **Adding steps for other initial setup tasks `bootstrap.py` might handle:**
    *   Generating `vv-signature.json` if it doesn't exist or needs updating based on core file changes.
    *   Any other one-time initializations (e.g., initializing a SQLite DB for the orchestrator state if that's the chosen method in `12.4`).
7.  **Logging within `bootstrap.py`:** For traceability of its actions.
8.  **Idempotency:** Reinforce how `bootstrap.py` ensures it can be run multiple times safely.

Let's create a more detailed version.

---

## 2. BOOTSTRAP (`bootstrap.py`)

O script `bootstrap.py` é um componente crucial executado pelo `install.sh` (e pode ser invocado manualmente se necessário) para preparar o ambiente e os assets iniciais do `VV-Video-AI-System`. Ele é projetado para ser idempotente, utilizando a flag `--skip-existing` para evitar a recriação ou sobrescrita desnecessária de arquivos e configurações já existentes.

**Execução Típica:** `python3 bootstrap.py [--skip-existing]`

---

### 2.1. Funcionalidades e Ordem de Execução

O `bootstrap.py` executa as seguintes tarefas na ordem especificada:

1.  **Leitura de Configuração e Variáveis de Ambiente:**
    *   Carrega variáveis de ambiente existentes (e.g., de um `.env` já presente ou do sistema) para obter configurações como `VIDEO_STORAGE_ROOT`.
    *   Define caminhos padrão se variáveis essenciais não estiverem definidas, emitindo avisos.

2.  **Criação da Estrutura de Diretórios em `${VIDEO_STORAGE_ROOT}`:**
    *   **Variável `VIDEO_STORAGE_ROOT`:** O script verifica se `VIDEO_STORAGE_ROOT` está definida. Se não, pode usar um caminho padrão (e.g., `/opt/vv-video-ai-system/storage`) e alertar o usuário. Deve verificar se o caminho base é acessível e tem permissões de escrita.
    *   **Diretórios Criados:**
        *   `input/`: Para vídeos novos aguardando processamento.
        *   `raw/`: (Opcional, se a política for manter cópias brutas) Para cópias originais dos vídeos.
        *   `processed/`: Para vídeos que concluíram o pipeline.
        *   `tv_curado/`: Para vídeos e playlists curados para exibição.
        *   `analytic/`: Para JSONs de análise de `scan_video.py`.
        *   `summaries/`: Para JSONs de sumários de `summarize_scene.py`.
        *   `loglines/`: Para arquivos LogLine (`.json` e `.md`).
        *   `playlists/`: Para `playlist.json` gerados por `tv_scheduler.py`.
        *   `fallback/`: Para assets de fallback.
        *   `corrupted/`: Para vídeos ou dados que falharam no processamento.
        *   `orchestrator_data/`: Para armazenar o estado do `orchestrator.py` (ver Seção `12.4`).
        *   `models/`: Diretório base para modelos de IA, incluindo o BitNet.
    *   **Idempotência (`--skip-existing`):** Se a flag `--skip-existing` for passada, o script verifica a existência de cada diretório antes de tentar criá-lo (`os.makedirs(path, exist_ok=True)` é intrinsecamente idempotente).
    *   **Permissões:** Os diretórios são criados com permissões que permitem ao usuário da aplicação (e.g., `app_user` definido na Seção 1) ler e escrever neles. Por exemplo, `0755` ou `0775` dependendo da política de grupo.

3.  **Configuração e Download do Modelo BitNet (Conforme Seção `12.6`):**
    *   **Localização:** O modelo será instalado em um subdiretório de `models/` (e.g., `models/bitnet_summarizer/`).
    *   **Mecanismo:**
        1.  Lê o `model_manifest.json` (localizado no repositório, e.g., `/opt/vv-video-ai-system/model_manifest.json`).
        2.  Verifica a versão do modelo atualmente instalada (e.g., lendo um arquivo `.current_model_info.json` em `models/`).
        3.  Se o modelo estiver ausente, desatualizado ou o checksum não corresponder ao do manifesto, o `bootstrap.py` baixa o artefato do modelo (e.g., `.tar.gz`) da URL especificada no manifesto.
        4.  Valida o checksum SHA256 do arquivo baixado.
        5.  Extrai e instala o modelo no diretório de destino.
        6.  Atualiza o arquivo `.current_model_info.json` local com a versão e checksum do modelo instalado.
    *   **Idempotência (`--skip-existing`):** Se a flag estiver ativa e uma versão correta do modelo já estiver presente e validada, esta etapa é pulada.
    *   **Tratamento de Erros:** Se o download falhar, o checksum não corresponder, ou a extração falhar, o script loga o erro e pode abortar ou continuar com um aviso, dependendo da criticidade (um sistema sem modelo de sumarização é severamente degradado).

4.  **Cópia de Assets de Fallback:**
    *   **Fonte:** Arquivos localizados no repositório, e.g., `fallback/loop_silencioso.mp4` e `fallback/placeholder.png`.
    *   **Destino:** `${VIDEO_STORAGE_ROOT}/fallback/`.
    *   **Idempotência (`--skip-existing`):** Se a flag estiver ativa, os arquivos só são copiados se não existirem no destino ou se os arquivos de origem forem mais recentes (verificação por timestamp ou checksum).
    *   **Tratamento de Erros:** Se os assets de fallback de origem estiverem faltando no repositório, loga um aviso.

5.  **Geração e Gerenciamento do Arquivo `.env` (Conforme Seção `12.7`):**
    *   **Arquivo de Exemplo:** Utiliza `.env.example` como template.
    *   **Criação/Atualização:**
        *   Se `.env` não existir, ele é criado como uma cópia de `.env.example`.
        *   Se `.env` existir e `--skip-existing` for fornecido, o arquivo `.env` existente é preservado. O script pode, no entanto, verificar se todas as chaves presentes no `.env.example` também existem no `.env` atual, adicionando as que faltam com seus valores padrão do exemplo e emitindo um aviso.
        *   **Sem `--skip-existing` (ou com uma flag `--update-env`):** O script poderia mesclar o `.env.example` com o `.env` existente, preservando os valores definidos pelo usuário no `.env` e adicionando quaisquer novas variáveis do `.env.example`. Sobrescrever valores definidos pelo usuário deve ser evitado a menos que explicitamente instruído.
    *   **Valores Padrão e Interatividade (Opcional):** Para variáveis críticas sem valor padrão no `.env.example` (e.g., `DVR_IP`), o script pode verificar se elas estão definidas no `.env` resultante. Se não, e se o script estiver rodando em modo interativo, poderia solicitar ao usuário que forneça esses valores. Em execuções não interativas (como pelo `install.sh`), apenas um aviso seria emitido.
    *   **Permissões:** Após criar ou modificar o `.env`, o script garante que ele tenha permissões restritas (e.g., `chmod 600 .env`) e seja propriedade do usuário da aplicação.

6.  **Inicialização de Outros Componentes (Conforme Necessário):**
    *   **Estado do Orquestrador (Seção `12.4`):** Se o orquestrador usar um banco de dados SQLite, o `bootstrap.py` pode ser responsável por criar o arquivo de banco de dados inicial (e.g., `orchestrator_data/orchestrator_db.sqlite`) e aplicar quaisquer schemas iniciais, se o arquivo não existir. Se for um arquivo de log simples, apenas garantir que o diretório `orchestrator_data/` exista pode ser suficiente.
    *   **Geração do `vv-signature.json`:** (Opcional) Se o `vv-signature.json` for um manifesto de integridade dos *arquivos do próprio sistema* (e não de dados gerados), o `bootstrap.py` poderia ter uma função para (re)gerar este arquivo, calculando hashes dos scripts e arquivos de configuração chave. Isso seria útil para verificar a integridade da instalação.

### 2.2. Flag `--skip-existing`

*   Esta flag é fundamental para a idempotência do `bootstrap.py`.
*   Quando presente, instrui o script a evitar ações que modificariam arquivos ou configurações já existentes e presumivelmente corretos.
    *   Não recria diretórios que já existem.
    *   Não baixa/reinstala o modelo BitNet se a versão correta já estiver presente.
    *   Não sobrescreve assets de fallback se já existirem no destino.
    *   Não sobrescreve um arquivo `.env` existente (mas pode adicionar chaves faltantes do `.env.example`).

### 2.3. Logging

*   O `bootstrap.py` deve logar suas ações de forma clara, indicando o que está fazendo, o que foi pulado devido a `--skip-existing`, e quaisquer avisos ou erros encontrados.
*   Os logs podem ir para `stdout` e também para o `logs/app.log` ou um log específico de bootstrap (e.g., `logs/bootstrap.log`).

### 2.4. Tratamento de Erros

*   O script deve tratar erros graciosamente.
*   Para erros críticos (e.g., incapacidade de criar `VIDEO_STORAGE_ROOT`, falha no download de um modelo essencial na primeira vez), o script pode sair com um código de erro diferente de zero, sinalizando falha para o `install.sh`.
*   Para erros menos críticos (e.g., asset de fallback opcional ausente), pode logar um aviso e continuar.

---

Esta versão expandida da Seção 2 oferece uma visão mais detalhada das responsabilidades, lógica e comportamento do `bootstrap.py`, especialmente em relação à idempotência e interações com outras partes do sistema como o gerenciamento de modelos e configurações.
