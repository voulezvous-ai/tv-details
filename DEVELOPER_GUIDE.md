# Guia do Desenvolvedor

Oriente-se aqui para criar plugins, integrações e contribuir com o core do **voulezvous-ai/tv-details**.

---

## 1. Ambiente

- Clone o repositório
- Instale dependências:
  ```sh
  python -m venv venv
  source venv/bin/activate
  pip install -r requirements.txt
  ```

---

## 2. Estrutura

- `/src` — Orquestrador, APIs, módulos principais
- `/plugins` — Plugins e módulos de IA
- `/scripts` — Scripts utilitários
- `/config` — Arquivos de configuração
- `/docs` — Documentação

---

## 3. Novo módulo/plugin

1. Crie pasta em `/plugins/<seu_modulo>`.
2. Implemente métodos padrão (init, process, shutdown).
3. Registre no orquestrador (`src/orchestrator.py`).
4. Adicione testes em `/tests`.
5. Documente o módulo (README próprio).

---

## 4. Boas práticas

- Siga PEP8 (Python) e padrões do projeto.
- Comente e use docstrings.
- Logging estruturado.
- Baixo acoplamento.

---

## 5. Testes

Veja [TESTING.md](TESTING.md).  
Execute testes localmente antes do PR.

---

## 6. Checklist PR

- Todos testes passam.
- Código documentado.
- PR bem descrito.

---

Dúvidas? Use discussions ou issues no GitHub.

---