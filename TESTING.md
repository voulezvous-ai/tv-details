# Guia de Testes

Como testar o **voulezvous-ai/tv-details** para garantir robustez e qualidade.

---

## Tipos de teste

- **Unitários:** Funções/métodos isolados.
- **Integração:** Comunicação entre módulos.
- **End-to-end:** Fluxos completos do usuário.

---

## Como executar

1. Instale dependências:
   ```sh
   pip install -r requirements-dev.txt
   ```
2. Execute todos os testes:
   ```sh
   ./scripts/run_tests.sh
   ```
   ou
   ```sh
   pytest tests/
   ```

---

## Exemplo (Python)

```python
def test_video_processing_returns_success():
    response = video_processor.process("amostra.mp4")
    assert response.status == "success"
```

---

## Boas práticas

- Todo novo módulo precisa de testes.
- Automatize testes no CI/CD.
- Mantenha alta cobertura (>= 80%).
- Corrija falhas antes do PR.

---

## Avançado

- Use `locust` para carga.
- Integre E2E ao pipeline.

---