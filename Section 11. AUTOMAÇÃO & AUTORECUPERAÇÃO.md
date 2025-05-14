Okay, let's move to the final section of the main blueprint before the `12.x` improvements: **Section 11: AUTOMAÇÃO & AUTORECUPERAÇÃO**.

Here's the current content from the `VV-Video-AI-System-Master-Blueprint-v2.2.md`:

```markdown
## 11. AUTOMAÇÃO & AUTORECUPERAÇÃO

*   **Watchtower**: atualizações automáticas de imagens (Ver Seção 12.5).
*   **Self-monitor LLM**: serviço de autodiagnóstico e remediação (Ver Seção 12.2).
*   **Cronjobs** para manutenção: rebuilds semanais e testes diários (`run_tests.sh`).
*   **Alertas** via webhook (Slack/Telegram) para incidentes (Ver Seção 12.9 para detalhamento).
```

This section highlights the system's capabilities for automated operation and self-healing. We've touched on these aspects in various `12.x` sections. We can enrich this by:

1.  **Expanding on `Watchtower`:**
    *   Briefly reiterate its role in keeping images (both public and custom via private registry as per `12.5`) up-to-date.
2.  **Detailing `Self-monitor LLM`:**
    *   Summarize its (optional/future) advanced capabilities for diagnosis and remediation, referencing `12.2`.
    *   Distinguish it from the basic `self_monitor.py`.
3.  **Clarifying "Cronjobs para manutenção":**
    *   The blueprint currently mentions "rebuilds semanais". This might need re-evaluation in light of the CI/CD strategy for images (`12.5`). If CI/CD handles image updates, what are these "rebuilds"? Perhaps it refers to a full system refresh or specific maintenance tasks not covered by `cleanup_old_files.py` or `log_exporter.py`.
    *   "testes diários (`run_tests.sh`)" - What kind of tests? Smoke tests on the live system? Integrity checks? How does `run_tests.sh` work?
    *   List the *other* key cronjobs already defined (`orchestrator.py`, `cleanup_old_files.py`, `log_exporter.py`, `self_monitor.py` if cron-based) as part of the automation suite.
4.  **Summarizing "Alertas via webhook":**
    *   Reiterate their role in notifying operators about critical incidents, linking to the detailed plan in `12.9`.
5.  **Adding a point about `self_monitor.py` (the basic one):** Even if the LLM monitor is future, the current `self_monitor.py` is a key part of autorecuperação.
6.  **Considering "Idempotência"** of scripts (`install.sh`, `bootstrap.py`) as a form of automation and recovery facilitation.
7.  **Mentioning "State Persistence" (`12.4`)** as crucial for recovery after restarts.

Let's create a more detailed version of Section 11.

---

## 11. AUTOMAÇÃO, AUTORECUPERAÇÃO E MANUTENÇÃO PROATIVA

O `VV-Video-AI-System` é projetado com um forte foco em operação autônoma ("hands-off"), auto-recuperação de falhas comuns e manutenção proativa para garantir disponibilidade e robustez 24/7, especialmente em dispositivos de borda.

---

### 11.1. Atualizações Automatizadas de Software

*   **`Watchtower` para Atualização de Imagens Docker (Conforme Seção `12.5`):**
    *   **Função:** O serviço `watchtower` monitora continuamente os registros Docker (públicos e o registro privado configurado para imagens customizadas do `VV-Video-AI-System`).
    *   **Operação:** Ao detectar uma nova versão de uma imagem em uso, `watchtower` automaticamente baixa a nova imagem, para o container existente e inicia um novo container com a imagem atualizada, preservando a configuração.
    *   **Benefício:** Garante que o sistema execute as versões mais recentes das imagens, incluindo patches de segurança e novas funcionalidades, com mínima intervenção manual. A integração com o pipeline de CI/CD (Seção `12.5`) assegura que apenas imagens testadas sejam implantadas.

### 11.2. Auto-Recuperação (Self-Healing)

*   **`self_monitor.py` (Monitor Básico de Saúde e Recuperação - Seção `6.1`):**
    *   **Função:** Este script monitora ativamente a saúde dos containers Docker definidos no `docker-compose.yml`.
    *   **Operação:** Verifica o estado dos containers e seus healthchecks. Se um container falhar ou se tornar "unhealthy" repetidamente, `self_monitor.py` tenta reiniciá-lo automaticamente um número configurável de vezes.
    *   **Escalonamento:** Se as tentativas de reinício falharem persistentemente, ele gera um alerta `CRITICAL` (conforme Seção `12.9`) para notificar os operadores.
    *   **Benefício:** Lida automaticamente com falhas transitórias de containers e problemas comuns, restaurando a funcionalidade do serviço sem intervenção manual.

*   **Políticas de Reinício do Docker (`restart: unless-stopped` / `on-failure` - Seção 4):**
    *   **Função:** As políticas de reinício configuradas no `docker-compose.yml` instruem o Docker daemon a reiniciar automaticamente containers que pararem inesperadamente.
    *   **Benefício:** Fornece um nível fundamental de auto-recuperação gerenciado diretamente pelo Docker.

*   **Persistência de Estado para Recuperação (Conforme Seção `12.4`):**
    *   **Função:** O `orchestrator.py` (e potencialmente outros componentes) persiste seu estado de processamento (e.g., quais vídeos já foram processados).
    *   **Benefício:** Em caso de reinício do orquestrador ou de todo o sistema, ele pode retomar o trabalho de onde parou, evitando reprocessamento desnecessário e garantindo consistência.

### 11.3. Manutenção Automatizada via Cronjobs

Diversos scripts são executados via `cron` para realizar tarefas de manutenção e operação de rotina:

*   **`scripts/orchestrator.py` (Seção 3):**
    *   Executado a cada N minutos (e.g., 5 minutos) para detectar e processar novos vídeos no pipeline.
*   **`scripts/cleanup_old_files.py` (Seção `6.4` e `12.3`):**
    *   Executado diariamente (ou com outra frequência definida) para remover arquivos antigos de dados processados, logs e outros diretórios, gerenciando o espaço em disco. A configuração inicial pode ser `--dry-run`, mas deve ser ajustada para limpeza efetiva.
*   **`scripts/log_exporter.py` (Seção `6.3`):**
    *   Executado diariamente para arquivar logs da aplicação e do sistema, facilitando a auditoria e a análise histórica.
*   **`scripts/self_monitor.py` (Seção `6.1`, se operado via cron):**
    *   Se não for um serviço de longa duração, pode ser executado via cron (e.g., a cada 1-5 minutos) para verificar a saúde dos containers.
*   **(Reavaliar) "Rebuilds Semanais":**
    *   A menção original a "rebuilds semanais" precisa ser contextualizada. Com a estratégia de CI/CD e `watchtower` (Seção `12.5`), as imagens são atualizadas continuamente.
    *   Um "rebuild semanal" pode se referir a:
        *   Uma execução forçada de `docker-compose build --pull --no-cache` para garantir que as camadas base das imagens sejam atualizadas, mesmo que os Dockerfiles não mudem. (Menos relevante se `watchtower` já atualiza para novas tags).
        *   Um script que força a recriação de todos os containers (`docker-compose up -d --force-recreate`) para limpar qualquer estado anômalo de container, embora a persistência de dados em volumes deva manter os dados importantes.
        *   Ou esta tarefa pode ser substituída/refinada pelas práticas de CI/CD.
*   **(Reavaliar) "Testes Diários (`run_tests.sh`):"**
    *   **Propósito:** Definir claramente o escopo deste script.
        *   **Smoke Tests:** Poderia executar um conjunto mínimo de testes no ambiente de produção para verificar funcionalidades críticas (e.g., capacidade de processar um arquivo de teste pequeno, API `/status` está acessível e saudável).
        *   **Verificações de Integridade:** Poderia invocar `scripts/logline_verifier.py` para um subconjunto de LogLines recentes, ou `scripts/verify_system_integrity.py` (se implementado, conforme Seção 10).
    *   O script `run_tests.sh` deve logar seus resultados e gerar alertas em caso de falha.

### 11.4. Alertas e Notificações Proativas (Conforme Seção `12.9`)

*   **Função:** O sistema é configurado para enviar alertas via webhooks (Slack, Telegram, etc.) para uma variedade de incidentes e condições anormais.
*   **Gatilhos:** Incluem falhas persistentes de containers, erros de processamento de dados, baixo espaço em disco, falhas de validação de schema/HMAC, etc.
*   **Benefício:** Permite que os operadores sejam notificados proativamente sobre problemas, muitas vezes antes que impactem os usuários finais, facilitando uma resposta rápida.

### 11.5. Idempotência de Scripts de Configuração

*   **`install.sh` (Seção 1) e `bootstrap.py` (Seção 2):** São projetados para serem idempotentes. Podem ser executados múltiplas vezes, convergindo o sistema para o estado configurado desejado sem causar efeitos colaterais negativos.
*   **Benefício:** Facilita a recuperação de configurações parciais, permite reexecuções seguras para aplicar atualizações na configuração base do sistema e simplifica a automação da implantação.

### 11.6. (Opcional/Futuro) `Self-monitor LLM` (Conforme Seção `12.2`)

*   **Função (Avançada):** Um serviço opcional que utiliza um Modelo de Linguagem Grande (LLM) para realizar diagnósticos mais profundos de problemas complexos (e.g., análise de padrões em logs, correlação de eventos) e, potencialmente, executar ações de remediação mais sofisticadas do que simples reinícios.
*   **Benefício:** Levaria a automação e a capacidade de auto-recuperação do sistema a um nível significativamente mais alto, reduzindo ainda mais a necessidade de intervenção manual.

---

Ao combinar estas várias camadas de automação e mecanismos de auto-recuperação, o `VV-Video-AI-System` visa minimizar a carga operacional, aumentar a resiliência e manter um alto nível de disponibilidade. A manutenção proativa e os alertas ajudam a identificar e resolver problemas antes que se tornem críticos.
