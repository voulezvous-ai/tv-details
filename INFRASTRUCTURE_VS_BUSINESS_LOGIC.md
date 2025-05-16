# Infraestrutura vs Lógica de Negócio no voulezvous-ai/tv-details

## Introdução

Ao analisar sistemas modernos como o **voulezvous-ai/tv-details**, é essencial diferenciar o que faz parte da **infraestrutura** e o que constitui a **lógica de negócio**. Essa separação facilita a manutenção, customização e evolução do sistema.

---

## O que é Infraestrutura?

Infraestrutura corresponde à base técnica que garante funcionamento, escalabilidade, segurança e confiabilidade do sistema, mas **não define as regras do seu negócio**. Exemplos no projeto incluem:

- Ambiente de execução (Docker, Docker Compose)
- Bootstrap e scripts de setup inicial
- Orquestração e deploy (CI/CD, cronjobs)
- Monitoramento & observabilidade (Prometheus, Grafana, logs)
- Segurança (NGINX + JWT, gestão de segredos, criptografia)
- Backup & restauração (rotinas, scripts)
- Automação de tarefas técnicas

A infraestrutura raramente é alterada para atender mudanças de negócio, mas pode ser customizada para escala, integração ou requisitos técnicos.

---

## O que é Lógica de Negócio?

Lógica de negócio é o conjunto de regras, operações e decisões que definem o funcionamento do sistema segundo as necessidades do seu domínio. No contexto deste projeto:

- Processamento de vídeo e IA (modelos, critérios de análise)
- Workflows e sequenciamento de módulos/plugins
- Regras de sucesso, erro, alerta, intervenção humana
- Parâmetros operacionais (thresholds, limites, prioridades)
- Configuração de módulos/plugins
- Relatórios, métricas e dashboards relevantes para o negócio
- Integrações específicas com sistemas externos

A lógica de negócio deve ser customizada conforme as necessidades da sua organização.

---

## Comparativo prático

| Infraestrutura                                 | Lógica de Negócio                                 |
|------------------------------------------------|---------------------------------------------------|
| Docker, Compose, CI/CD, scripts de setup       | Módulos/Plugins de processamento de vídeo/IA      |
| Segurança, NGINX, JWT, backups                 | Workflow de análise, regras de alerta             |
| Monitoramento (Prometheus, Grafana), logs      | Critérios de sucesso/erro, thresholds             |
| Cronjobs técnicos, automação de manutenção     | Parâmetros de processamento, relatórios           |
| Configuração de rede, balanceamento            | Integrações com sistemas de negócio               |

---

## O que você pode decidir?

- **Infraestrutura:**  
  Ajustar para performance, segurança, escalabilidade ou integração técnica.
- **Lógica de negócio:**  
  Personalizar fluxos, regras, modelos de IA, integrações e relatórios conforme seu negócio.

---

## Perguntas para Customização da Lógica de Negócio

A seguir, uma lista refinada de perguntas para orientar a customização do sistema, com sugestões padrão para cada uma.

### 1. Fluxo de Processamento e Workflows

**1. Quais etapas (módulos) compõem o fluxo padrão de processamento de vídeo?**  
_Default:_ Todas as etapas principais do sistema, na ordem sugerida pela documentação.

**2. Deseja ativar ou desativar algum módulo/plugin?**  
_Default:_ Todos os módulos recomendados ativos.

**3. A ordem de execução dos módulos deve ser alterada?**  
_Default:_ Ordem padrão conforme a documentação.

**4. Existem condições para pular ou repetir etapas conforme o resultado de um módulo?**  
_Default:_ Não; todos os módulos executam sequencialmente.

---

### 2. Parâmetros Operacionais e Thresholds

**5. Quais parâmetros (thresholds) cada módulo deve usar?**  
_Default:_ Valores recomendados pela documentação técnica.

**6. Quais critérios definem sucesso, falha ou alerta em cada etapa?**  
_Default:_ Critérios dos módulos padrão (ex: erro = score < 0.6).

**7. Quais tipos de vídeo devem ser aceitos/rejeitados?**  
_Default:_ Todos os formatos suportados/documentados (MP4, MOV, AVI).

---

### 3. Modelos de Inteligência Artificial

**8. Quais modelos de IA devem ser utilizados em cada etapa?**  
_Default:_ Modelos padrão embarcados e validados pelo projeto.

**9. Deseja treinar/importar modelos customizados?**  
_Default:_ Apenas modelos oficiais, exceto quando indicado por especialistas.

---

### 4. Integrações e Interoperabilidade

**10. O sistema deve enviar resultados/alertas para outros sistemas (BI, CRM, etc)?**  
_Default:_ Não; integração apenas via exportação manual ou APIs já integradas.

**11. Quais APIs externas podem ser chamadas durante o processamento?**  
_Default:_ Apenas as pré-aprovadas/documentadas.

**12. Como tratar falhas na comunicação com sistemas externos?**  
_Default:_ Repetir tentativa até 3 vezes; se falhar, registrar alerta.

---

### 5. Relatórios e Métricas

**13. Quais indicadores e métricas são importantes para seu negócio?**  
_Default:_ Métricas padrão (volume, tempo de processamento, taxa de erro/sucesso).

**14. Relatórios e dashboards devem ser customizados?**  
_Default:_ Dashboards padrão Grafana/Prometheus.

**15. Com que frequência relatórios devem ser gerados/enviados?**  
_Default:_ Semanais, enviados manualmente ou sob demanda.

---

### 6. Alertas e Notificações

**16. Quando e como disparar alertas para a equipe?**  
_Default:_ Apenas em falhas críticas, via e-mail configurado.

**17. Quais eventos devem ser notificados ou ignorados?**  
_Default:_ Eventos críticos notificados; eventos menores apenas logados.

**18. Quem deve receber cada tipo de alerta?**  
_Default:_ Equipe técnica principal.

---

### 7. Usuários e Permissões

**19. Quem pode acessar/configurar módulos e fluxos?**  
_Default:_ Apenas administradores/usuários técnicos.

**20. Perfis de acesso diferenciados são necessários?**  
_Default:_ Não, todos os usuários técnicos têm o mesmo perfil.

---

### 8. Agendamento e Automação

**21. Existem horários específicos para rodar certos módulos ou fluxos?**  
_Default:_ Não, processamento contínuo.

**22. Deseja agendar tarefas além das rotinas padrão (backup, limpeza)?**  
_Default:_ Apenas rotinas básicas já implementadas.

---

### 9. Regras de Negócio Específicas

**23. Existem regras específicas do seu domínio a implementar?**  
_Default:_ Não, seguir regras padrão do sistema.

**24. Deseja adicionar validações exclusivas no início/fim do workflow?**  
_Default:_ Não, apenas as existentes.

---

### 10. Outros

**25. Como tratar dados sensíveis ou confidenciais?**  
_Default:_ Seguir práticas de criptografia e anonimização padrão.

**26. Como proceder em caso de erro irreversível?**  
_Default:_ Registrar alerta crítico, pausar fluxo e aguardar intervenção manual.

---

### Como usar esta lista

- Utilize estas perguntas em reuniões, onboarding ou revisões para decidir onde e como customizar o sistema.
- Para cada resposta diferente do default, registre a decisão e implemente a customização correspondente na configuração, código ou documentação.

Se desejar exemplos práticos de customização para qualquer pergunta, solicite informando o número da questão!

---