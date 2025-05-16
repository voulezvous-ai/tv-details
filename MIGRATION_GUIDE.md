# Guia de Migração e Atualização

Procedimento seguro para migrar/atualizar o **voulezvous-ai/tv-details**.

---

## Antes de migrar

1. Faça backup completo.
2. Leia [CHANGELOG.md](CHANGELOG.md) e [RELEASES.md](RELEASES.md).
3. Planeje janela de manutenção.

---

## Atualização padrão

```sh
git pull
docker compose pull
docker compose up --build -d
```

---

## Ajustes pós-update

- Compare `.env` com novo `.env.example`.
- Revise configs em `/config`.

---

## Migração de banco

- Siga `/migrations` se scripts existirem.
- Veja instruções do release.

---

## Rollback

Restaure o backup anterior se necessário.

---

## Suporte

Consulte [SUPPORT.md](SUPPORT.md).

---