# ROADMAP_PIPELINE_DASHBOARD.md

> Plano técnico para evolução do MSConecta de "bot Telegram como único painel" para um
> pipeline de produção editorial com visibilidade e controle centralizados. Documento
> de referência para todas as sessões de Claude Code que trabalharem neste projeto —
> ler antes de iniciar qualquer fase abaixo.
>
> Criado em 2026-08-13, motivado pelo crescimento de escala do MSConecta (~30
> notícias/dia, ~2 Reels/dia) e por incidentes reais do mesmo dia que expuseram a
> fragilidade do modelo atual de estado (ver `HISTORICO_MUDANCAS.md`, entradas de
> 2026-08-13): publicações reportadas como bem-sucedidas em log que na prática
> falharam silenciosamente, sem nenhum mecanismo de verificação real nem visão
> unificada do que está acontecendo no sistema.

---

## 1. Motivação

O MSConecta hoje depende inteiramente do **Telegram como painel de controle**: o editor
aprova, ajusta e publica tudo por mensagens e botões inline, sem nenhuma visão
consolidada de:

- Quantas notícias estão em cada estágio do pipeline agora.
- O que está programado para sair hoje (planner/calendário).
- Se uma publicação que "deu certo" no log realmente saiu de verdade (ver o
  incidente do canal do Instagram silenciosamente parado por dias, 2026-08-13).
- A saúde real dos processos em ambos os ambientes (VPS e PC), além de "a porta
  responde" (ver o incidente do processo zumbi na porta 8765, mesmo dia).

Na escala atual (30 notícias/dia), esse modelo já é difícil de auditar manualmente.
Continuar crescendo sem um pipeline visual e um modelo de dados unificado aumenta o
risco de problemas como os de 2026-08-13 passarem despercebidos por mais tempo.

## 2. Objetivo

Construir um **dashboard de pipeline de produção editorial** que dê visão e controle
sobre todo o ciclo de vida de uma notícia ou Reel — desde a descoberta da pauta até a
confirmação real de publicação — sem substituir abruptamente o fluxo do Telegram (que
continua útil para aprovações rápidas via celular), mas centralizando a lógica de
negócio numa camada de serviço compartilhada entre os dois.

## 3. Princípios de design

- **Não quebrar o que está em produção.** Cada fase deve ser aditiva; o fluxo do
  Telegram continua funcionando durante toda a transição.
- **Um modelo de dados único, não mais JSONs espalhados.** `estado.json`,
  `agendamentos.json`, `historico_publicacoes.json`, `noticias_vistas.json`,
  `videos_sugestoes.json` viram uma única fonte de verdade.
- **"Publicado" só depois de verificado de verdade.** Aplicar a lição do incidente do
  canal do Instagram: nunca confiar só na resposta HTTP de outro processo como prova
  de que algo aconteceu no mundo real.
- **Saúde de processo = identidade do processo, não só resposta de porta.** Aplicar a
  lição do processo zumbi: um watchdog/health-check deve confirmar qual processo está
  respondendo, não só que *algo* respondeu.
- **Reaproveitar, não recriar.** `editor_visual.py` e sua UI de posicionamento de
  imagem já funcionam bem — o dashboard novo deve embutir/reutilizar essa peça, não
  reescrevê-la.
- **Cada fase é standalone e entregável.** Nenhuma fase depende de "terminar tudo"
  para gerar valor.

## 4. Modelo de dados (Fase 0)

Banco proposto: **SQLite** (arquivo único, zero infraestrutura nova, mais que
suficiente para o volume — milhares de linhas por ano, não milhões).

### Tabela `content_items` (notícias)

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | INTEGER PK | |
| `noticia_id` | TEXT | ID/slug da notícia no CMS do site |
| `titulo` | TEXT | |
| `url_site` | TEXT | URL publicada no site |
| `editoria` | TEXT | |
| `estagio` | TEXT | Ver enum de estágios abaixo |
| `origem` | TEXT | `bloco1`, `bloco2`, `monitor_noticias`, `manual` |
| `criado_em` | DATETIME | |
| `atualizado_em` | DATETIME | |
| `agendado_para` | DATETIME NULL | Se programado |
| `feed_path` / `story_path` | TEXT NULL | Caminho das imagens geradas |
| `legenda` | TEXT NULL | |
| `publicado_feed_em` / `publicado_story_em` / `publicado_canal_em` | DATETIME NULL | Timestamp de cada sub-publicação |
| `verificado_canal_em` | DATETIME NULL | Quando a verificação REAL confirmou o post no canal |
| `erro_detalhe` | TEXT NULL | Último erro relevante, se houver |

**Estágios (`estagio`)**:

```
descoberta → redacao → design_gerado → aguardando_aprovacao →
ajustando → aprovado → agendado → publicando →
publicado_parcial → publicado_completo → verificado → erro
```

### Tabela `reels`

Mesma lógica, com estágios próprios:

```
descoberta_video → aprovado_curadoria → renderizando →
aguardando_aprovacao → publicando → publicado → verificado → erro
```

Colunas adicionais: `duracao_segundos`, `elegivel_reels` (bool, com base no limite de
~90s da Graph API), `render_path`, `render_origem` (`pc_windows` | `remotion_vps` |
`ffmpeg_basico_vps` — reaproveitando a cadeia de fallback já documentada na seção 3.6
do `CONTEXTO_MSCONECTA.md`).

### Tabela `pipeline_events`

Log de auditoria de toda transição de estágio: `content_item_id`, `estagio_anterior`,
`estagio_novo`, `timestamp`, `origem` (`telegram` | `dashboard` | `automatico`),
`detalhe`.

### Tabela `system_health`

`servico` (ex: `msconecta-bot`, `msconecta-api`, `render_pc`, `story_listener_pc`),
`ambiente` (`vps` | `pc`), `status` (`ok` | `degradado` | `falha`), `pid_esperado`,
`pid_atual`, `checado_em`, `detalhe`. O check precisa validar **identidade do
processo** (caminho do executável/args), não só resposta HTTP — ver princípio de
design acima.

### Migração

Escrever um script de migração único que lê os JSONs existentes e popula essas
tabelas, preservando o histórico existente onde possível. Os JSONs continuam sendo
escritos em paralelo durante a transição (dual-write), até todos os fluxos migrarem
para ler/escrever direto no banco.

## 5. Arquitetura em camadas

```
┌─────────────────────────────────────────────┐
│ Frontend (Fase 1+)                           │
│ Board/Kanban · Planner · Preview · Saúde     │
└───────────────────┬───────────────────────────┘
                    │ HTTP/REST
┌───────────────────▼───────────────────────────┐
│ Camada de serviço (API) — Fase 2               │
│ Mesmas ações hoje só acessíveis via Telegram:  │
│ aprovar, ajustar, agendar, reenviar, publicar  │
└──────┬──────────────────────────────┬──────────┘
       │                              │
┌──────▼──────────┐          ┌────────▼─────────┐
│ Telegram bot     │          │ Dashboard novo    │
│ (cliente 1)      │          │ (cliente 2)       │
└──────────────────┘          └───────────────────┘
       │                              │
       └─────────────┬──────────────┘
       ┌─────────────▼──────────────┐
       │ SQLite (Fase 0)             │
       │ content_items · reels ·     │
       │ pipeline_events ·           │
       │ system_health               │
       └───────────────────────────────┘
```

**Stack recomendado** (aproveitando o que já existe no projeto):
- Backend/API: **FastAPI** (já em uso em `editor_visual.py` — expandir, não recriar).
- Frontend: React (SPA) ou HTML+htmx se preferir manter simplicidade — decisão pode
  ficar para o início da Fase 1, não bloqueia a Fase 0.
- Banco: SQLite, acessado via `sqlite3` (stdlib) ou SQLAlchemy se a complexidade
  justificar.

## 6. Fases

### Fase 0 — Modelo de dados unificado
**Entrega:** schema SQLite criado, script de migração dos JSONs existentes, dual-write
implementado nos pontos que já escrevem `estado.json`/`agendamentos.json`/etc.
**Critério de conclusão:** o banco reflete fielmente o estado atual do sistema, sem
nenhuma mudança de comportamento visível no Telegram.
**Risco:** baixo.

### Fase 1 — Dashboard somente leitura
**Entrega:** board (Kanban) por estágio, planner do dia (visão do que está agendado),
painel de saúde do sistema (lendo `system_health`), tudo em modo leitura.
**Critério de conclusão:** dá pra ver, sem usar o Telegram, tudo que está em produção
agora — quantas notícias em cada estágio, o que sai hoje, o que está saudável/quebrado.
**Risco:** baixo — pura visibilidade, nenhuma ação nova.

### Fase 2 — Ações no dashboard
**Entrega:** aprovar/rejeitar/ajustar/agendar/reenviar a partir do dashboard, via a
camada de serviço compartilhada (refatorando a lógica hoje embutida em
`orquestrador.py`/`telegram_bot.py`).
**Critério de conclusão:** um editor consegue rodar o dia inteiro sem abrir o
Telegram, se preferir — mas o Telegram continua funcionando em paralelo.
**Risco:** médio — toca a lógica de produção real; exige testes cuidadosos e rollback
fácil.

### Fase 3 — Revisão de Reels integrada
**Entrega:** preview de vídeo inline, aprovação/reprovação de Reel direto no
dashboard, visibilidade da cadeia de fallback de render (PC → Remotion VPS → FFmpeg
básico) e do aviso de elegibilidade (~90s) já implementado.
**Risco:** médio.

### Fase 4 — Verificação real de publicação
**Entrega:** job de reconciliação que confirma publicação de verdade (ID do post via
Graph API, e/ou verificação visual do canal via CDP como foi feito manualmente em
2026-08-13), atualizando `verificado_canal_em`/`verificado` só após confirmação real.
Alertas ativos quando um item fica "publicado" no log mas não "verificado" depois de
X minutos.
**Critério de conclusão:** o incidente de 2026-08-13 (canal mudo por dias, log dizendo
sucesso) se torna estruturalmente impossível de passar despercebido.
**Risco:** médio-alto — é a mudança mais importante em termos de confiabilidade, mas
também a que mexe mais fundo na lógica de publicação existente.

## 7. Fora de escopo (por ora)

- Substituir o Telegram completamente — ele continua como canal de aprovação rápida.
- Migrar para um banco relacional mais robusto (Postgres etc.) — SQLite é suficiente
  para este volume; reavaliar só se a escala mudar de ordem de grandeza.
- Resolver o `ETIMEDOUT` do Remotion na VPS (pendência separada, já registrada em
  `CONTEXTO_MSCONECTA.md` seção 3.6) — não bloqueia nenhuma fase deste roadmap.
- Migração completa das credenciais hardcoded para `portal_config.py` (pendência já
  registrada na seção 6 de `CONTEXTO_MSCONECTA.md`) — desejável, mas paralela a este
  roadmap, não pré-requisito.

## 8. Como usar este documento

Cada sessão de Claude Code que for trabalhar numa fase deste roadmap deve:
1. Ler este documento inteiro antes de começar.
2. Confirmar qual fase está em andamento (ver `HISTORICO_MUDANCAS.md` para o estado
   mais recente de progresso).
3. Ao concluir uma fase, registrar em `HISTORICO_MUDANCAS.md` normalmente, e atualizar
   a seção correspondente deste roadmap se algo mudou de plano durante a implementação.
