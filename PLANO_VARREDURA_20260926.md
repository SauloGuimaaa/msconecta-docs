# PLANO_VARREDURA_20260926.md

> **Status: SOMENTE PLANO. Nada foi aplicado.** Varredura feita em 2026-09-26
> (noite), só leitura: código em `/root/msconecta`, `agendamentos.json`,
> `estado.json`, `msconecta.db` (modo `ro`), logs, crontab, arquivos de
> ambiente (**só nomes de variável, nunca valores**), portas/firewall.
> Nenhum arquivo de código, dado, cron, env ou serviço foi alterado. Nenhum
> teste foi executado (alguns testes antigos já tocaram Telegram real; rodar a
> suíte é o passo 1 do lote de commit, item C6).
>
> **Objetivo:** deixar a operação diária **simples e segura** o bastante para
> ser passada a outra pessoa, sem o contexto técnico desta semana. Prioridade
> é segurança e simplicidade, não funcionalidade nova.
>
> Aplicação: só depois de revisão de Saulo, lote por lote (seção 5).

**Severidade:** **CRÍTICO** = já causou (ou causa hoje) perda, publicação
indevida ou estado inconsistente, e um operador novo vai esbarrar nisso.
**IMPORTANTE** = risco real, sem incidente confirmado ainda, ou bloqueia o
handoff. **DESEJÁVEL** = limpeza/clareza.

---

## 1. Resumo executivo (5 coisas para ler primeiro)

1. **"Descartar" não cancela o que já está agendado.** `pipeline_acoes.descartar()`
   marca `cancelado` no banco e tira da fila do Telegram, mas **não mexe no
   `agendamentos.json`**, que é o que o cron lê para publicar. **4 notícias
   descartadas no dashboard em 25/09 foram publicadas mesmo assim** (C1).
   Essa mesma falha é a **causa raiz do "sync remarcando cancelados como
   agendado"** (item 6 do pedido).
2. **"pular design" / botão "Cancelar" = descarte permanente, sem confirmação,
   sem dizer o que foi descartado, e agindo sobre o item "atual" (não sobre a
   mensagem clicada).** O botão "Cancelar" responde "Publicação cancelada",
   mas executa exatamente o mesmo descarte. Sinônimos que o classificador de
   IA também transforma em descarte: "proximo", "proximo design", "skip" (C2, C3).
3. **O agendamento tem ~12 camadas; 4 bastam.** Como o gate de 25-30 min já é o
   limite físico real (~33/dia na janela 7h-22h), teto móvel, conferência de
   fantasmas, FIFO por permuta, horário preservado e o ramo especial de
   reposição podem sair do caminho de decisão (seção 3).
4. **Alertas mortos:** o comando `openclaw` (WhatsApp) **não existe neste
   servidor**. Todo aviso que só passa por ele é silencioso, incluindo
   **"publicação parcial: STORY falhou"** (I2). O monitor de notícias também
   não alerta quando o site fica ilegível ou quando o design falha (I3).
5. **Handoff:** não existe manual para operador. `CONTEXTO_MSCONECTA.md` tem
   115 KB e é técnico, `MANUAL_OPERACIONAL.md` é de maio, e a ajuda do bot nem
   menciona que "pular" apaga. Também não há como **ver ou cancelar um post já
   agendado** por nenhuma das duas interfaces (I7, I10).

---

## 2. Tabela de achados em ordem de prioridade

| # | Sev. | Área (pedido) | Achado | Esforço |
|---|---|---|---|---|
| C1 | CRÍTICO | 1, 6 | Descartar não remove/cancela a entrada pendente em `agendamentos.json` → item descartado é publicado; sync "ressuscita" o estágio | baixo |
| C2 | CRÍTICO | 1 | "pular design"/"Cancelar"/"proximo"/"skip" = descarte permanente sem confirmação nem eco do título | baixo |
| C3 | CRÍTICO | 1, 8 | Botões do Telegram agem sobre `estado['ultima_pasta']` (global), não sobre a mensagem clicada | médio |
| C4 | CRÍTICO | 6 | `sync_pipeline_db.py` sobrescreve o estágio do banco pelo JSON (inclusive estados só do banco) e grava eventos com timestamp = horário agendado (futuro) | baixo |
| C5 | CRÍTICO | 5 | `estado.json` escrito sem trava por `monitor_noticias.py` (cron 30 min) e 13 pontos do bot/reel → perda/ressurreição de itens da fila de aprovação | médio |
| C6 | CRÍTICO | 3 | ~1.900 linhas não commitadas em 15 arquivos + scripts de produção não rastreados (um está no cron) + dados de runtime rastreados no git | médio |
| I1 | IMPORTANTE | 2 | Agendamento com camadas redundantes; duas janelas diferentes (7-22 e 9-20); backoff menor que o gate | médio |
| I2 | IMPORTANTE | 5 | `openclaw` não instalado: 7 alertas WhatsApp mortos, incluindo o de STORY falho | baixo |
| I3 | IMPORTANTE | 5 | `monitor_noticias.py`: falha total de scraping e falha de design só vão para `print` (sem alerta, retentativa infinita) | baixo |
| I4 | IMPORTANTE | 5 | Callbacks do bot (`pub:agora`, `focal`, `pub_ig`, `pub:cancelar`) não avisam falha (mesmo bug de 16/08: condição `not stdout`) | baixo |
| I5 | IMPORTANTE | 4 | Segredos: token do bot em 15 arquivos rastreados no git; senha SSH em `publicar_via_pc.sh`; `.env` com permissão 644; duplicatas; rotação do Cloudinary pendente | médio |
| I6 | IMPORTANTE | 4 | Editor visual exposto em `0.0.0.0:8090` por HTTP puro, com token na URL; Bot API local publicado pelo Docker em `0.0.0.0:8081` | baixo |
| I7 | IMPORTANTE | 7, 8 | Não há como ver/cancelar/remarcar um post **já agendado** (nem Telegram, nem web) | médio |
| I8 | IMPORTANTE | 8 | Mesmo nome, efeito diferente entre Telegram e web ("só o feed", "Cancelar"/"Pular") | baixo |
| I9 | IMPORTANTE | 1 | "posta só o story" publica NA HORA, fora do gate e fora da fila; "aprovado **pode postar**"/"posta agora" agora só agendam | baixo |
| I10 | IMPORTANTE | 7 | Sem manual do operador; ajuda do bot desatualizada | médio (texto) |
| I11 | IMPORTANTE | 7 | Documentação divergente do estado real (restauração desfeita às 15:20 UTC não registrada; `ROADMAP_OPERACAO_SIMPLES.md` não existe) | baixo |
| I12 | IMPORTANTE | 1 | Sem item pendente, aprovar/story caem em `encontrar_pasta_recente()` → age sobre a pasta mais recente do disco | baixo |
| D1 | DESEJÁVEL | 5 | Story "sucesso" = PC aceitou o pedido (`ok:true`), não que publicou | médio |
| D2 | DESEJÁVEL | 5 | `aprovar_e_postar_agora()` diz "Threads e Facebook" fixo, mesmo se falharam | trivial |
| D3 | DESEJÁVEL | 4 | Dezenas de `.bak*` com segredos; `spconecta/.env` com `FACEBOOK_SYSTEM_TOKEN` morto; `StrictHostKeyChecking=no` | baixo |
| D4 | DESEJÁVEL | 3 | Funky Fresh (outro cliente) no mesmo repositório e crontab | médio |

---

## 3. Detalhe dos achados

### C1 — CRÍTICO — Descartar não cancela o agendamento (e é a causa raiz do item 6)

**Evidência**
- `pipeline_acoes.descartar()` (linha ~500) só faz `_avancar_estado_apos_acao()`
  (estado.json) + `_marcar_cancelado_no_banco()`. **Nenhuma escrita em
  `agendamentos.json`.** O cron (`processar_agendamentos()`) só lê
  `agendamentos.json`.
- Conferido no banco + `instagram.log.1`: 4 itens com último evento explícito
  `cancelado` (dashboard) que foram publicados depois pelo cron:

  | Notícia | Descartado (UTC) | Publicado pelo cron (UTC) |
  |---|---|---|
  | Itaporã inicia obra de R$ 1,9 mi | 25/09 15:00:29 | 25/09 15:10:01 |
  | Impressão 3D ganha espaço em CG | 25/09 15:01:12 | 25/09 15:20:02 |
  | ALEMS aprova lei contra idadismo | 25/09 15:17:50 | 25/09 16:10:02 |
  | Senai Corumbá recebe 300 inscritos | 25/09 16:53:51 | 25/09 21:10:01 |

- Depois disso, `sync_pipeline_db.py` reescreve o estágio no banco para
  `agendado`/`publicado_completo` a partir do JSON (ver C4). É isso que aparece
  como "sync remarcando cancelado como agendado".
- Variante do mesmo problema: 8 itens marcados `publicado_parcial` por ação
  explícita estão como `erro` no banco, porque a última entrada deles no JSON é
  um `erro` antigo.

**Risco:** o operador descarta uma notícia (errada, vencida, sensível) e ela sai
mesmo assim. Nada avisa.

**Correção proposta**
1. `descartar(pasta)`: sob `_lock_agendamentos_io()`, marcar **toda** entrada
   `pendente` da pasta em `agendamentos.json` como `status="cancelado"`, com
   `cancelado_em` e `motivo`. Não apagar a entrada (rastreabilidade).
2. A mensagem de retorno diz o que aconteceu de fato: "Descartado: <título>
   (também removido da fila de publicação das HH:MM)".
3. Teste: descartar um item `pendente` → o cron não o publica, e o sync o mantém
   `cancelado`.

**Verificação:** repetir a consulta desta varredura ("último evento explícito ≠
estágio atual") e esperar zero casos `cancelado → publicado`.

---

### C2 — CRÍTICO — "pular design" e "Cancelar" são descarte permanente sem confirmação

**Evidência**
- `orquestrador.py:_COMANDOS_DIRETOS`: `'pular'`, `'pular design'` → `acao: pular`.
  O prompt do classificador (Haiku) também mapeia **"proximo", "proximo
  design", "skip"** para `pular`.
- `acao == 'pular'` → `pipeline_acoes.descartar(pasta)`. O retorno
  (`ResultadoAcao`) é **ignorado**: o operador não recebe "descartado: X", só
  vê o próximo item aparecer.
- A mensagem de instrução ao operador diz `pular design - proximo da fila`, o
  que parece "passar para o próximo", não "apagar este".
- Botão **"Cancelar"** (`pub:cancelar`, `telegram_bot.py` ~linha 789) responde
  *"Publicação cancelada. Verificando próxima da fila..."* e roda
  `orquestrador.py 'pular design'`, ou seja, **o mesmo descarte**.
- Cada mensagem de texto vira uma thread + subprocesso em paralelo
  (`telegram_bot.py:219`). Dez "pular" digitados em sequência rápida
  (09:23:34-09:25:09 UTC de hoje) viram dez descartes independentes, sem nenhum
  freio.
- Hoje não existe "desfazer". A restauração foi feita à mão no JSON e no banco.

**Correção proposta (sem funcionalidade nova além do mínimo)**
1. **Renomear:** botão e comando passam a ser **"🗑 Descartar notícia"**.
   "pular"/"proximo"/"skip" deixam de descartar. Ou passam a fazer **"deixar
   para depois"**, que move o item para o fim da fila sem descartar (decisão
   de Saulo, ver seção 6).
2. **Confirmação em 2 passos:** "Descartar *<título>*? Ela NÃO será publicada.
   [Sim, descartar] [Não]". O botão de confirmação carrega a pasta (ver C3).
3. **Eco do resultado:** "🗑 Descartada: <título>. Para desfazer em até 24h:
   *desfazer descarte*".
4. **`desfazer descarte`:** recoloca o último item descartado na frente da fila
   de aprovação (sem publicar, sem aprovar).
5. Remover `skip`/`proximo` do prompt do classificador.
6. Mesma confirmação no dashboard ("Pular / descartar" → "Descartar",
   `hx-confirm`).

---

### C3 — CRÍTICO — Botões agem sobre o item "atual", não sobre a mensagem clicada

**Evidência**
- `pub:agora`, `pub:programar`, `pub:cancelar`, `repos:menu`, `editor` são
  literais fixos, sem `msg_id`. Todos resolvem a pasta por
  `estado['ultima_pasta']` (`orquestrador.acao_publicar_feed/acao_programar`,
  bloco `pular`, `telegram_bot` `repos`/`editor`).
- Já documentado como causa da notícia "XP Investimentos" perdida (17/09) e
  como "Cuidado conhecido" na entrada de hoje ("botões de mensagens antigas agem
  sobre a nova cabeça"). Com uma fila de 23 itens, clicar num botão de uma
  mensagem de 2 telas acima aprova, reposiciona ou **descarta outra notícia**.
- `imgia:*` e `undo:*` já resolvem a pasta por `msg_id`
  (`image_approval_utils.registrar_msg_aprovacao()`), então o mecanismo já existe.

**Correção proposta**
- Incluir o `msg_id` no `callback_data` de todos os botões de aprovação
  (`pub:agora:{msg_id}` etc.) e resolver a pasta pelo mapa já existente.
- Se a pasta daquela mensagem **não** for mais a pendente (já aprovada ou
  descartada), responder "Esta mensagem é antiga: <título> já foi <estado>" e
  não agir.
- Comandos por texto ("aprovado") continuam valendo para o item atual, mas a
  resposta sempre cita o título afetado.

---

### C4 — CRÍTICO — Sync: o JSON sobrescreve o banco e os eventos têm data futura (item 6)

**Evidência**
- `SQL_UPSERT_CONTENT_ITEM` faz `estagio=excluded.estagio` sem condição. A
  entrada "canônica" é a **última da pasta na ordem do arquivo**
  (`agrupar_agendamentos`, que supõe o arquivo append-only). Edições manuais de
  hoje (restaurações, reset FIFO, fantasmas) quebram essa suposição.
- Estados que só existem no banco (`cancelado` via descartar, `aguardando_aprovacao`
  via reset, `publicado_*` via `pipeline_acoes`) são desfeitos no ciclo seguinte
  quando a pasta tem **qualquer** entrada no JSON.
- Eventos `sync` usam `ts = publicado_em or horario`, ou seja, **o horário
  agendado (futuro)**, e `estagio_anterior` vazio. Exemplo real (Dourados, curso
  de português): evento `id 20745528`, `→ agendado`, timestamp `26/09 23:53`,
  gravado **antes** do evento `id 20745656` (15:20 UTC, "restauração
  desfeita"). Ordenado por timestamp, parece que o sync "remarcou como
  agendado" depois do cancelamento. Ordenado por `id`, não remarcou.
  O `HISTORICO` de hoje já tinha notado esse efeito em outros itens.
- Estado atual conferido: os 5 itens restaurados e depois "desfeitos" estão
  **consistentes** (`cancelado` no banco, sem entrada pendente no JSON). A
  anomalia observada hoje tem duas causas: (a) C1, quando havia entrada
  pendente, e (b) timestamp futuro, quando não havia.

**Correção proposta** (depois de C1, que remove a causa principal)
1. Timestamp do evento de sync = momento da sincronização (`now`). O horário
   agendado continua em `agendado_para`.
2. O sync não rebaixa um estado terminal explícito (`cancelado`) enquanto a
   entrada do JSON não for mais nova que o evento explícito. Depois de C1 isso
   vira só rede de segurança.
3. Parar de editar `agendamentos.json` à mão fora do lock/das funções. Ferramenta
   única de manutenção, ver I7.

---

### C5 — CRÍTICO — `estado.json` escrito sem trava por vários processos

**Evidência**
- `estado_lock()` (fcntl) só é usado em `orquestrador.py` e `pipeline_acoes.py`.
- `monitor_noticias.py` (cron a cada 30 min) faz ler-modificar-gravar de
  `fila_aprovacao` **sem trava** (linhas ~248-268).
- `telegram_bot.py`/`reel_handler.py`/`editor_visual.py`/`image_approval_utils.py`:
  13 gravações diretas (`ap`, `ap_lote`, `pub:programar`, `resetar reel`,
  `_atualizar_estado` do editor...).
- Cenário: o operador descarta ou aprova (pop da fila) enquanto o monitor está
  gravando → o monitor regrava a fila antiga → o item descartado **volta** à
  fila (e ao dashboard como `aguardando_aprovacao`), ou uma notícia nova some.
  Não há incidente confirmado ainda, mas a janela abre a cada 30 min, justo no
  horário de pico de aprovação.

**Correção proposta:** um único helper `estado_atualizar(fn)` (lock + ler +
aplicar + gravar atômico via arquivo temporário + `rename`) usado por todos os
escritores. Começar por `monitor_noticias.py` e `pub:programar`.

---

### C6 — CRÍTICO (bloqueia o handoff) — Código não commitado

**Estado atual** (`git status` em `/root/msconecta`, último commit `11e7386`)
- 15 arquivos modificados, +1.906/−243 linhas. `publicar_instagram.py` sozinho
  tem +1.270 linhas em 44 blocos (HEAD tem 36 KB, disco tem 98 KB). Tudo de
  25-26/09: lock de `agendamentos.json`, rate limit/backoff, cota real, teto
  diário/móvel, auto-slot, horário preservado, idempotência por canal, ordem
  de visita, conferência com a API, relatório diário.
- Também modificados: `pipeline_acoes.py` (+306), `sync_pipeline_db.py` (+112,
  WAL/retry), `orquestrador.py`, `telegram_bot.py`, `dashboard_pipeline.py`,
  `editor_visual.py` (correção da pasta no reposicionamento em outro dia),
  `publicar_reel.py`, `renovar_token.py`, `metricas_instagram.py`,
  `post_story_agora.py`, `pipeline_lib.py`.
- **Não rastreados, mas em produção:** `renovar_token_meta.py` (**está no
  crontab**), `renovar_token_threads.py`, `atualizar_credencial_env.py`,
  `diagnostico_ig/` (no crontab), `gerenciar_fila_repostagem.py`,
  `soltar_fila_repostagem.py`, 9 arquivos `test_*.py`.
- **Dados de runtime rastreados no git** (geram "modificado" eterno):
  `historico_publicacoes.json`, `cardapio_funky_fresh.json`,
  `funky_fresh_perfil_completo.json`.
- **Incompleto/descartável:** `slot_simples.py` (só dry-run, não ligado);
  `horarios_preservados_aprovacao.json` (0 entradas, mecanismo de uso único);
  `fila_repostagem.json` (10 descartados + 1 `pendente_validacao`);
  `listar_falsos_publicados_rate_limit.py`, `auditar_site_vs_instagram.py`,
  `REVISAO_FASE2_20260925.txt`, `relatorio_gaps_hoje.csv`, `preview_lote/`,
  `reset_fifo_*.json`, `reposicoes_removidas_*.json` (artefatos de diagnóstico).
- Existem **snapshots cronológicos** de `publicar_instagram.py` de hoje
  (`backups/…bak_20260926_idempotencia` 07:52 → `…regras_v2` 07:54 →
  `…ordem_fila` 08:02 → `…conferencia_api` 08:29 → `…pareamento_legenda` 08:38
  → `…intervalo_unificado` 08:45 → disco). Com eles dá para reconstruir parte
  da sequência.

**Plano seguro (nenhum passo muda produção)**
0. **Congelar:** nenhuma outra sessão edita `/root/msconecta` durante o lote.
   `tar` do diretório (sem `output/`) para `backups/`.
1. **Rodar a suíte numa cópia isolada** (`cp -a` para um diretório temporário,
   `TELEGRAM_BOT_TOKEN`/`INSTAGRAM_TOKEN` vazios no ambiente, rede de saída
   bloqueada para `api.telegram.org`/`graph.*`). Registrar o resultado. Teste
   que falhar vira achado, não é "consertado" no mesmo lote.
2. **`.gitignore`** para runtime: `estado*.json`, `cache_*.json`,
   `historico_publicacoes.json`, `agendamentos.json*`, `*.db-wal`, `*.db-shm`,
   `preview_lote/`, `reset_fifo_*`, `reposicoes_*`, `estado_token_*.json`,
   `fila_repostagem.json`, `horarios_preservados_*`. Depois
   `git rm --cached` dos 3 JSON rastreados (o arquivo fica no disco).
3. **Commits por tema, sem reescrever código:**
   - `publicar_instagram.py`: se o diff entre snapshots consecutivos for de um
     tema só, 1 commit por snapshot (HEAD→idempotencia = "25/09: rate limit,
     cota, teto, auto-slot, lock"; →regras_v2 = idempotência; ... →disco =
     intervalo unificado). Se algum passo misturar temas (houve duas sessões
     editando em paralelo, ver HISTORICO), aceitar **um commit-retrato**
     "estado em produção 26/09", com a lista de funcionalidades na mensagem.
     Separar perfeitamente não compensa: boa parte desse código sai no
     item I1.
   - `pipeline_acoes.py` + `orquestrador.py` + `telegram_bot.py` +
     `dashboard_pipeline.py`: "auto-slot no Telegram, Fase 2, bug de status".
   - `sync_pipeline_db.py` + `pipeline_lib.py`: "WAL + retry".
   - `editor_visual.py`: "pasta preservada no reposicionamento".
   - Tokens: `renovar_token*.py`, `atualizar_credencial_env.py`,
     `metricas_instagram.py`, `post_story_agora.py` + testes.
   - `diagnostico_ig/` (sem dados; conferir antes se não há token em arquivo).
4. **Ferramentas pontuais** (`soltar_/gerenciar_fila_repostagem`, `listar_falsos_*`,
   `auditar_*`, `slot_simples.py`) vão para `ferramentas/` num commit próprio
   ou são arquivadas. Nada de uso único na raiz.
5. **Tag** `producao-20260926` no último commit. É o ponto de retorno antes de
   qualquer simplificação.
6. **Regra nova em `CONVENCOES.md`:** toda sessão termina com commit do código
   que tocou (não só dos docs); editar arquivo executado pelo cron sempre numa
   cópia + `py_compile` + `mv` atômico (já é a prática, mas não está escrito).
7. **Atenção a segredos no commit:** 15 arquivos já rastreados contêm o token do
   bot (ver I5). O repositório **não tem remote**, então não há exposição
   externa, mas **não adicionar remote** antes de I5.

---

### I1 — IMPORTANTE — Agendamento: o que é essencial e o que é complexidade acumulada (item 2)

**Camadas hoje** (`publicar_instagram.py` + `auto_slot_regras.py`, flag
`AUTO_SLOT_REGRAS_V2=true` no `.env`)

| # | Camada | Onde age | Veredito | Por quê |
|---|---|---|---|---|
| 1 | **Gate de espaçamento 25-30 min** (`espacamento_liberado`) | cron | **MANTER (essencial)** | É o limitador físico real (~52/24h) e foi o que parou os rate limits de 25/09 |
| 2 | **Tratamento de rate limit** (code 9/2207042 → continua `pendente`, conta tentativa, desiste em 4 com alerta) | cron | **MANTER, simplificado** | Essencial contra erro de API. Mas o backoff de 10/20/40/80 min + `proximo_slot_livre` (janela 9-20h) é redundante: 10-20 min é **menor que o gate**, então não faz nada. Basta "tenta no próximo slot do gate" |
| 3 | **Idempotência por canal** (`story_ok/threads_ok/facebook_ok`) | cron | **MANTER (essencial)** | Impede duplicar Story/Threads/Facebook em retentativa (caso santa-emilia) |
| 4 | **"publicado" só com feed confirmado** (`ResultadoPublicacao`) | cron | **MANTER (essencial)** | Foi a origem dos fantasmas |
| 5 | Cota real `content_publishing_limit` a 90% | cron **e** auto-slot | **MANTER só no cron** | É barata (1 GET por ciclo) e protege contra o limite oficial se houver posts manuais/Reels. No cálculo do horário, só complica |
| 6 | Ordem de visita por (tentativas, horário) | cron | **MANTER** | Pequena, evita monopólio de slot |
| 7 | Janela 7h-22h CG (auto-slot) **e** janela 9h-20h (reagendamentos) | ambos | **UNIFICAR em uma** | Duas janelas diferentes confundem ("por que foi para 9h e não 7h?") |
| 8 | Teto diário (calendário) / **teto móvel 60/24h** + `_reconciliar_teto_movel` a cada ciclo | cron + auto-slot | **REMOVER da decisão** | O gate satura antes (~52/24h < 60). Hoje o teto só criou problemas: saltos de horário e contaminação por fantasmas. Pode virar **métrica** no relatório diário |
| 9 | Conferência local × API de "fantasmas" (`_publicados_nao_confirmados`, pareamento por legenda, cache) | cron + auto-slot | **REMOVER** (sai junto com o 8) | Só existe para corrigir a contagem do teto. Sem teto, não tem função. Os fantasmas já foram corrigidos na origem (camada 4) |
| 10 | FIFO por permuta de horários (`plano_fifo`) | auto-slot | **REMOVER** | Causou o churn de horários de hoje (Coxim/Esquema/Governo recebendo os slots de outros itens, salto 14:35→16:27). FIFO vem de graça se a **fila de aprovação** já estiver em ordem de geração e a publicação seguir a ordem de aprovação |
| 11 | Horário preservado (`candidato_base`, `horarios_preservados_aprovacao.json`) | auto-slot | **REMOVER** | Mecanismo de uso único (desfazer o lote de 25/09). Arquivo hoje com 0 entradas |
| 12 | Alerta ">24h desde a geração" | cron | **MANTER como aviso, fora do agendamento** | Só avisa. Mover para o relatório diário ou 1x/dia, não por ciclo |
| 13 | Ramo especial de reposição (`reposicao`, `REPOSICAO_IGNORA_JANELA`, `fila_repostagem.json`) | cron | **REMOVER depois de conferir que a fila está vazia** | 10 descartados + 1 `pendente_validacao`. O ramo especial foi fonte de 2 bugs hoje |

**Versão mais simples ainda segura contra erro de API (proposta)**

```
Aprovar (auto):
  horario = max(último pendente, agora) + 27 min
  se horario cair fora de 07:00–22:00 CG → próximo dia 07:00 (+27 min em fila)
  grava 1 entrada pendente (idempotente por pasta)

Programar HH:MM (manual): grava no horário pedido. O gate pode segurar
  até ~30 min; a resposta ao operador avisa isso.

Cron (a cada 1 min), itens pendentes vencidos em ordem de horário:
  1. gate 25-30 min fechado → espera (não mexe no item)
  2. cota oficial ≥ 90% → pausa o ciclo, alerta 1x/dia
  3. publica (pulando canais já publicados)
  4. feed confirmado → publicado | rate limit → tentativas+1, continua pendente
     (sai no próximo slot do gate); na 4ª → erro + alerta Telegram
Relatório diário: publicados/24h, itens >24h, erros.
```

- Proteções mantidas contra erro de API: gate, cota oficial, tratamento de
  rate limit com limite de tentativas, idempotência por canal, status só com
  feed confirmado.
- Removido do caminho de decisão: teto móvel/diário, conferência com a API,
  FIFO por permuta, horário preservado, janela dupla, backoff próprio, ramo de
  reposição.
- Ordem de aplicação sugerida: (a) commit + tag (C6); (b) desligar
  `AUTO_SLOT_REGRAS_V2` (1 linha no `.env`, reversível) e observar 1 dia; (c)
  trocar `calcular_slot_auto_aprovacao()` pela fórmula acima (o `slot_simples.py`
  já tem a função pura, falta a janela); (d) só depois remover o código morto.
  Cada passo com a simulação de cron em cópia da fila, que é obrigatória desde
  hoje.
- Decisão em aberto (Saulo): manter a janela 7h-22h? Recomendação: **sim**, porque
  sem ela a fila atrasada cai de madrugada. É a única regra além do intervalo.

---

### I2 — IMPORTANTE — `openclaw` não existe: alertas de WhatsApp são silenciosos

**Evidência:** `which openclaw` → vazio. `publicar_instagram.py` chama
`openclaw message send` em 7 pontos, todos com `except Exception: pass`. O único
aviso de **"publicação parcial" (feed saiu e STORY falhou, ou o contrário)**
(~linha 1175) usa só `openclaw`. Feed, Threads, Facebook e Reel já têm
`_alertar_telegram`, mas o **Story não tem**. O HISTORICO de 24/09 já dizia "não
funciona no cron". Na prática não funciona em lugar nenhum.

**Correção:** trocar o alerta de publicação parcial por `_alertar_telegram`. Os
outros 6 `openclaw` ("✅ publicado", "agendado") são ruído: remover.

---

### I3 — IMPORTANTE — Monitor de notícias falha sem avisar

**Evidência** (`monitor_noticias.py`)
- `main()`: `except Exception as e: print(f"Erro no scrape: {e}"); return`.
  Nenhum alerta. Com o site ilegível (PC fora + proxies bloqueados, já aconteceu
  em 16/08 e 01/09), o sistema fica cego por horas.
- `gerar_design()`: se não houver `DESIGNS_PRONTOS`, der timeout ou exceção,
  só `print`. A notícia não é marcada como vista e volta a ser tentada **a cada
  30 min, para sempre**, sem contador (só o caso "saída sem OUTPUT_PATH" alerta).
- As perdas estruturais da Fase 1 do `ROADMAP_ALTO_VOLUME.md` (página 1, 5 por
  ciclo, mais nova primeiro) continuam sem alerta.

**Correção:** contador persistido por URL e por "falha de scrape consecutiva".
Alerta Telegram na 3ª falha seguida de scraping (~1h30) e na 3ª falha de design
da mesma notícia, com o link. Mesmo padrão "alerta só na mudança de estado" do
`health_check_pipeline.py`.

---

### I4 — IMPORTANTE — Callbacks do bot escondem falhas

**Evidência** (`telegram_bot.py`)
- `pub:agora` (o botão de aprovar!) e `focal`: `if r.returncode != 0 and not
  r.stdout.strip()`. O orquestrador sempre escreve log em stdout, então a
  condição quase nunca é verdadeira. É o mesmo bug corrigido em `_worker_texto`
  em 16/08.
- `pub:cancelar` e `pub_ig`: nenhuma checagem de retorno, nenhuma mensagem de erro.
- `subprocess.run(... timeout=...)` sem `try/except TimeoutExpired` dentro de
  thread daemon (padrão nº 3 da lista de falhas silenciosas).

**Correção:** um helper único `_rodar_orquestrador(cmd, chat_id)` com
try/except, timeout tratado e aviso em qualquer `returncode != 0`.

---

### I5 — IMPORTANTE — Segredos pendentes (item 4)

Tudo verificado **só por nome/caminho**, nenhum valor exibido.

| Segredo | Onde | Risco | Correção |
|---|---|---|---|
| Token do bot Telegram | Literal em **15 arquivos rastreados no git** (`telegram_bot.py`, `orquestrador.py`, `monitor_noticias.py`, `editor_visual.py`, `image_approval_utils.py`, `portal_config.py`, `pipeline.py`, `notificar_pautas.py`, `relatorio_diario/semanal.py`, `reel_handler.py`, `monitor_videos.py`, `batch_12_aprovacao.py`, `journalist/redator.py`, `journalist/monitor_bloco1.py`) + 9 cópias em `arquivo/backup_20260610/` também rastreadas + histórico do git | Quem lê o repo/backup controla o bot (manda mensagens em nome do MSConecta, lê a conversa). O repo não tem remote hoje, mas um `git push` futuro publicaria tudo | (1) Todos passam a ler `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` do ambiente (`/etc/msconecta-bot.env`, que já tem) via `portal_config`; (2) **revogar o token no @BotFather** e gravar o novo só no env (o histórico do git fica inerte, sem reescrever); (3) nunca adicionar remote antes disso |
| Senha SSH do PC Windows | `publicar_via_pc.sh`, 6 linhas `sshpass -p` (rastreado no git), com `StrictHostKeyChecking=no` | Senha de login do PC servidor em texto plano | Chave SSH (ed25519) só para esse uso, `authorized_keys` no Windows, remover `sshpass`. Trocar a senha do usuário Windows depois. Se o script não é mais usado (é plano B, e o SSH estava inacessível em 01/09), **aposentar** |
| `.env` do projeto | `/root/msconecta/.env` com permissão **644** (qualquer usuário local lê). Há outros usuários na VPS (`ubuntu`, `claude-runner`) | Tokens do Instagram, Threads, Facebook, Cloudinary legíveis por outros usuários | `chmod 600` (o cron roda como root). Idem `/root/spconecta/.env` |
| Duplicatas | `CLOUDINARY_API_KEY/SECRET` em 5 arquivos (`.env`, `bot.env`, `/etc/environment`, `/etc/profile.d/msconecta.sh`, `spconecta/.env`); `INSTAGRAM_APP_SECRET` em 5 | Mesma confusão de "cópia velha" do Threads de hoje; `/etc/environment` e `profile.d` são legíveis por todos | Fonte única: `.env` (publicador) + `/etc/msconecta-bot.env` (serviços). **Remover** de `/etc/environment` e `/etc/profile.d` depois de auditar consumidores (mesmo procedimento de hoje para os tokens) |
| Cloudinary (exposto hoje) | ativo e igual ao de produção | Exposição real (impresso num `grep`) | **Ação de Saulo:** rotacionar no painel e rodar `atualizar_credencial_env.py` (pronto) |
| `FACEBOOK_SYSTEM_TOKEN` morto | `spconecta/.env` (usado como principal lá) | Não é do MSConecta, mas o SPConecta publica com token morto | Decidir fora deste plano |
| Backups com segredos | dezenas de `*.bak*` em `/etc`, `backups/`, `.env.bak_*` | Higiene | Depois da rotação: apagar ou mover para diretório 700 |

---

### I6 — IMPORTANTE — Portas expostas

**Evidência**
- `editor_visual.py` escuta em `0.0.0.0:8090`, e o `ufw` libera 8090 para
  qualquer origem. O bot envia o link `http://72.60.151.58:8090/editor?token=...`,
  com **token em URL sem TLS** (válido por 4h, não é de uso único). O
  `CONTEXTO` diz que o editor também é servido por nginx em `https://<host>/editor`.
- `docker-proxy` publica o Bot API local em `0.0.0.0:8081` (e `[::]:8081`).
  Portas publicadas pelo Docker normalmente **passam por fora do ufw**.
  **Verificar** de fora antes de concluir. Exploração exige o token do bot, mas
  o modo `--local` expõe arquivos baixados.

**Correção:** editor em `127.0.0.1:8090` + link HTTPS via nginx + fechar 8090 no
ufw. Bot API com `127.0.0.1:8081:8081` no `docker-compose`.

---

### I7 — IMPORTANTE — Não dá para ver nem cancelar um post já agendado

**Evidência:** Telegram `fila` lista só `fila_aprovacao` (aguardando). O
dashboard `/pipeline/revisao` age só sobre `aguardando_aprovacao`, e o Planner é
só leitura. Para desmarcar um post aprovado por engano hoje é preciso editar
`agendamentos.json` à mão, que é justamente a origem de metade dos problemas
de hoje.

**Correção (mínima, para o operador):**
- Telegram: comando `agendados` (lista numerada com horário CG e título) e
  `cancelar agendado N` (com confirmação, via a mesma função de C1).
- Dashboard: botão "Cancelar agendamento" no Planner (mesma função, com
  `hx-confirm`).

---

### I8 — IMPORTANTE — Telegram × Web: consistência (item 8)

| Ação | Telegram | Dashboard | Problema |
|---|---|---|---|
| Aprovar, sistema escolhe horário | "aprovado pode postar" / botão "✅ Aprovado (agenda automática)" | "Aprovar (horário automático)" | OK. Mas o texto "**pode postar**" sugere publicar agora |
| Aprovar e publicar já | **não existe** ("posta agora" e "publica" hoje agendam) | "Aprovar e postar agora" | Só a web tem publicação imediata (é a ação mais arriscada para rate limit) |
| Só o feed | "so o feed" → **agenda** (auto-slot) | "Publicar só feed" → **publica na hora** | **Mesmo nome, efeito diferente** |
| Só o story | "posta so o story" → **publica na hora**, fora do gate e da fila (I9) | não existe | Ação imediata escondida só no Telegram |
| Horário fixo | "programa para as Xh" / botão "Programar horário" | "Aprovar e agendar" + campo | OK |
| Descartar | "pular design", "proximo", "skip", botão "Cancelar" | "Pular / descartar" | 5 nomes, nenhum diz "apagar"; nenhum confirma (C2) |
| Reposicionar / Editor / Melhorar com IA | sim | não | OK (web é revisão rápida), mas a web não avisa que existem |
| Ver/cancelar agendados | não | não (Planner só leitura) | I7 |
| Ordem da fila | `estado.json:fila_aprovacao` (ordem de chegada/edição manual) | banco, `criado_em ASC` | **Duas filas diferentes**: o "próximo" pode não ser o mesmo nos dois canais |

**Proposta de nomenclatura única** (mesmo texto nos dois canais):
**Aprovar** (horário automático) · **Aprovar só o feed** (horário automático) ·
**Agendar às…** · **Publicar agora** (com confirmação, só na web; alinhado com
Saulo) · **Descartar** (com confirmação) · **Deixar para depois** (move para o fim,
opcional). Remover "posta so o story" imediato ou colocá-lo atrás de
confirmação. Uma fila só: o dashboard lê a ordem de `fila_aprovacao`, ou os
dois ordenam por `criado_em`.

---

### I9 — IMPORTANTE — Frases que o operador vai usar fazem outra coisa

- "posta so o story" → `post_story_agora.py` na hora, **sem gate** (as ações de
  navegador somam no anti-spam, ver Fase 0 do `ROADMAP_ALTO_VOLUME`), sem
  avançar a fila e sem marcar o banco.
- "aprovado pode postar", "posta agora", "vai no ar", "publica" → agendam
  (auto-slot). O operador novo vai achar que publicou.
- Correção: resposta sempre com o efeito real e o horário ("Agendada para 14:27,
  não foi publicada agora"). Mapear "posta agora"/"publica" para a mesma ação,
  com esse texto. Story imediato: remover ou exigir confirmação.

---

### I10 — IMPORTANTE — Prontidão para handoff (item 7)

**Avaliação:** a documentação atual **não serve** para um operador sem o
contexto desta semana.
- `CONTEXTO_MSCONECTA.md`: 516 linhas / 115 KB, descreve internals (funções,
  locks, bugs históricos). É bom para engenharia, inutilizável como guia do dia.
- `CLAUDE.md`: só geração de imagem. É nota para IA, não para operador.
- `MANUAL_OPERACIONAL.md` (projeto): última atualização 18/05, desatualizado
  (cita crontab inexistente).
- Ajuda do bot (`acao_ajuda`): não menciona descarte nem reposicionamento, e
  lista "posta so o story" sem avisar que é imediato.

**Proposta:** novo `MANUAL_DO_OPERADOR.md` (neste repositório), linguagem simples,
**2-3 páginas**, escrito **depois** de C2/C3/I7/I8 (senão documenta nomes que vão
mudar). Sumário:
1. O que acontece sozinho (notícia nova vira design e chega no Telegram, e o
   sistema publica no horário, uma a cada ~27 min, entre 7h e 22h).
2. O dia a dia, em 4 botões: Aprovar · Aprovar só o feed · Agendar às… ·
   Descartar (e como desfazer).
3. Imagem ruim: Reposicionar / Editor visual / Melhorar com IA.
4. Ver o que está agendado e cancelar um agendamento.
5. Alertas: tabela "mensagem → o que significa → o que fazer → quando chamar
   Saulo" (Feed NÃO publicado, Rate limit, PC offline, cota perto do limite,
   item >24h, token expirando, falha de design).
6. O que **nunca** fazer (editar arquivos, mandar vários comandos seguidos sem
   esperar a resposta, clicar em botões de mensagens antigas).
7. Contato/escalonamento.

Também: `acao_ajuda()` passa a ser um resumo desse manual. `MANUAL_OPERACIONAL.md`
antigo é marcado como obsoleto.

---

### I11 — IMPORTANTE — Documentação divergente do estado real

- **Restauração desfeita não registrada:** o banco tem, às **15:20:25 UTC de
  26/09**, eventos "Restauracao desfeita a pedido de Saulo (2026-09-26): itens
  voltam a descartados" para os 5 itens restaurados (Dourados curso, Peteca,
  Magistério, Autocine, Corumbá mutirão). Estão `cancelado` e **sem entrada** em
  `agendamentos.json`. O `HISTORICO_MUDANCAS.md` (entrada do topo) ainda diz que
  estão `pendente`/`agendado` para 19:53-21:41 CG. **Não corrigido nesta sessão**
  (só mapeamento). Registrar na próxima sessão de aplicação.
- `ROADMAP_OPERACAO_SIMPLES.md`, citado no pedido, **não existe** no
  repositório. Este plano cumpre esse papel por enquanto.
- `CONTEXTO` seção "Auto-slot" descreve V2 ligada + simplificação pendente. Está
  correto hoje, mas muda com I1.

---

### I12 — IMPORTANTE — Fallback para "pasta mais recente"

`acao_publicar_feed`/`acao_publicar_so_feed`/`acao_programar`/`acao_publicar_story`
usam `estado.get('ultima_pasta') or encontrar_pasta_recente()`. Não checam se
há aprovação **pendente**. "aprovado" enviado sem item pendente reaprova a
última pasta (o `agendar_auto` evita duplicar se ainda estiver `pendente`, mas não
se já foi publicada). Correção: exigir item pendente, senão responder "Nada
aguardando aprovação".

---

### D1-D4 — DESEJÁVEL

- **D1** Story considerado publicado quando o PC responde `ok:true` (pedido
  aceito). Handler travado no PC = "sucesso" falso (caso de 21/09). Confirmar
  via `GET /{ig-user}/stories` alguns minutos depois ou endpoint de status no PC.
- **D2** `aprovar_e_postar_agora()` responde "Publicado: feed + story no
  Instagram, Threads e Facebook." fixo. Usar `threads_ok`/`facebook_ok` do
  resultado.
- **D3** Limpar backups com segredos depois da rotação (I5); trocar
  `StrictHostKeyChecking=no`.
- **D4** Funky Fresh (outro cliente) dentro de `/root/msconecta` e do mesmo
  crontab. Separar em diretório próprio num momento calmo. Não urgente.

---

## 4. O que NÃO foi achado como problema (para não mexer)

- Lock de `agendamentos.json` (`_lock_agendamentos_io`) e trava por pasta
  (`_trava_pasta`): corretos.
- Idempotência por canal e status "publicado só com feed confirmado": corretos
  e essenciais.
- Timeouts nas chamadas da Graph API (24/09) e retry do Threads: presentes.
- `health_check_pipeline.py` (PC/serviços) e renovação de tokens com watchdog:
  cobrem o que prometem.
- `msconecta-docs` (repositório com remote no GitHub): nenhum padrão de token
  de bot encontrado no histórico.

---

## 5. Ordem de execução sugerida (lotes para autorizar um a um)

| Lote | Conteúdo | Por que nesta ordem | Risco |
|---|---|---|---|
| **L0** | C6 passos 0-5 (congelar, testar em cópia, `.gitignore`, commits, tag) | Ponto de retorno antes de mexer em qualquer coisa | nenhum em produção |
| **L1** | C1 (descartar cancela o agendamento) + C2 (confirmação/eco/renomear/desfazer) + C3 (botões por `msg_id`) | Fecha a perda de hoje e a publicação indevida de ontem | baixo-médio (bot + `pipeline_acoes`) |
| **L2** | C4 (sync) + C5 (lock único do `estado.json`) | Consistência entre banco, fila e JSON | baixo |
| **L3** | I2 + I3 + I4 (alertas mortos/silenciosos) | Rede de segurança antes do handoff | baixo |
| **L4** | I1 (agendamento simples, em 3 passos: flag off → fórmula → limpeza) | Depois de L0-L3, com a simulação de cron obrigatória | médio |
| **L5** | I7 + I8 + I9 + I12 (ações e nomes) | Define a interface final | baixo-médio |
| **L6** | I10 (manual do operador) + ajuda do bot + I11 | Documenta a interface já estável | nenhum |
| **L7** | I5 + I6 (segredos, portas) | Parte depende de Saulo (BotFather, Cloudinary, senha Windows). Pode rodar em paralelo a L1-L6 | baixo, com restart um a um |
| **L8** | D1-D4 | Quando der | baixo |

---

## 6. Decisões pendentes de Saulo (antes de aplicar)

1. **"pular"**: vira "Deixar para depois" (não descarta) ou é removido? Descartar
   passa a ser só o botão/comando "Descartar" com confirmação.
2. **Publicar agora**: fica só no dashboard (com confirmação), some de vez, ou
   volta ao Telegram com confirmação?
3. **"posta so o story" imediato**: remover ou exigir confirmação?
4. **Janela 7h-22h CG**: manter como única regra além do intervalo? (recomendado)
5. **Teto 60/24h**: aceita tirar do cálculo e deixar só como número no relatório
   diário?
6. **Commit de `publicar_instagram.py`**: aceita um commit-retrato se a separação
   por snapshots não sair limpa?
7. **Fila única**: o dashboard passa a seguir a ordem da fila do Telegram, ou os
   dois ordenam por data de geração?
8. **Token do bot**: autoriza revogar no @BotFather depois de migrar o código
   para ler do ambiente (exige restart do bot)?
9. **`publicar_via_pc.sh`**: aposentar ou migrar para chave SSH?
10. Registrar no `HISTORICO` a restauração desfeita das 15:20 UTC (I11). O motivo
    é conhecido por Saulo, não pela varredura.

---

## 7. Como esta varredura foi feita (para reproduzir)

- Leitura: `CONTEXTO_MSCONECTA.md`, `HISTORICO_MUDANCAS.md` (entradas de 21-26/09),
  `CONVENCOES.md`, `ROADMAP_PIPELINE_DASHBOARD.md`, `ROADMAP_ALTO_VOLUME.md`,
  `/root/msconecta/CLAUDE.md`.
- Código: `orquestrador.py`, `telegram_bot.py`, `pipeline_acoes.py`,
  `publicar_instagram.py` (`processar_agendamentos`, auto-slot, alertas),
  `sync_pipeline_db.py`, `pipeline_lib.py`, `monitor_noticias.py`,
  `dashboard_pipeline.py`, `slot_simples.py`, `editor_visual.py`.
- Dados (só leitura): `msconecta.db` aberto com `mode=ro`. Consulta-chave de C1/C4:
  para cada `content_item` recente, comparar o último evento com
  `origem != 'sync'` (por `id`) com o `estagio` atual e com o último status da
  pasta em `agendamentos.json`. Resultado: 55 divergências em 426 itens (39
  agendado→publicado, esperadas; 4 **cancelado→publicado**; 8
  publicado_parcial→erro; 3 agendado→cancelado; 1 aguardando→erro).
- Segredos: `grep -l` por padrão de token e nomes de variável; `sshpass -p`
  mascarado na saída. Nenhum valor impresso.
- Rede: `ss -ltnp`, `ufw status`, `iptables -S INPUT` (sem teste externo).
