# ROADMAP_ALTO_VOLUME.md

> Plano técnico para adaptar o pipeline do MSConecta a um volume diário de
> notícias significativamente maior que a linha de base original (~30/dia).
> Documento de referência para todas as sessões de Claude Code que trabalharem
> neste projeto — ler antes de iniciar qualquer fase abaixo.
>
> Criado em 2026-09-24, motivado por um dia com mais de 60 notícias publicadas
> no site — o dobro do volume documentado originalmente em
> CONTEXTO_MSCONECTA.md. Não se sabe ainda se esse volume é um pico pontual ou
> o novo normal; este roadmap prepara o sistema para o segundo cenário, sem
> assumir que ele vai se confirmar.

---

> ## ⚠️ CAPACIDADE REAL DO SISTEMA (descoberta de 2026-09-26) — LER PRIMEIRO
>
> **O gargalo de volume é o gate de publicação de 25-30 min, NÃO o teto de 60/24h.**
> `espacamento_liberado()` (`publicar_instagram.py`; `INTERVALO_MIN_PUBLICACAO_MINUTOS`=25
> + `INTERVALO_JITTER_MAX_MINUTOS`=5) libera no máximo uma tentativa de publicação
> a cada 25-30 min (média ~27,5). Isso impõe um **limite físico**:
>
> | Janela | Capacidade máxima |
> |---|---|
> | Janela do auto-slot (07h-22h CG, 15h) | **~33 posts/dia** |
> | 24h corridas (cron roda o dia todo) | **~52 posts/24h** (1440 ÷ 27,5) |
>
> O teto móvel `TETO_MOVEL_FEEDS_24H=60` **nunca é o limite efetivo**: o gate
> satura antes. Qualquer plano que precise de >33/dia dentro da janela 7h-22h
> (ou >52/24h) exige mudar o gate ou a janela — o auto-slot sozinho não resolve.
> O gate foi elevado de 10 para 25-30 min depois de 16 rate limits reais da Graph
> API (2026-09-25); baixá-lo é risco medido, não otimização gratuita.
>
> **Auto-slot e gate agora usam a MESMA constante** (fonte única, 2026-09-26):
> antes o auto-slot marcava slots de 15-20 min e o gate segurava a publicação em
> 25-30 min, então o horário mostrado prometia algo que não se cumpria (fila
> acumulava atraso, ~2h no fim do dia). Agora o horário mostrado = mais cedo
> possível real. Ver `HISTORICO_MUDANCAS.md` (2026-09-26, espaçamento unificado).


## 1. Motivação

O `monitor_noticias.py` foi desenhado para um volume bem menor do que o
observado hoje. Ele tem uma limitação estrutural já identificada em incidente
anterior (ver `HISTORICO_MUDANCAS.md`, 2026-09-22): só lê a página 1 do site
(20 notícias mais recentes) e só tenta processar 5 por ciclo. Numa rotina de
~30 notícias/dia isso já causava perda silenciosa ocasional; com 60+/dia, o
problema escala proporcionalmente — mais notícias empurram itens para fora da
página 1 antes de terem a vez na fila de 5, e eles somem para sempre, sem
alerta.

Ao mesmo tempo, a capacidade de publicar no mesmo dia esbarra em limites reais
da Meta Graph API (observado ao vivo em 2026-09-24: erro "User is performing
too many actions" ao tentar recuperar um backlog rapidamente demais).

## 2. Restrição de design não-negociável

**A aprovação de cada notícia continua 100% manual, feita por Saulo.** Esta
não é uma limitação temporária a ser contornada — é uma decisão explícita.
Nenhuma fase deste roadmap deve implementar aprovação automática, mesmo que
pareça uma solução óbvia para o volume. O trabalho de engenharia deve reduzir
o ATRITO de cada aprovação, nunca eliminar a aprovação em si.

**Consequência importante, para deixar registrada**: com aprovação manual
mantida, o teto real de throughput do sistema é o tempo disponível de Saulo
para revisar, não a infraestrutura. Nenhuma fase abaixo resolve isso — na
melhor das hipóteses, reduz o tempo gasto por aprovação. Se o volume alto se
confirmar como rotina e a revisão manual de todo item deixar de ser
sustentável, essa é uma decisão a ser revisitada por Saulo no futuro, não
algo que uma sessão de código deva presumir ou empurrar.

## 3. Objetivo

Adaptar o pipeline para: (a) nunca perder uma notícia silenciosamente,
independente do volume; (b) publicar o máximo possível de notícias já
aprovadas no mesmo dia, respeitando os limites reais da Meta; (c) reduzir o
tempo que Saulo gasta por aprovação, sem automatizar a decisão em si.

## 4. Princípios de design

- **Nunca perder notícia em silêncio.** Estende o princípio já consolidado em
  `incidents-and-learnings` (falha silenciosa é o padrão de risco #1 do
  projeto) para a etapa de DESCOBERTA, não só publicação — hoje o problema
  nem chega a gerar alerta quando uma notícia é perdida antes de tentar.
- **Aprovação manual é intocável.** Ver seção 2.
- **Medir o limite real da Meta antes de desenhar em cima dele.** Não
  assumir número — o "49/100" observado hoje pode não refletir o limite real
  (pode ser por hora, por dia, por tipo de ação). Investigar com evidência
  antes de construir o pacer.
- **Aditivo, não disruptivo.** Cada fase deve ser segura de aplicar sem
  quebrar o que já funciona — mesmo padrão de todas as correções desta
  semana.

## 5. Fases

### Fase 0 — Investigação
**Entrega:** medição real dos limites de taxa da Meta Graph API (por
hora/dia, rígido/suave, por tipo de ação — feed, story, Threads, Facebook
podem ter limites distintos, já sabemos que são rastreados separadamente
desde o fix de 2026-09-22); auditoria numérica de quantas notícias
tipicamente se perdem por dia no comportamento atual de página-1+5-por-ciclo,
usando dados reais de dias de alto volume.
**Risco:** nenhum — é só leitura/medição, sem mudança de comportamento.

#### Resultados da Fase 0 (medição feita em 2026-09-24, ~22h UTC)

Nenhum código de produção foi alterado. Fontes: documentação oficial da Meta,
2 chamadas GET de leitura às APIs (`content_publishing_limit`,
`threads_publishing_limit`), `instagram.log` + `monitor.log` (15 dias
retidos, 2026-09-10 a 2026-09-24), `logs/reel.log`, `noticias_vistas.json`,
`cache_noticiasmetadados.json`, `output/**/meta.json` e os sitemaps de
notícias do próprio site (`/arquivos/sitemap/news_08_26_*.xml` e
`news_09_26_*.xml`, 1.499 notícias de 2026-08-01 a 2026-09-24).

**A. Limites da Meta — documentação oficial**

| Plataforma | Limite documentado | Janela | Como consultar | Uso medido agora |
|---|---|---|---|---|
| Instagram Feed (Content Publishing API) | 100 posts publicados via API (carrossel conta como 1) | 24h móvel | `GET /{ig-id}/content_publishing_limit` → `config.quota_total=100`, `quota_duration=86400` | **49/100** |
| Instagram Story | não se aplica — o Story **não usa a API** (automação de navegador no PC Windows, ver `CONTEXTO_MSCONECTA.md` 3.5); nenhuma cota de API documentada para esse caminho | — | — | 48 stories hoje |
| Threads | 250 posts (+1.000 replies) | 24h móvel | `GET /{threads-id}/threads_publishing_limit` → `quota_total=250` | **58/250** |
| Facebook Page | sem teto de posts documentado; limite é de **chamadas**: 4800 × usuários engajados / 24h | 24h | header `X-Business-Use-Case-Usage` | irrelevante hoje — ver achado 3 abaixo |
| Chamadas de API (Instagram/Threads) | 4800 × impressões / 24h; headers `X-App-Usage`/`X-Business-Use-Case-Usage` são **% de uso numa janela móvel de 1h** | 1h (header) | headers de resposta | `X-App-Usage: call_volume 0, cpu_time 0` — longe de qualquer limite |

Cada plataforma tem teto próprio e independente (Feed e Threads têm
endpoints de cota separados; Story não passa pela API).

Fontes: [Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing),
[Rate Limiting](https://developers.facebook.com/docs/graph-api/overview/rate-limiting),
[Threads overview](https://developers.facebook.com/docs/threads/overview),
[Instagram error codes](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/error-codes).

**De onde veio o "49/100"**: não é header — é o corpo da resposta de
`GET /{ig-id}/content_publishing_limit` (`quota_usage` / `quota_total`), janela
**móvel de 24h** (`quota_duration=86400`). Reconsultado às ~22h30 UTC: continua
49. Pelos logs, o que saiu nas últimas 24h foi 39 feeds via cron + 2
republicações manuais + 2 Reels (`publicar_reel.py`) = 43. Os ~6 que faltam
para chegar em 49 não foram explicados (hipótese não confirmada: tentativas de
`media_publish` que falharam também contam).

**A. Observação empírica — o "too many actions" NÃO é a cota de 100**

1. **Histórico**: em 15 dias de `instagram.log`, o erro `User is performing
   too many actions` só aparece em **2026-09-24** (15 vezes). Nenhuma
   ocorrência de 2026-09-10 a 2026-09-23.
2. **Não foi só no catch-up**: o erro começou às **01:45 UTC, antes do
   travamento do lock** (02:01), e voltou às 17:45, 18:15, 18:30, 21:00 e
   21:15, todas com a cadência normal de 1 feed a cada 15 min. Isso corrige a
   leitura do incidente do mesmo dia (`HISTORICO_MUDANCAS.md`), que atribuía o
   erro a uma rajada de recuperação da fila.
3. **Sempre na etapa `media_publish`**: o container é criado e fica
   `FINISHED`; a falha é só no publish.
4. **Limiar observado**: a primeira falha veio quando os feeds publicados nas
   últimas 24h chegaram a **48**, o maior valor dos 15 dias. O maior valor
   anterior foi **43** (2026-09-23 23:46), sem nenhum erro. Contando todas as
   ações da conta no Instagram (feed via API + story e canal via navegador),
   foram 144 em 24h na primeira falha, contra 129 no máximo anterior, que
   passou sem erro.
5. **Não parece haver teto por hora**: antes de hoje houve janelas de 1h com
   até 16 ações sem erro. Hoje houve falhas com 5–12 ações na hora.
6. **Interpretação (hipótese, não confirmada)**: a mensagem não é nenhuma
   das mensagens oficiais de cota (código 9 / subcódigo 2207042 = "maximum
   number of posts"), e a cota oficial estava em 49/100. O mais provável é um
   **limite anti-spam/de ação da conta** (família código 4 / subcódigo 2207051,
   "restrict certain activity"): não documentado numericamente, dinâmico, e
   possivelmente somando também as ações de navegador (story + canal). **O
   log só grava a mensagem, não o `code`/`error_subcode`**. Gravar esses
   campos é pré-requisito para confirmar o diagnóstico (mudança de código,
   fica para a Fase 1/3).

**Número recomendado (Tarefa A):** **até ~40 feeds por 24h móveis**,
mantendo o espaçamento mínimo de 15 min já usado (≈ 2–4/h), com Threads sem
restrição prática (250/24h) e Story limitado pelo mesmo teto do feed (é a
mesma conta, e as ações somam no anti-spam).
- Fonte do número: **observação empírica, não documentação**. 43/24h passou
  limpo, os erros começaram em 48/24h, e 40 dá margem abaixo disso. O teto
  oficial de 100/24h **não é** o limite que manda na prática hoje.
- Confiança: **baixa-média**. É só 1 dia com o erro, e limites anti-spam da
  Meta mudam por conta e por histórico. Recalibrar depois de mais dias de
  alto volume, e só depois que o log passar a gravar `code`/`subcode`.
- Consequência para a Fase 3: com 60–70 notícias/dia aprovadas, **não cabe
  tudo no feed do mesmo dia** com margem segura. O pacer precisa de um teto
  diário (não só de um intervalo mínimo), e o excedente precisa de uma regra
  explícita a definir com Saulo (ex.: próximo dia, só Story/Threads,
  priorização).

**Achados colaterais (não corrigidos, fora do escopo da Fase 0):**
1. `HISTORICO_MUDANCAS.md` **não tem entrada de 2026-09-22**. As referências
   deste roadmap ao "incidente de 2026-09-22" (seção 1) e ao "fix de
   2026-09-22" (Fase 0) não têm registro correspondente no histórico nem no
   git deste repositório.
2. `monitor_noticias.py` processa as notícias novas **da mais nova para a mais
   antiga** (`novas[:5]` na ordem da página). Num pico, as mais antigas do
   backlog ficam sempre para depois e acabam saindo da página 1. É o
   mecanismo real das perdas medidas na Tarefa B abaixo.
3. **Facebook Page não publica nada há pelo menos 15 dias**: 100% das
   tentativas em `instagram.log` (todas desde 2026-09-10) retornam
   `(#200) The permission(s) publish_actions are not available. It has been
   deprecated`. Falha silenciosa: o fluxo segue normalmente. Precisa de
   investigação própria (token/permissão da Page).

**B. Perda real do `monitor_noticias.py`**

Método: cada notícia dos sitemaps do site (data/hora = `lastmod`, horário de
Brasília) foi cruzada com tudo que o monitor tocou (`noticias_vistas.json`,
`cache_noticiasmetadados.json`, `meta.json` das pastas de output, títulos em
`monitor.log`), por slug e por título. Uma notícia "nunca tocada" = perdida.
Casos conferidos manualmente em 2026-09-24: 1 falso positivo (título e slug
editados depois do processamento) e 4 publicadas depois do último ciclo
(pendentes, não perdidas) foram descontados.

| Faixa de volume diário | Dias | Notícias | Perdidas | % | Perdidas/dia |
|---|---|---|---|---|---|
| < 30/dia | 26 | 378 | 6 | 1,6% | 0,23 |
| 30–44/dia | 19 | 697 | 12 | 1,7% | 0,63 |
| 45+/dia | 8 | 424 | 10 | 2,4% | 1,25 |
| **Total (2026-08-01 a 09-24)** | 53 | 1.499 | **28** | 1,9% | 0,53 |

Dias de alto volume: 08-13 (52): 0 · 08-19 (45): 2 · 09-02 (46): 0 · 09-03
(52): 0 · 09-04 (46): 2 · 09-10 (56): 1 · 09-23 (57): 0 · **09-24 (70): 5**.

- **Escala com o volume, mas não de forma linear**: o preditor forte é o
  **pico concentrado**, não o total do dia. Nos 22 dias em que o máximo de
  publicações numa janela de 90 min ficou em ≤ 12, houve 1 perda. Nos 31 dias
  com picos de 13+ em 90 min, houve 27. Um dia de 57 notícias bem distribuídas
  (09-23) teve 0 perdas; um dia de 25 com um pico (08-24) teve 2.
- **Mecanismo confirmado no log de 09-24**: às 17:30 UTC havia 10 novas
  (processou 5, as mais novas), às 18:00 havia 15 (processou 5), às 18:30
  havia 10. As 5 notícias publicadas entre 14:00 e 14:13 BRT (17:00–17:13 UTC)
  foram passadas para trás a cada ciclo e saíram da página 1. Mesmo padrão em
  09-10 (12:30–13:30 UTC, backlog de 9 → 12 → 10). Falhas de scrape (PC
  offline + proxies bloqueados) aumentam o backlog e agravam o efeito.
- **As perdas são silenciosas**: nenhuma das 28 gerou alerta. Elas vêm em
  lotes de 2–3 notícias publicadas com poucos minutos de diferença.
- Ressalva de método: `lastmod` é a data de **modificação**, não
  necessariamente a de publicação. Notícias editadas depois podem cair no dia
  errado; isso não afeta a detecção de perda, só a distribuição por dia.

**Implicações para o desenho da Fase 1**: processar do mais antigo para o
mais novo (ou garantir que o backlog nunca exceda a janela lida), paginar pelo
menos a página 2 (`/noticias/page-2` existe) ou usar o sitemap de notícias
do mês como fonte de descoberta (tem `lastmod`, não fica restrito às 20
últimas), e alertar sempre que uma notícia sair da janela sem ter sido
processada.

### Fase 1 — Corrigir o gargalo estrutural de descoberta
**Entrega:** `monitor_noticias.py` passa a paginar além da página 1 e a
processar todo o volume descoberto por ciclo (não mais travado em 5) — com
controle de custo se a geração de design envolver chamadas de IA pagas
(upscale via Replicate, decisão de destaque de título). Qualquer notícia que
mesmo assim falhe a gerar design após N tentativas dispara alerta explícito
via Telegram/WhatsApp, nunca desaparece sem rastro.
**Risco:** baixo-médio — toca o núcleo da descoberta automática; testar com
volume real antes de considerar concluído.

### Fase 2 — Aprovação mais rápida por item
**Entrega:** modo de revisão rápida no dashboard do pipeline (Board),
priorizando thumbnail + ação em poucos cliques/toques, para reduzir o tempo
médio de aprovação por notícia sem enfraquecer a revisão visual real de
Saulo. Telegram continua disponível em paralelo, não é substituído.
**Risco:** baixo — é interface nova sobre dado já existente, não muda a
lógica de aprovação em si.

### Fase 3 — Distribuição inteligente de publicação no mesmo dia
**Entrega:** depois de aprovadas, as notícias são automaticamente
distribuídas pelas horas restantes do dia no ritmo máximo seguro (informado
pelos números reais da Fase 0), priorizando as mais antigas/urgentes
primeiro, com margem de segurança abaixo do limite real para nunca repetir o
"too many actions" de 2026-09-24.
**Risco:** médio — toca a lógica de agendamento/publicação automática em
produção ativa; testar com cautela, seguindo o mesmo padrão de catch-up
gradual já usado nos incidentes desta semana.

## 6. Fora de escopo (por ora)

- Aprovação automática de qualquer categoria de notícia — explicitamente
  recusada por Saulo (ver seção 2).
- Garantir literalmente 100% das notícias no mesmo dia, para qualquer volume
  — os limites da Meta impõem um teto real; o objetivo honesto é maximizar o
  throughput seguro do dia, não prometer o impossível.
- Decidir se 60+/dia é definitivamente o novo normal — isso só vai ficar
  claro com mais dias de observação; este roadmap prepara o sistema para
  essa possibilidade sem assumi-la como fato.

### Nota 2026-09-26 — Auto-slot: regras V2 implementadas (flag desligada)

O auto-slot é preenchimento contínuo (janela 7h-22h CG, 15-20min com jitter).
As 3 lacunas apontadas antes foram implementadas em `auto_slot_regras.py`,
**ligadas em produção desde 2026-09-26** (`AUTO_SLOT_REGRAS_V2=true` no `.env`; remover a linha desliga):
1. **Teto móvel de 60/24h** (Saulo escolheu 60, não os ~40 da Tarefa A acima).
   Ao ligar, ~20 dos 25 pendentes de 26/09 são empurrados (não descartados).
   Se o rate limit code 9 reaparecer, reavaliar para baixo **com dados**.
2. **FIFO por horário de geração** (`content_items.criado_em`): permuta os
   horários do auto-slot (itens marcados `origem_slot=auto_slot`).
3. **Alerta de itens >24h desde a geração** (no horário planejado): resumo
   único por ciclo no Telegram, `alerta_24h_enviado_em` evita repetição,
   **nunca descarta** (aprovação manual é intocável — seção 2).

Observação de capacidade: com ~76 itens numa janela de 24h, mesmo o teto 60
adia a fila para 27/09 ~07:38; a causa é volume acima do teto, não o
algoritmo de slots.

### Nota 2026-09-26 (2) — Teto móvel: contagem local não é a verdade

O teto móvel de 60/24h depende da contagem de publicados; entradas
`publicado` falsas (feed falhou por rate limit, status gravado errado)
inflaram a contagem em 9 e empurraram a fila para 18h. Agora o teto confere
a contagem local com `quota_usage`/mídias da API real (só age no excesso,
fail-safe se a API cair) e alerta 1x/dia se divergir ≥3. Pendência de fundo
(Fase 1/3): gravar `code`/`subcode` do erro e nunca marcar `publicado`
sem confirmação do feed.

### Nota 2026-09-26 (3) — O gate de 25-30 min é o teto real de volume

Ver o quadro no topo deste documento. Consequência para as fases: qualquer
meta de volume deve ser comparada com ~33/dia (janela 7h-22h CG) ou ~52/24h,
não com o teto de 60. Fase 3 (distribuição no dia) e qualquer discussão de
"aumentar o teto" são inócuas enquanto o gate não mudar. Decisão pendente de
Saulo, se o volume orgânico exigir mais: estender a janela do auto-slot
(ex.: 6h-23h CG → ~36/dia) e/ou reduzir o gate com medição de rate limit.
Auto-slot e gate leem as mesmas constantes (`INTERVALO_MIN_PUBLICACAO_MINUTOS`,
`INTERVALO_JITTER_MAX_MINUTOS`); as variáveis `INTERVALO_*_APROVACAO_MINUTOS`
foram aposentadas.

### Nota 2026-09-26 (4) — Mudança de direção: auto-slot simplificado (intervalo fixo)

**Decisão de Saulo (definitiva):** abandonar a complexidade do teto móvel de 24h
e do FIFO no cálculo **ativo** do horário. O cálculo passa a ser apenas:

`PRÓXIMO_SLOT = MAX(último_horário_já_agendado, agora) + 27min`

(27min = meio do intervalo 25-30min validado contra a Meta; ver Nota 3). Sem
teto de 24h como bloqueio, sem FIFO reordenando, sem recalcular por itens
antigos ou fantasmas. Implementação: `slot_simples.py` (dry-run feito em
26/09; **aplicação pendente de confirmação**).

**Motivo:** a complexidade acumulada gerou mais bugs do que valor no mesmo dia:
contaminação do teto por "publicados" fantasmas (fila empurrada em 3-5h),
saltos de horário, FIFO permutando horários entre aprovações (itens novos
recebendo os mesmos slots de itens já aprovados; salto de 1h52 entre 14:35 e
16:27 CG) e confusão de ordem. Como o gate de 25-30min já limita fisicamente
a ~33/dia (~52/24h), o teto de 60 era redundante.

**Mantido, separado do cálculo:** alerta de 24h desde a geração (só avisa,
nunca descarta nem bloqueia). O teto móvel pode continuar como **métrica** de
monitoramento, sem influenciar o horário.

**Ponto em aberto:** sem a janela 7h-22h CG a fila atrasada cai na madrugada;
decidir se a janela fica ou não.

## 7. Como usar este documento

Cada sessão que for trabalhar numa fase deve ler este documento inteiro
antes de começar, confirmar qual fase está em andamento via
`HISTORICO_MUDANCAS.md`, e nunca implementar nada que contradiga a seção 2
(aprovação manual intocável), mesmo que pareça uma otimização óbvia de
throughput.
