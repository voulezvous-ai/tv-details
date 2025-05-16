# Manual Operacional e de Administração

Este manual é destinado a administradores e operadores do sistema, cobrindo rotinas essenciais de monitoramento, backup, automação e segurança.

---

## Monitoramento do sistema

- Acesse o painel Grafana em `http://localhost:3000` (padrão) para visualizar métricas em tempo real.
- O Prometheus coleta informações de CPU, memória, latência e status de todos os módulos.

---

## Backup e restauração

- Os backups automáticos seguem a configuração detalhada em `12.8 🛡️ Estratégia de Backup e Recuperação de Dados.md`.
- Para restaurar manualmente, execute:
  ```sh
  docker exec -it <container> restore-backup.sh <backup_file>
  ```

---

## Gestão de segredos

- Segredos e chaves ficam nos arquivos `.env` ou são gerenciados via painel administrativo seguro.
- Atualize segredos conforme as melhores práticas de segurança.

---

## Automação e tarefas periódicas

- Cronjobs realizam limpeza automática, verificação de integridade e outras rotinas.
- Scripts auxiliares estão documentados em `Section 6: SCRIPTS AUXILIARES.md`.

---

## Atualização segura

1. Faça backup antes de atualizar.
2. Atualize o código e reinicie os containers:
   ```sh
   git pull
   docker compose up --build -d
   ```

---