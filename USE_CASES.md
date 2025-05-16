# Casos de Uso e Exemplos Avançados

## Integração com outros sistemas

- **API REST:** Envie vídeos para processamento e receba resultados via webhook.
- **Exemplo de envio:**
  ```sh
  curl -X POST http://localhost:8080/api/videos -F file=@meu_video.mp4
  ```

---

## Fluxo típico de uso

1. Upload do vídeo pelo usuário ou sistema externo.
2. Processamento automático por módulos de IA definidos.
3. Monitoramento do status em tempo real no painel.
4. Recebimento dos resultados via interface ou webhook.

---

## Scripts úteis

- **Backup manual:**  
  ```sh
  ./scripts/backup.sh
  ```
- **Limpeza de dados antigos:**  
  ```sh
  ./scripts/cleanup.sh
  ```

---

## Docker Compose

- **Executar o sistema:**
  ```sh
  docker compose up
  ```
- **Atualizar imagens:**
  ```sh
  docker compose pull
  docker compose up -d
  ```

---

## Dicas

- Combine módulos para criar fluxos personalizados (ex: processamento + análise + alertas).
- Use o painel de métricas para identificar gargalos e otimizar performance.

---