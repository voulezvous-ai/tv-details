# Guia de Instalação

Siga este guia para instalar o **voulezvous-ai/tv-details** em ambientes de desenvolvimento ou produção.

---

## 1. Pré-requisitos

- Docker ([instalar](https://docs.docker.com/get-docker/))
- Docker Compose ([instalar](https://docs.docker.com/compose/install/))
- Git ([instalar](https://git-scm.com/downloads))
- Linux, Windows ou MacOS
- Recomendado: 8GB RAM+

---

## 2. Instalação Rápida

```sh
git clone https://github.com/voulezvous-ai/tv-details.git
cd tv-details
cp .env.example .env
# Edite .env conforme necessário
docker compose up -d
```

---

## 3. Acesso

- Interface web: [http://localhost:8080](http://localhost:8080)
- Métricas (Grafana): [http://localhost:3000](http://localhost:3000)
- API REST: [http://localhost:8080/api](http://localhost:8080/api)

---

## 4. Dicas de troubleshooting

- **Porta ocupada:** Altere portas no `docker-compose.yml`.
- **Permissões:** Use terminal como admin/root se necessário.
- **Containers não sobem:** Veja logs com `docker compose logs <serviço>`.

---

## 5. Teste pós-instalação

- Acesse o painel e confira serviços ativos.
- Faça upload de vídeo teste e valide processamento.

---

## 6. Desinstalação

```sh
docker compose down -v
```

---

## 7. Suporte

Consulte [SUPPORT.md](SUPPORT.md) ou abra uma issue no GitHub.

---