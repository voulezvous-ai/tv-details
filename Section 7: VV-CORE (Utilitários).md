Okay, let's proceed to **Section 7: VV-CORE (Utilitários)**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 7. VV-CORE (Utilitários)

*   `compute_sha256(path: Path) -> str`
*   `load_schema(path: Path) -> dict`
*   `validate_schema(data: dict, schema_path: Path) -> None`
*   `sign_payload(payload: bytes, secret: bytes) -> str`
*   `verify_signature(payload: bytes, secret: bytes, signature: str) -> bool`
*   `load_key_store(path: Path) -> dict` (Ver Seção 12.7 para gestão de segredos)
```

This section lists utility functions intended to be shared across different modules of the `VV-Video-AI-System`. We can enrich this by:

1.  **Describing the purpose and benefits of having a `vv-core` module.**
2.  **For each function listed:**
    *   Briefly explain its purpose.
    *   Mention any key libraries it might use (e.g., `hashlib`, `jsonschema`, `hmac`).
    *   Clarify any important implementation details or assumptions.
    *   For `validate_schema`, specify what happens on validation failure (e.g., raises an exception).
    *   For `load_key_store`, reiterate its connection to secure secret management from `12.7`.
3.  **Considering other potential utility functions** that might be useful in `vv-core`, based on the needs of other modules described. Examples:
    *   Standardized logging setup function.
    *   Functions for atomic file writes.
    *   Date/time manipulation utilities (consistent UTC Z formatting).
    *   A simple alerter function (as discussed in `12.9`).
    *   Path manipulation utilities, especially around `VIDEO_STORAGE_ROOT`.
4.  **Discussing how these utilities are imported and used by other services.**
5.  **Mentioning testing estratégia for `vv-core`.**

Let's create a more detailed version of Section 7.

---

## 7. `vv-core/` (Módulo de Utilitários Compartilhados)

O diretório `vv-core/` é um módulo Python projetado para abrigar funcionalidades de utilidade comum, compartilhadas por diversos scripts e serviços dentro do `VV-Video-AI-System`. O objetivo é promover a reutilização de código, garantir consistência em operações críticas (como hashing, validação e assinatura) e simplificar a manutenção.

**Estrutura:**

```
vv-core/
├── __init__.py
└── utils.py      # Arquivo principal contendo as funções utilitárias
└── alerter.py    # (Opcional, se centralizar alertas aqui - ver 12.9)
└── config_loader.py # (Opcional, para carregar configurações de forma padronizada)
```

**Uso:**
Os scripts em outros diretórios (e.g., `scan_video/scan_video.py`, `scripts/orchestrator.py`) importam as funções necessárias de `vv-core.utils` ou outros submódulos de `vv-core`. O `PYTHONPATH` deve ser configurado adequadamente nos Dockerfiles ou no ambiente de execução para permitir esses imports.

---

### 7.1. Funções Utilitárias Principais (`vv-core/utils.py`)

A seguir, uma descrição das funções utilitárias chave e suas responsabilidades:

*   **`compute_sha256(file_path: Path) -> str`**
    *   **Propósito:** Calcula e retorna o hash SHA256 do conteúdo de um arquivo especificado. Usado para gerar identificadores únicos de arquivos (e.g., vídeos) e para verificações de integridade.
    *   **Implementação:** Lê o arquivo em blocos para lidar eficientemente com arquivos grandes.
    *   **Biblioteca:** `hashlib`.
    *   **Retorno:** String hexadecimal representando o hash SHA256.
    *   **Erro:** Levanta `FileNotFoundError` se o arquivo não existir, ou `IOError` para outros problemas de leitura.

*   **`load_schema(schema_path: Path) -> dict`**
    *   **Propósito:** Carrega um arquivo de schema JSON (e.g., de `schemas/analytic.schema.json`) do disco e o parseia em um dicionário Python.
    *   **Implementação:** Simplesmente abre, lê e parseia o JSON. Pode incluir caching em memória se os schemas forem lidos frequentemente e não mudarem.
    *   **Biblioteca:** `json`.
    *   **Retorno:** Dicionário Python representando o schema.
    *   **Erro:** Levanta `FileNotFoundError` ou `JSONDecodeError`.

*   **`validate_schema(data: dict, schema_path: Path_or_schema_dict: Union[Path, dict]) -> None`**
    *   **Propósito:** Valida um dicionário de dados (`data`) contra um schema JSON especificado (fornecido como caminho para o arquivo de schema ou como um dicionário de schema já carregado).
    *   **Implementação:** Utiliza a biblioteca `jsonschema` para realizar a validação.
    *   **Biblioteca:** `jsonschema`.
    *   **Retorno:** `None` se a validação for bem-sucedida.
    *   **Erro:** Levanta uma exceção específica (e.g., `jsonschema.exceptions.ValidationError` ou uma exceção customizada `SchemaValidationError`) se a validação falhar, contendo detalhes sobre a falha.

*   **`sign_payload(payload: bytes, secret_key: bytes, algorithm: str = "sha256") -> str`**
    *   **Propósito:** Gera uma assinatura HMAC (Hash-based Message Authentication Code) para um payload de bytes usando uma chave secreta especificada. Usado primariamente para assinar LogLines.
    *   **Implementação:** Utiliza o algoritmo HMAC com o hash especificado (padrão SHA256).
    *   **Biblioteca:** `hmac`, `hashlib`.
    *   **Argumentos:**
        *   `payload`: Os dados a serem assinados.
        *   `secret_key`: A chave secreta para a geração do HMAC.
        *   `algorithm`: O algoritmo de hash a ser usado (e.g., 'sha256', 'sha512').
    *   **Retorno:** String hexadecimal representando a assinatura HMAC.

*   **`verify_signature(payload: bytes, secret_key: bytes, provided_signature: str, algorithm: str = "sha256") -> bool`**
    *   **Propósito:** Verifica se uma assinatura HMAC fornecida é válida para um determinado payload e chave secreta. Usado por `scripts/logline_verifier.py`.
    *   **Implementação:** Recalcula a assinatura HMAC do payload e a compara de forma segura (usando `hmac.compare_digest` para evitar ataques de timing) com a assinatura fornecida.
    *   **Biblioteca:** `hmac`, `hashlib`.
    *   **Retorno:** `True` se a assinatura for válida, `False` caso contrário.

*   **`load_key_store(key_store_path: Path, key_id: str) -> Optional[str]` (Conforme Seção `12.7`)**
    *   **Propósito:** Carregar uma chave secreta específica (identificada por `key_id`) de um "key store" (que pode ser um arquivo JSON, YAML, ou interagir com um sistema de gerenciamento de segredos). Esta função abstrai a forma como as chaves HMAC (e potencialmente outras) são armazenadas e recuperadas.
    *   **Implementação:** A implementação exata dependerá da estratégia de gestão de segredos adotada (Seção `12.7`).
        *   **Simples (arquivo JSON/YAML):** Lê um arquivo local (protegido por permissões de sistema) que mapeia `key_id` para chaves secretas.
        *   **Avançado:** Poderia interagir com Docker Secrets, HashiCorp Vault, ou variáveis de ambiente de forma padronizada.
    *   **Retorno:** A chave secreta como string, ou `None` se o `key_id` não for encontrado ou o key store não puder ser acessado. Deve logar erros de forma segura sem expor informações sensíveis.

---

### 7.2. Potenciais Funções Utilitárias Adicionais

Com base nas necessidades do sistema, as seguintes funções também poderiam ser incluídas em `vv-core/utils.py` ou em submódulos:

*   **`setup_logging(log_level: str = "INFO", log_file: Optional[Path] = None) -> None`**
    *   **Propósito:** Configurar o logging de forma padronizada para todos os scripts e serviços, incluindo formato, nível e destino (console e/ou arquivo).
    *   **Biblioteca:** `logging`.

*   **`atomic_write_json(data: dict, output_path: Path) -> None`**
    *   **Propósito:** Escrever dados JSON em um arquivo de forma atômica (escreve em um arquivo temporário no mesmo diretório e depois renomeia para o `output_path`). Previne arquivos parcialmente escritos em caso de falha.
    *   **Biblioteca:** `json`, `os`, `tempfile`.

*   **`format_utc_timestamp(dt_object: Optional[datetime] = None) -> str`**
    *   **Propósito:** Formatar um objeto `datetime` (ou o tempo atual se `None`) como uma string UTC no formato ISO 8601 com 'Z' (e.g., "YYYY-MM-DDTHH:MM:SS.ffffffZ").
    *   **Biblioteca:** `datetime`.

*   **`get_video_storage_path(sub_dir: str, filename: str = "") -> Path`**
    *   **Propósito:** Construir caminhos de forma consistente dentro da estrutura de `VIDEO_STORAGE_ROOT`, lendo `VIDEO_STORAGE_ROOT` de uma variável de ambiente ou configuração.
    *   **Exemplo:** `get_video_storage_path("processed", "video.mp4")` -> `/opt/vv-video-ai-system/storage/processed/video.mp4`.
    *   **Biblioteca:** `os`, `pathlib`.

*   **`send_system_alert(level: str, title: str, message_details: dict, source_component: str)` (em `vv-core/alerter.py`)**
    *   **Propósito:** Função centralizada para enviar alertas, conforme detalhado na Seção `12.9`.

---

### 7.3. Testes

*   O módulo `vv-core/` deve ter uma suíte de testes unitários abrangente (e.g., em `tests/test_vv_core_utils.py`).
*   Cada função utilitária deve ser testada com casos de sucesso e de falha esperados.
*   A cobertura de testes para `vv-core` deve ser alta, dada a sua criticidade e uso por múltiplos componentes.

---

Esta seção detalhada para `vv-core/` estabelece um conjunto robusto de utilitários compartilhados, enfatizando a importância da reutilização de código, consistência e testabilidade para as funcionalidades base do `VV-Video-AI-System`.
