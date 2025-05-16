# Manual do Usuário

## Sobre o sistema

O **voulezvous-ai/tv-details** é um sistema robusto para automação, gerenciamento e monitoramento de fluxos de vídeo com inteligência artificial. Este manual orienta o usuário final na instalação, configuração e utilização do sistema.

---

## Requisitos mínimos

- Computador com Linux, Windows ou MacOS
- Docker e Docker Compose instalados ([Instruções de instalação do Docker](https://www.docker.com/products/docker-desktop/))
- Git instalado ([Instruções de instalação do Git](https://git-scm.com/downloads))
- Acesso à internet

---

## Instalação rápida

1. **Baixe o repositório:**
   ```sh
   git clone https://github.com/voulezvous-ai/tv-details.git
   cd tv-details
   ```

2. **Inicie o sistema:**
   ```sh
   docker compose up
   ```

3. **Acesse o painel:**
   - Abra o navegador e acesse [http://localhost:8080](http://localhost:8080)

---

## Primeiros passos

- Siga as instruções da tela para criar sua conta e configurar o ambiente inicial.
- Utilize o painel de controle para validar que todos os serviços estejam ativos (status verde).
- Em caso de erro, reinicie o Docker e repita o processo.

---

## Exemplos de uso

- **Cadastrar novo fluxo de vídeo:**  
  Acesse "Adicionar Fluxo", preencha as informações e clique em salvar.
- **Visualizar métricas:**  
  Vá até "Monitoramento" para ver gráficos e indicadores em tempo real.
- **Exportar logs:**  
  Acesse "Logs" no menu e faça o download dos registros das operações.

---

## Solução de problemas

- **Não abre no navegador:**  
  Certifique-se de que o Docker está rodando e o endereço está correto (`localhost:8080`).
- **Erro de permissão:**  
  Execute o terminal como administrador (Windows) ou root (Linux/Mac).
- **Dúvidas frequentes:**  
  Consulte o FAQ.md ou abra uma issue no [GitHub do projeto](https://github.com/voulezvous-ai/tv-details/issues).

---

## Glossário

- **Docker:** Plataforma para rodar aplicações isoladas em containers.
- **Pipeline:** Sequência automatizada de passos como build, teste e deploy.
- **Orquestrador:** Componente que gerencia a execução dos scripts e módulos.

---