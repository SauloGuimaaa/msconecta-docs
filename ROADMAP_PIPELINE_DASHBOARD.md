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

### 4.1 Correções ao schema a partir dos dados reais (Fase 0, executada em 2026-08-14)

Ao implementar a Fase 0 e ler os JSONs de produção linha a linha (não só a
documentação), várias premissas deste roadmap precisaram de ajuste. Registradas aqui
para qualquer sessão futura não repetir a mesma investigação:

- **Notícias e Reels não são arquivos separados** — ambos vivem juntos em
  `agendamentos.json` (lista única), diferenciados pelo campo `tipo` (`null` = notícia
  feed/story, `"reel"` = Reel). O script de migração precisa dividir por esse campo,
  não ler de dois arquivos distintos.
- **Estágio `cancelado` estava faltando no enum** — produção tem notícias com
  `status: "cancelado"` (reagendamentos descartados). Adicionado a ambos os enums
  (`content_items` e `reels`).
- **A identidade real de um item hoje é a `pasta`** (diretório de output), não um ID
  de notícia — `noticia_id`/`titulo`/`url_site`/`editoria` não existem em
  `agendamentos.json` para notícias (só para Reels); vêm de `meta.json` dentro da
  própria pasta, quando esse arquivo ainda existe em disco (79 de 1103 pastas não
  tinham mais `meta.json` no momento da migração — provavelmente limpeza histórica —,
  então essas linhas ficaram com `titulo`/`url_site` nulos). Fallback: `editoria`
  também pode ser extraído do próprio nome da pasta quando `meta.json` falta.
- **9 pastas tinham múltiplas entradas em `agendamentos.json`** (cadeias de
  reagendamento/retry, ex: `cancelado` → `agendado` → `publicado`). O modelo trata a
  **última entrada** como o estado atual do `content_item`, e preserva a cadeia
  completa em `pipeline_events` — validação real e não-sintética de por que essa
  tabela existe.
- **Granularidade de publicação é mais grosseira do que o roadmap assumia**:
  `agendamentos.json` grava só um timestamp `publicado_em` por item, não um por
  feed/story/canal. `publicado_feed_em`/`publicado_story_em` foram preenchidos com o
  mesmo valor (aproximação; `publicado_story_em` fica nulo só quando `feed_only=true`).
  **`publicado_canal_em` e `verificado_canal_em` não existem em nenhum JSON de
  produção hoje** — confirma exatamente a lacuna que motiva a Fase 4; ficaram nulos
  para 100% das linhas migradas.
- **`duracao_segundos`, `elegivel_reels` e `render_origem` de Reels não têm dado
  histórico** — não são gravados em `agendamentos.json`; só passam a existir a partir
  de instrumentação futura (a checagem de duração de 2026-08-13 já ajuda, mas ainda
  não persiste o valor num campo estruturado).
- **`pipeline_events.origem` ganhou um 4º valor: `'migracao'`** — usado para marcar
  honestamente os eventos reconstruídos a partir dos JSONs nesta migração inicial,
  distintos de eventos gerados ao vivo por `telegram`/`dashboard`/`automatico`
  depois que o dual-write existir.
- **`saude_estado.json` está morto/parado desde 2026-06-15** (~2 meses sem escrita
  no momento da migração) — migrado para `system_health` só como registro histórico,
  explicitamente marcado como tal no campo `detalhe`. **Não reflete a saúde atual do
  sistema.** Reforça (com dado real, não hipotético) por que a Fase 1+ precisa de um
  health-check ativo de verdade, e não pode só ler esse arquivo achando que é "ao
  vivo". Além disso, o arquivo nunca gravou `pid_esperado`/`pid_atual` — só
  `ok`/`detalhe` por serviço — ou seja, nem a checagem antiga cumpria o princípio de
  "identidade do processo" definido na seção 3; essa instrumentação ainda não existe
  e precisa ser construída do zero na Fase 1+, não só lida de um arquivo existente.
- **`videos_sugestoes.json`/`videos_vistos.json` (saída de `monitor_videos.py`,
  curadoria automática via YouTube) também estão parados desde 2026-06-15** — mas os
  74 Reels publicados desde então continuaram saindo normalmente, vindos de
  **submissão manual ao bot** (fluxo já documentado em `CONTEXTO_MSCONECTA.md` §Reels:
  "vídeos virais curados **ou enviados manualmente ao bot**"). Ou seja, a curadoria
  automática via YouTube parece dormente na prática, e o fluxo manual é hoje o
  caminho real de produção. Os 12 candidatos pendentes nesse arquivo foram migrados
  para `reels` com `estagio='descoberta_video'` e `origem='migracao_videos_sugestoes_dormente'`
  — marcados explicitamente como dormentes para a Fase 1 não os exibir misturados com
  itens de um fluxo ativo. **Decisão a tomar por Saulo, fora do escopo da Fase 0**:
  reativar `monitor_videos.py` ou descontinuá-lo formalmente.
- **`design_queue.json` é um artefato morto de uma versão anterior do pipeline** —
  sem escrita desde 2026-04-26, e nenhuma de suas 21 URLs aparece em
  `agendamentos.json` (zero sobreposição). Não migrado para `content_items` (sem
  título/dados confiáveis para reconstruir uma linha útil). Candidato a
  arquivamento/remoção numa limpeza futura, fora do escopo desta fase.
- **`noticias_vistas.json` é só uma lista plana de URLs** (dedup/"já visto" para o
  scraping), sem título nem metadados — não é um registro de conteúdo. 652 das 1669
  URLs (39%) não aparecem em `agendamentos.json` — pode ser notícia vista mas não
  selecionada para design, ou anterior à existência do campo `noticia_url`. **Decisão
  registrada**: não sintetizar linhas de `content_items` sem título a partir desse
  arquivo (geraria cards vazios num Kanban futuro). Se uma sessão futura confirmar a
  semântica exata de `monitor_noticias.py` para este arquivo, revisar essa decisão.

### 4.2 Decisão de arquitetura: sincronização incremental em vez de dual-write (2026-08-14)

A Fase 0 originalmente descrita neste documento (seção 6) previa **dual-write**:
instrumentar `orquestrador.py`, `telegram_bot.py`, `reel_handler.py`,
`publicar_instagram.py` e `monitor_noticias.py` para escreverem no `msconecta.db`
toda vez que já escrevem nos JSONs de produção. **Essa abordagem foi descartada em
favor de um job de sincronização incremental separado** (`sync_pipeline_db.py`),
pelos seguintes motivos:

- **Raio de explosão menor.** Dual-write significa tocar 5+ scripts que já rodam em
  produção (alguns via cron a cada minuto — `publicar_instagram.py`), cada um um
  ponto novo de risco de regressão no fluxo real de publicação. Um job de
  sincronização separado, rodando por fora, tem exatamente **um** ponto de falha
  possível — o próprio job — e uma falha nele não pode, por construção, impedir uma
  notícia ou Reel de ser publicado (ele só lê os mesmos JSONs depois do fato).
- **Nenhum script de produção precisa saber que o banco existe.** Reforça o
  princípio já registrado na seção 3 ("não quebrar o que está em produção") de forma
  mais literal do que o dual-write permitiria — não é só "sem mudança de
  comportamento visível", é "zero linhas alteradas" nesses arquivos.
- **Mesmo padrão de risco já mapeado na seção 6 do `CONTEXTO_MSCONECTA.md`**
  (falhas silenciosas em threads/timeouts não tratados) não se aplica a um job que
  roda isolado, com seu próprio lock, log estruturado e alerta — ao contrário de
  inserir escrita adicional dentro de um fluxo que já tem 3 incidentes documentados
  de falha silenciosa na mesma semana (11 e 13/08/2026).
- **Trade-off aceito**: o banco fica sempre alguns minutos "atrás" da produção (o
  intervalo do cron do sync), em vez de instantâneo como o dual-write daria. Aceitável
  para o caso de uso (dashboard de visão geral, não um sistema transacional).

**Implementação**: `sync_pipeline_db.py`, na raiz de `/root/msconecta/`, lê os mesmos
JSONs que `migrar_dados_iniciais.py` já lê (exceto `saude_estado.json`,
`design_queue.json` e `noticias_vistas.json` — fontes mortas ou já descartadas na
Fase 0, ver seção 4.1) e faz **UPSERT** (`INSERT ... ON CONFLICT ... DO UPDATE`,
suportado desde SQLite 3.24) em vez de recriar o banco. Chave natural: `pasta` em
`content_items`; `video_path` (ou `fonte_url`, para candidatos de Reel ainda sem
vídeo renderizado) em `reels`. `pipeline_events` usa duas **UNIQUE INDEX parciais**
— `(content_item_id, estagio_novo, timestamp) WHERE content_item_id IS NOT NULL` e o
equivalente para `reel_id` — em vez de uma única `UNIQUE` composta na tabela: **bug
descoberto e corrigido durante a implementação** — em SQL, `NULL` nunca é considerado
igual a outro `NULL`, então uma `UNIQUE(content_item_id, reel_id, estagio_novo,
timestamp)` comum nunca detecta conflito nesta tabela (`reel_id` é sempre `NULL` nos
eventos de notícia, e vice-versa) — confirmado com um teste isolado antes de aplicar
a correção definitiva (duas `CREATE UNIQUE INDEX ... WHERE ...`).

Uma transação por execução (`BEGIN`/`COMMIT`/`ROLLBACK`), lock via `fcntl.flock` não
bloqueante em `/tmp/sync_pipeline_db.lock` (liberado automaticamente pelo kernel
mesmo se o processo morrer — sem risco de lockfile "preso" por acidente), log em
`logs/sync_pipeline_db.log`, alerta no Telegram em falha (token/chat id lidos de
`/etc/msconecta-bot.env`, nunca hardcoded — mesmo padrão de `monitor_saude.py`) e
também se o lock ficar preso por 3 execuções seguidas (sinal de travamento, não só
uma sobreposição pontual). Modo `--dry-run` roda a sincronização inteira dentro da
transação e sempre dá `ROLLBACK` no final, nunca persiste.

**Status: implementado e validado em `--dry-run`, aguardando aprovação de Saulo para
entrar no cron.** Ver `HISTORICO_MUDANCAS.md`, entrada de 2026-08-14, para o
resultado dos testes (idempotência confirmada em execuções repetidas, detecção de
mudança incremental confirmada com um item sintético temporário, caminho de
persistência real também testado e revertido sem deixar resíduo).

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
**Entrega:** schema SQLite criado, script de migração dos JSONs existentes, banco
mantido atualizado por um job de sincronização incremental (ver decisão de
arquitetura na seção 4.2 — substitui o dual-write espalhado pelos scripts de
produção originalmente previsto aqui).
**Critério de conclusão:** o banco reflete fielmente o estado atual do sistema, sem
nenhuma mudança de comportamento visível no Telegram.
**Risco:** baixo.

**Status: praticamente concluída em 2026-08-14, aguardando aprovação final de
Saulo.** Executado:
1. Schema (`/root/msconecta/pipeline_schema.sql`), script de migração inicial única
   (`/root/msconecta/migrar_dados_iniciais.py`) e script de verificação
   (`/root/msconecta/verificar_migracao.py`) — banco em `/root/msconecta/msconecta.db`
   populado a partir dos JSONs de produção + `estado.json` + `saude_estado.json`, com
   100% de correspondência na amostra de verificação. Ver seção 4.1 para os ajustes
   de schema descobertos ao confrontar com os dados reais.
2. Lógica de parsing compartilhada extraída para `/root/msconecta/pipeline_lib.py`
   (usada tanto pela migração quanto pela sincronização, evitando duplicação).
3. Job de sincronização incremental (`/root/msconecta/sync_pipeline_db.py`) — ver
   seção 4.2 para a decisão de arquitetura e detalhes de implementação. Testado com
   sucesso em `--dry-run` (idempotência e detecção de mudança incremental
   confirmadas) e com uma execução real controlada (persistência confirmada, sem
   deixar resíduo). **Ativado no cron em 2026-08-14** (`*/5 * * * *`), aprovado por
   Saulo — 2 execuções reais confirmadas sem erro e sem alerta de lock indevido (ver
   `HISTORICO_MUDANCAS.md`).

**Fase 0 concluída em 2026-08-14.**

**Nenhum script de produção foi tocado** — os JSONs continuam sendo a única fonte de
verdade em produção, e nenhum deles (`orquestrador.py`, `telegram_bot.py`,
`reel_handler.py`, `publicar_instagram.py`, `monitor_noticias.py`) precisa saber que
o banco existe.

### Fase 1 — Dashboard somente leitura
**Entrega:** board (Kanban) por estágio, planner do dia (visão do que está agendado),
painel de saúde do sistema (lendo `system_health`), tudo em modo leitura.
**Critério de conclusão:** dá pra ver, sem usar o Telegram, tudo que está em produção
agora — quantas notícias em cada estágio, o que sai hoje, o que está saudável/quebrado.
**Risco:** baixo — pura visibilidade, nenhuma ação nova.

**Status: implementada em 2026-08-14, aguardando validação visual de Saulo.**
- **Health-check ativo** (`health_check_pipeline.py`) construído do zero — não usa
  `saude_estado.json` (morto desde 2026-06-15, ver seção 4.1). Checa identidade de
  processo: VPS via `systemctl` + `/proc/<pid>/cmdline` (msconecta-bot/api/editor/
  dashboard); PC Windows via assinatura de resposta HTTP sobre Tailscale
  (`story_listener_novo.ps1` porta 8765, servidor de render porta 3001), timeout de
  rede tratado como `desconhecido`, não `falha`. Roda manualmente por ora — cron
  sugerido ainda não aprovado (ver `HISTORICO_MUDANCAS.md`).
- **Dashboard** (`dashboard_pipeline.py`): FastAPI + htmx, sem build step de
  frontend (decisão confirmada — complexidade de SPA não se justifica antes da
  Fase 2). Conexão SQLite em modo `mode=ro` (somente leitura garantida a nível de
  driver, não só por convenção de código). Bind `127.0.0.1:8095` (porta livre
  confirmada antes de escolher — não colide com 8085/8090/3001/8765 nem com os
  outros serviços já rodando na VPS). Systemd `msconecta-pipeline-dashboard.service`.
  Exposto em `https://pipeline.korabot.com.br/pipeline` (TLS via Let's Encrypt,
  ver abaixo), protegido por autenticação básica (arquivo
  `/etc/nginx/.htpasswd_pipeline`, senha comunicada a Saulo via Telegram, nunca em
  texto plano em nenhum arquivo). **Desvio do plano**: optou-se por nginx basic
  auth em vez do padrão de token de `editor_tokens.py` — esse mecanismo foi
  desenhado para sessões de edição por-pasta-de-notícia com TTL de 4h (link
  efêmero enviado pelo bot), semântica que não encaixa numa ferramenta de
  navegação contínua por todo o pipeline; basic auth é a melhor opção mais simples.
- Validado localmente e via nginx (autenticação correta/incorreta, fragmentos htmx,
  estáticos) antes de considerar a fase pronta — ver `HISTORICO_MUDANCAS.md` para o
  detalhe dos testes.

**Endurecimento de segurança (2026-08-14, mesma sessão que revisou o resultado da
Fase 1) — duas pendências fechadas antes de seguir para qualquer coisa nova:**
- **TLS via subdomínio dedicado**: DNS de `pipeline.korabot.com.br` já apontado por
  Saulo para o IP da VPS antes de começar (confirmado via `dig`); criado
  `/etc/nginx/sites-available/pipeline` (novo `server_name`, mesma `location
  /pipeline`, mesmo `proxy_pass` para `127.0.0.1:8095`, mesma autenticação básica);
  certificado emitido via `certbot --nginx -d pipeline.korabot.com.br --redirect`
  (reaproveitou a conta Let's Encrypt já registrada nesta VPS para os demais
  domínios `*.korabot.com.br`); confirmado emissor `Let's Encrypt` e validade de 90
  dias via `openssl s_client`, e redirecionamento automático HTTP→HTTPS (301). A
  `location /pipeline` foi **removida** do bloco antigo (`msconecta-designs`,
  `server_name` = IP direto) — o acesso por IP não expõe mais o dashboard (cai na
  SPA que já ocupa a location `/` desse bloco). Renovação automática via
  `certbot.timer` (systemd) já estava habilitada por padrão nesta VPS para os
  demais certificados — só confirmado, nada novo configurado.
- **Rotação de senha**: a senha anterior (comunicada em texto plano numa sessão
  anterior) foi tratada como comprometida e rotacionada. **Bug real encontrado e
  corrigido durante a rotação**: a primeira tentativa gravou
  `/etc/nginx/.htpasswd_pipeline` com dono `root:root`, mas o worker do nginx roda
  como `www-data` — o arquivo ficou ilegível para o nginx (`open() ... Permission
  denied` no error log), quebrando a autenticação por completo (tanto a senha nova
  quanto a antiga passaram a retornar HTTP 500, não 401/200). Corrigido para
  `root:www-data` (modo 640); a senha rotacionada nessa primeira tentativa nunca
  chegou a funcionar de verdade, então uma segunda rotação foi feita — dessa vez
  validando com sucesso/senha-errada **antes** de notificar Saulo, para não repetir
  o erro de comunicar uma credencial não testada. Nova senha entregue via mensagem
  do bot do Telegram (mesmo canal já usado por todo o projeto para aprovações),
  nunca impressa nesta sessão nem em nenhum arquivo de documentação.

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
