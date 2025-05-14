Okay, let's do a comprehensive enrichment of **Section 1: INSTALAÇÃO & BOOTSTRAP 100% AUTOMÁTICOS**.

I'll aim to expand on the existing points, add clarity, and incorporate considerations from our discussions on the `12.x` sections.

---

## 1. INSTALAÇÃO & BOOTSTRAP 100% AUTOMÁTICOS

O `VV-Video-AI-System` é projetado para uma instalação e configuração inicial o mais automatizada e "hands-off" possível, visando a implantação rápida e consistente, especialmente em dispositivos de borda.

### 1.1 One-liner de Instalação (Método Primário)

O método primário de instalação é através de um comando único que baixa e executa o script `install.sh`:

```bash
curl -fsSL https://raw.githubusercontent.com/voulezvous-ai/vv-video-ai-system/main/install.sh | sudo bash
```

*   **Funcionamento:**
    *   `curl -fsSL`: Baixa o script. `-f` falha silenciosamente em erros de servidor, `-sS` suprime a barra de progresso mas mostra erros, `-L` segue redirecionamentos.
    *   `| sudo bash`: O script baixado é passado diretamente para o `bash` para execução com privilégios de superusuário (`sudo`).
*   **Considerações de Segurança:**
    *   Este método (`curl | sudo bash`) é conveniente, mas implica um nível de confiança no mantenedor do repositório, pois o script é executado com privilégios elevados sem inspeção prévia pelo usuário.
    *   Para ambientes com políticas de segurança mais rigorosas, recomenda-se baixar o script, inspecioná-lo e, em seguida, executá-lo manualmente:
        ```bash
        curl -fsSL -o install_vv_ai.sh https://raw.githubusercontent.com/voulezvous-ai/vv-video-ai-system/main/install.sh
        # Inspecionar o conteúdo de install_vv_ai.sh
        chmod +x install_vv_ai.sh
        sudo ./install_vv_ai.sh
        ```

### 1.2 Descrição Detalhada do `install.sh`

O script `install.sh` é idempotente, significando que pode ser executado múltiplas vezes sem causar efeitos colaterais negativos, sempre buscando convergir o sistema para o estado desejado.

*   **1. Auto-elevação de Privilégios:**
    *   O script verifica se está sendo executado como root. Se não, tenta re-executar-se usando `sudo "$0" "$@"`, passando quaisquer argumentos originais. Isso garante que as operações que exigem privilégios elevados (instalação de pacotes, configuração de systemd) sejam bem-sucedidas.

*   **2. Instalação Silenciosa de Dependências:**
    *   Identifica o gerenciador de pacotes do sistema (e.g., `apt` para Debian/Ubuntu, `yum`/`dnf` para RHEL/CentOS/Fedora – o script deve ser robusto para diferentes distribuições Linux comuns em edge devices ou especificar as suportadas).
    *   Instala as seguintes dependências essenciais, se ausentes:
        *   `docker`: Motor de containerização.
        *   `docker-compose`: Ferramenta para definir e rodar aplicações Docker multi-container. (Verificar a versão correta, e.g., Docker Compose V2 plugin `docker compose`).
        *   `git`: Para clonar o repositório.
        *   `python3` e `python3-pip`: Para executar scripts Python.
        *   `ffmpeg`: Para processamento de vídeo (e.g., por `vv_ingest` ou se necessário para análise).
        *   `jq`: Utilitário de linha de comando para processamento de JSON, útil para scripts ou validações.
    *   A instalação é feita de forma "silenciosa" ou "não interativa" (e.g., `apt-get update && apt-get install -y <pacote>`).
    *   Verifica se os comandos principais (e.g., `docker`, `python3`) estão disponíveis após a tentativa de instalação.

*   **3. Clone/Atualização do Repositório:**
    *   O repositório do sistema é clonado ou atualizado em `/opt/vv-video-ai-system`.
    *   **Lógica de Idempotência:**
        *   Se o diretório `/opt/vv-video-ai-system` não existir, clona o repositório (`git clone <URL_REPO> /opt/vv-video-ai-system`).
        *   Se o diretório existir e for um repositório Git válido, executa `git pull` para buscar as últimas alterações da branch principal.
        *   **Manuseio de Conflitos/Alterações Locais:** O script pode optar por uma estratégia mais agressiva como `git reset --hard origin/main` para garantir um estado limpo, mas isso descartaria quaisquer alterações locais não commitadas no diretório. Essa estratégia deve ser claramente documentada, e talvez um backup de arquivos de configuração modificados localmente (como `.env`) seja feito antes do reset. Alternativamente, pode avisar sobre alterações locais e pausar.

*   **4. Execução do `bootstrap.py`:**
    *   Navega para `/opt/vv-video-ai-system` e executa `python3 bootstrap.py --skip-existing`.
    *   A flag `--skip-existing` (detalhada na Seção 2) garante que o `bootstrap.py` não sobrescreva configurações ou dados existentes desnecessariamente, tornando-o também idempotente.
    *   O `bootstrap.py` é responsável por criar a estrutura de diretórios, baixar modelos de IA, copiar assets de fallback e gerar um `.env` inicial a partir do `.env.example` (ver Seção 2 e 12.7).

*   **5. Validações Pós-Bootstrap:**
    *   **Existência de Assets Chave:** Verifica se `fallback/loop_silencioso.mp4` existe em `${VIDEO_STORAGE_ROOT}/fallback/` (caminho definido pelo `bootstrap.py`).
    *   **Configuração de `DVR_IP`:**
        *   Verifica se o arquivo `.env` (em `/opt/vv-video-ai-system/.env`) existe.
        *   Verifica se a variável `DVR_IP` está definida e não está vazia no `.env`. Pode incluir uma validação de formato básica (e.g., se parece um IP ou hostname).
        *   Se `DVR_IP` não estiver configurado, o script pode emitir um aviso proeminente ou pausar, instruindo o usuário a configurar esta variável essencial.
    *   **Outras Validações:** Poderia incluir verificação de permissões nos diretórios de armazenamento.

*   **6. Preparação e Inicialização dos Containers Docker:**
    *   `docker-compose pull`: Baixa as versões mais recentes das imagens Docker definidas no `docker-compose.yml` (especialmente útil para imagens base ou públicas como `containrrr/watchtower`).
    *   `docker-compose up -d --build`:
        *   `--build`: Constrói as imagens locais (e.g., `agent_dvr`, `scan_video`) se elas não existirem ou se seus Dockerfiles/contextos foram alterados (Docker detecta isso).
        *   `-d`: Inicia os containers em modo detached (background).
        *   Garante que todos os serviços definidos no `docker-compose.yml` sejam iniciados.

*   **7. Configuração do Serviço Systemd:**
    *   Cria ou atualiza o arquivo de unit systemd `vv-video-ai.service` em `/etc/systemd/system/`.
    *   Conteúdo do `vv-video-ai.service` (exemplo):
        ```ini
        [Unit]
        Description=VV Video AI System
        Requires=docker.service
        After=docker.service network-online.target

        [Service]
        User=app_user # Recomenda-se um usuário dedicado, não root
        Group=app_group
        WorkingDirectory=/opt/vv-video-ai-system
        Restart=on-failure
        RestartSec=5s
        ExecStart=/usr/local/bin/docker-compose up # Ou caminho completo para docker compose plugin
        ExecStop=/usr/local/bin/docker-compose down
        # Variáveis de ambiente podem ser carregadas de um EnvironmentFile aqui, se não via .env

        [Install]
        WantedBy=multi-user.target
        ```
    *   Cria ou atualiza o timer opcional `vv-video-ai.timer` (se um delay no boot for desejado).
    *   Executa `sudo systemctl daemon-reload` para que o systemd reconheça as mudanças.
    *   Executa `sudo systemctl enable vv-video-ai.service` (e o timer, se aplicável) para iniciar no boot.
    *   Executa `sudo systemctl start vv-video-ai.service` para iniciar o sistema imediatamente (a menos que o timer esteja configurado para atrasar).
    *   **Usuário de Execução:** O script deve criar um usuário e grupo dedicados (`app_user:app_group`) se não existirem, para rodar os serviços Docker e os cronjobs, em vez de usar `root`. As permissões em `/opt/vv-video-ai-system` e `${VIDEO_STORAGE_ROOT}` devem ser ajustadas para este usuário.

*   **8. Configuração de Cronjobs Idempotentes:**
    *   Os cronjobs são adicionados à crontab do usuário `app_user` (ou outro usuário designado).
    *   A idempotência é garantida verificando se o cronjob já existe com o mesmo comando antes de adicioná-lo, ou removendo e recriando.
    *   Exemplos de cronjobs configurados:
        *   **Orquestração:** `*/5 * * * * /usr/bin/python3 /opt/vv-video-ai-system/scripts/orchestrator.py >> /opt/vv-video-ai-system/logs/orchestrator_cron.log 2>&1`
            *   Roda a cada 5 minutos.
            *   Redireciona stdout e stderr para um arquivo de log específico.
        *   **Limpeza (Inicial Dry-Run):** `*/10 * * * * /usr/bin/python3 /opt/vv-video-ai-system/scripts/cleanup_old_files.py --dry-run --days 30 --dirs "${VIDEO_STORAGE_ROOT}/processed" >> /opt/vv-video-ai-system/logs/cleanup_dryrun_cron.log 2>&1`
            *   Roda a cada 10 minutos (esta frequência para dry-run é alta, pode ser ajustada para horária).
            *   Reforça a necessidade de configurar a limpeza efetiva conforme Seção `12.3`.
        *   **Exportação de Logs:** `0 2 * * * /usr/bin/python3 /opt/vv-video-ai-system/scripts/log_exporter.py >> /opt/vv-video-ai-system/logs/log_exporter_cron.log 2>&1`
            *   Roda diariamente às 02:00.
    *   É importante que os caminhos para `python3` e os scripts sejam absolutos. A variável `VIDEO_STORAGE_ROOT` deve estar disponível no ambiente do cron ou ser substituída pelo caminho absoluto no comando do cronjob.

*   **9. Logging da Instalação:**
    *   O script `install.sh` deve logar suas principais ações em `stdout` e também em um arquivo de log (e.g., `/var/log/vv_video_ai_install.log`) para facilitar o troubleshooting em caso de falhas na instalação.

*   **10. Mensagem de Conclusão:**
    *   Ao final, exibe uma mensagem indicando que a instalação/atualização foi concluída, o status dos serviços e quaisquer próximos passos recomendados (e.g., "Verifique a configuração de `DVR_IP` em `/opt/vv-video-ai-system/.env` se for a primeira instalação.").

### 1.3 Configuração Inicial Pós-Instalação

Embora o `install.sh` automatize muito, algumas verificações ou configurações manuais podem ser necessárias, especialmente na primeira vez:

1.  **Verificar/Definir `DVR_IP`:** E outras credenciais ou configurações específicas do ambiente no arquivo `/opt/vv-video-ai-system/.env`. Após modificar o `.env`, um `docker-compose restart agent_dvr` (ou o serviço que usa a variável) pode ser necessário.
2.  **Segurança do `.env`:** Garantir que o arquivo `/opt/vv-video-ai-system/.env` tenha permissões restritas (e.g., `chmod 600 .env`) e seja propriedade do `app_user`. Este ponto conecta-se com as recomendações da Seção `12.7` sobre gestão de segredos.
3.  **Verificar Status dos Serviços:** `sudo systemctl status vv-video-ai.service` e `docker-compose ps`.
4.  **Revisar Logs Iniciais:** Verificar `logs/app.log` e logs de containers individuais para quaisquer erros de inicialização.
5.  **Configurar Limpeza Efetiva:** Modificar o cronjob de `cleanup_old_files.py` para remover a flag `--dry-run` conforme a política de retenção definida (Seção `12.3`).

---

This expanded version of Section 1 provides much more detail on the installation process, the behavior of `install.sh`, and considerations for security and robustness. It also sets the stage for a more managed system by mentioning user accounts and log management during installation.
