# CONTEXTO_MSCONECTA.md

> Documento de mapeamento técnico do projeto **MSConecta**, gerado por IA em 2026-08-06 a partir de leitura direta do código-fonte (em `/root/msconecta`), configurações de sistema (systemd, cron, nginx) e documentação interna já existente (`CLAUDE.md`, `MANUAL_OPERACIONAL.md`, `cms_msconecta.md`). Onde o código não deixava algo claro, o item foi marcado como **"não identificado"** em vez de suposto.
>
> Esta é a cópia **versionada** deste documento, mantida em `/root/msconecta-docs/` (repositório git separado dos scripts do projeto, ver `CONVENCOES.md` nesta mesma pasta para as regras de atualização).
>
> **Nenhuma credencial, token ou senha está reproduzido neste documento**, mesmo quando o autor destas notas os encontrou em texto plano no código-fonte (ver seção 6 — Pontos de Atenção).

---

## 1. VISÃO GERAL

O **MSConecta** é o sistema de automação editorial e de redes sociais do portal de notícias **msconecta.com.br** (Mato Grosso do Sul). Ele cobre praticamente todo o ciclo de uma notícia:

1. **Descoberta de pautas** (monitoramento de fontes/RSS locais do MS),
2. **Redação automática** da notícia via LLM e publicação no CMS do site,
3. **Geração automática do material visual** para redes sociais (imagem de feed e de story, no padrão editorial do veículo),
4. **Publicação multiplataforma** (Instagram feed + story com link sticker, Facebook Page, Threads),
5. **Geração de Reels** (vídeos verticais) a partir de vídeos virais curados,
6. **Aprovação humana via bot de Telegram** (o "editor" aprova/ajusta cada etapa antes de publicar),
7. **Relatórios e métricas** de publicação e engajamento.

Todo o fluxo é pensado em torno de um **editor humano usando o Telegram como painel de controle**: o bot manda pautas e previews de design com botões inline, e a pessoa aprova, rejeita ou pede ajustes em linguagem natural.

### Stack técnica

| Camada | Tecnologia |
|---|---|
| Linguagem principal | Python 3.12 (scripts standalone, sem framework web tradicional) |
| Automação de redes/Reels | Node.js — Express 5 + Remotion 4 (`/opt/msconecta-reels`) |
| Bot de mensagens | Telegram Bot API, via um **servidor Telegram Bot API local** (`telegram-bot-api --local`, porta 8081) — não usa a API pública `api.telegram.org` para o bot principal |
| Geração de imagem | Pillow (PIL), NumPy, biblioteca `seam-carving` (recorte de imagem com "seam carving"/ costura de custura mínima) |
| IA — texto | Groq (`llama-3.3-70b-versatile`) para redação de notícias; Anthropic Claude (`claude-haiku-4-5-20251001`) para classificação de intenção do bot e para Vision (detecção de ponto focal em imagens) |
| Web scraping | `requests` + `BeautifulSoup4` |
| CMS do site | WEBSG v5.6.3 (CMS de terceiros, PHP, acessado via automação de formulário/POST — ver `cms_msconecta.md`) |
| APIs de redes sociais | Graph API do Instagram (`graph.instagram.com`), Graph API do Facebook (`graph.facebook.com`), API do Threads |
| Hospedagem de imagem para posts | Cloudinary (upload das imagens geradas antes de enviar para a Graph API) |
| Editor visual web | FastAPI + Uvicorn (`editor_visual.py`), servido na porta 8090 |
| Dashboard de monitoramento | Servidor HTTP nativo do Python (`http.server`), porta 8085 |
| Automação de terceiros | n8n (self-hosted, usado para pelo menos um fluxo — calendário editorial de outro cliente hospedado na mesma VPS, ver observação abaixo) |
| Proxy/servidor web | nginx |
| Processos persistentes | systemd (5 serviços dedicados, ver seção 2) |

### VPS x PC servidor — por que dois ambientes

O sistema roda principalmente em uma **VPS Linux** (`72.60.151.58`), mas depende de um **PC Windows** (`100.108.170.120`, acessível via **Tailscale**, hostname/usuário `msconecta`) para tarefas que a VPS não consegue fazer sozinha:

- **Site com bloqueio Cloudflare/anti-bot para IPs de datacenter**: `msconecta.com.br` bloqueia requisições vindas da VPS (403), mas não do IP residencial do PC Windows. Por isso, buscar notícias, imagens e publicar no CMS depende do PC.
- **Postagem de Story no Instagram com link sticker via navegador**: feito via Chrome + Playwright rodando no PC Windows (a API oficial do Instagram tem limitações para link sticker que o fluxo contorna via automação de navegador).
- **Renderização de vídeo (Reels)**: o PC Windows (`RENDER_BASE = http://100.108.170.120:3001`) é a **primeira via tentada** (não um fallback de sobrecarga), via um servidor FFmpeg puro (não Remotion — ver seção 3.6 para a cadeia real de fallback e a correção desse ponto, registrada em `HISTORICO_MUDANCAS.md` 2026-08-13).

A VPS conversa com o PC via **chamadas HTTP simples** para pequenos servidores rodando no Windows (PowerShell `HttpListener` e/ou Playwright em Node/Python), todos expostos apenas na rede privada do Tailscale. Detalhes completos na seção 5.

**Observação sobre escopo da pasta**: o diretório `/root/msconecta` também contém scripts de um cliente completamente diferente — um cardápio/redes sociais do restaurante **"Funky Fresh"** (`atualizar_cardapio.sh`, `funky_metricas.py`, `cardapio_funky_fresh.json`, `renovar_token_funky.py` etc.) e ainda um agendamento para outro projeto ainda (`gairdin_cg`, em `/root/brandfield/`). Esses itens **não fazem parte da automação do portal MSConecta** e foram excluídos do mapeamento abaixo, exceto quando citados como ponto de atenção (pasta compartilhada entre projetos não relacionados).

---

## 2. ESTRUTURA DE PASTAS

Árvore resumida de `/root/msconecta` (raiz do projeto na VPS), com apenas as pastas relevantes à automação do portal:

```
/root/msconecta/
├── journalist/                  # Pipeline de descoberta de pauta + redação automática
│   ├── redator.py                 # Redige a notícia (Groq) e publica no CMS
│   ├── monitor_bloco1.py          # Coleta pautas do bloco 1 (manhã)
│   ├── monitor_bloco2.py          # Coleta pautas do bloco 2 (tarde)
│   ├── monitor_base.py            # Funções compartilhadas (RSS, scraping) entre os dois monitores
│   ├── buscar_imagem.py           # Busca imagem via Pexels quando a pauta não tem imagem própria (Unsplash removido 2026-08-16)
│   ├── buscar_imagem_smart.py     # Variante "inteligente" de busca de imagem (não detalhado neste mapeamento)
│   ├── coletar_historico_site.py  # Coleta histórico de notícias já publicadas no site
│   ├── analisar_historico.py      # Análises sobre o histórico coletado
│   ├── descobrir_fontes.py        # Descoberta de fontes/RSS candidatas
│   ├── perfil_editorial.json      # Perfil/estilo editorial usado pelo redator
│   ├── fontes_recomendadas.json   # Lista de fontes/RSS recomendadas
│   ├── pautas/bloco1/, bloco2/    # JSONs de pautas coletadas por bloco/dia
│   └── state/                     # Estado persistido dos monitores (bloco1_state.json, bloco2_state.json)
│
├── fontes/, fonts/               # Fontes tipográficas usadas na geração de imagem (Arboria, Barlow, Morganite)
├── moldes/                       # Templates SVG/PNG do layout de feed e story (feed.svg, story.svg, camadas alpha)
├── static/                       # Front-end do editor visual (editor.html), servido por editor_visual.py
│
├── output/YYYY-MM-DD/EDITORIA_titulo-slug/   # Saída de cada geração de design: feed.png, story.png, legenda.txt, meta.json
├── logs/                         # Logs das automações (telegram_bot.log, pipeline.log, pautas.log, relatorio.log, reel.log)
├── local-bot-api/                # docker-compose do servidor local do Telegram Bot API (dados em local-bot-api/data)
├── arquivo/                      # Backups pontuais de código (backup_20260610/, patches_20260616/)
├── relatorios/                   # PDFs de relatórios estratégicos/analíticos gerados periodicamente
├── img_temp/, tmp/, backups/     # Diretórios de trabalho/temporários
│
├── gerar_tudo.py                 # ENTRADA: gerador principal de feed.png + story.png a partir de uma URL ou modo manual
├── gerar_design.sh                # Wrapper de shell em torno de gerar_tudo.py (usado pelo monitor e pela skill de IA)
├── corrigir_posicao.py            # Ajuste incremental de posição/zoom/crop de uma imagem já gerada
├── editor_visual.py                # ENTRADA: servidor FastAPI do editor visual de posicionamento (systemd: msconecta-editor)
├── editor_tokens.py                # Tokens de sessão de curta duração para o editor visual
│
├── orquestrador.py                 # Classificador de intenção + roteador de ações do bot (usa Claude Haiku)
├── telegram_bot.py                 # ENTRADA: bot Telegram (polling), delega para orquestrador.py e reel_handler.py (systemd: msconecta-bot)
├── image_approval_utils.py         # Infra compartilhada: substituição manual de feed/story + botão "Melhorar com IA" (ver seção 3.10)
├── gemini_editor.py                 # Chamada ao Gemini (edição de imagem) usada pelo "Melhorar com IA" (ver seção 3.10)
├── pipeline.py                     # ENTRADA: pipeline editorial completo (pauta → redator → design → Instagram)
├── notificar_pautas.py             # ENTRADA: envia pautas coletadas para aprovação via Telegram
├── monitor_noticias.py             # ENTRADA (cron): detecta notícias novas publicadas no site e gera design automaticamente
├── design_queue.py                 # Fila com backoff exponencial para geração de designs (evita 429)
├── batch_12_aprovacao.py           # Variante manual: busca e enfileira as últimas 12 notícias para aprovação em lote
│
├── publicar_instagram.py           # ENTRADA (cron a cada minuto, --processar): publica feed/story/agendamentos no Instagram
├── publicar_multiplatforma.py      # Orquestra publicar_instagram.py e reporta status por plataforma (IG, FB, Threads, story)
├── postar_linkedin.py              # Publicação no LinkedIn (não detalhado neste mapeamento)
├── reel_handler.py                 # Orquestra geração/postagem de Reels, integra com /opt/msconecta-reels e o PC Windows
│
├── relatorio_diario.py             # ENTRADA: relatório diário de publicações via Telegram
├── relatorio_semanal.py            # ENTRADA: relatório semanal via Telegram
├── monitor_saude.py                # ENTRADA: verifica saúde dos serviços/integrações e alerta mudanças de estado
├── monitor_videos.py                # ENTRADA: curadoria automática de vídeos virais candidatos a Reels
├── metricas_instagram.py            # Coleta métricas de engajamento do Instagram
│
├── dashboard.py                    # ENTRADA: dashboard web de monitoramento (systemd: msconecta-dashboard, porta 8085)
├── portal_config.py                # Camada de configuração multi-portal (ver seção 6)
│
├── msconecta.db                    # Banco SQLite do pipeline de produção editorial (ver seção 3.9)
├── pipeline_schema.sql              # Schema do banco acima (content_items, reels, pipeline_events, system_health)
├── pipeline_lib.py                  # Funções de parsing compartilhadas entre migração e sincronização
├── migrar_dados_iniciais.py         # Migração única (bootstrap) dos JSONs de produção para msconecta.db
├── sync_pipeline_db.py              # ENTRADA (cron a cada 5min): sincronização incremental de msconecta.db
├── verificar_migracao.py            # Script de conferência manual (amostragem) da migração
├── health_check_pipeline.py         # ENTRADA: health-check ativo por identidade de processo (ver seção 3.9)
├── dashboard_pipeline.py            # ENTRADA: dashboard do pipeline (systemd: msconecta-pipeline-dashboard, porta 8095)
├── static_pipeline/                 # Estáticos do dashboard do pipeline (htmx.min.js vendorizado)
│
├── .env                             # Variáveis de ambiente locais ao projeto (não versionado — ver seção 4)
├── CLAUDE.md                        # Notas técnicas profundas sobre a lógica de geração de imagem (algoritmo de foco, quebra de linha, upscale via IA — removido 2026-08-16, reintroduzido 2026-08-19)
├── MANUAL_OPERACIONAL.md            # Manual operacional (comandos do bot, arquitetura resumida) — parcialmente desatualizado, ver seção 6
├── cms_msconecta.md                 # Documentação de engenharia reversa do CMS WEBSG (campos do formulário, endpoint de publicação)
└── PADRAO_LEGENDA.md                # Padrão de formatação das legendas de post
```

Outros diretórios relacionados, fora de `/root/msconecta`:

```
/opt/msconecta-reels/               # App Node.js/Remotion que renderiza os vídeos de Reels
├── server.js                        # ENTRADA: API Express (systemd: msconecta-api, porta 3001, bind 127.0.0.1)
├── src/compositions/ReelMSConecta.tsx  # Composição Remotion do template de Reel
└── scripts/render.js, analyze-frame.js # Scripts de renderização/análise de frame

/root/portais/msconecta/            # Configuração do sistema multi-portal (ver seção 6) — não identificado o conteúdo completo
```

### Arquivos de entrada/execução principais

| Arquivo | Como é iniciado |
|---|---|
| `telegram_bot.py` | Processo persistente via systemd (`msconecta-bot.service`) |
| `dashboard.py` | Processo persistente via systemd (`msconecta-dashboard.service`) |
| `editor_visual.py` | Processo persistente via systemd (`msconecta-editor.service`) |
| `/opt/msconecta-reels/server.js` | Processo persistente via systemd (`msconecta-api.service`) |
| `dashboard_pipeline.py` | Processo persistente via systemd (`msconecta-pipeline-dashboard.service`, porta 8095 — ver seção 3.9) |
| `monitor_noticias.py` | Cron, a cada 30 minutos |
| `publicar_instagram.py --processar` | Cron, a cada 1 minuto |
| `sync_pipeline_db.py` | Cron, a cada 5 minutos (ativado 2026-08-14 — ver seção 3.9) |
| `monitor_videos.py` | Cron, `0 14,20 * * *` (14:00 e 20:00 UTC — reativado 2026-08-14 após perda acidental de crontab em 2026-06-15/16, ver `HISTORICO_MUDANCAS.md`) |
| `health_check_pipeline.py` | Cron, a cada 5 minutos (ativado 2026-08-25 — ver seção 3.9) |
| `gerar_tudo.py` | Chamado sob demanda (via `gerar_design.sh`, pelo monitor, pelo pipeline ou manualmente) |
| `orquestrador.py` | Chamado sob demanda pelo `telegram_bot.py` (não roda sozinho) |
| `pipeline.py`, `notificar_pautas.py`, `relatorio_diario.py`, `relatorio_semanal.py`, `monitor_saude.py` | Scripts de execução manual/pontual — **agendamento atual não identificado com certeza** (ver seção 6) |

---

## 3. AUTOMAÇÕES E FLUXOS

### 3.1 Descoberta e notificação de pautas ("Bloco 1" e "Bloco 2")

- **Objetivo**: varrer fontes noticiosas (RSS e sites) relevantes para o MS, gerar sugestões de pauta priorizadas, e apresentá-las ao editor para aprovação.
- **Arquivos**: `journalist/monitor_bloco1.py`, `journalist/monitor_bloco2.py`, `journalist/monitor_base.py` (coleta), `notificar_pautas.py` (envio ao Telegram).
- **Gatilho**: segundo `MANUAL_OPERACIONAL.md`, um crontab agendava bloco 1 às 6h/9h e bloco 2 às 11h/14h (horário de Campo Grande). **Esse crontab não foi encontrado na crontab atual do sistema** (`crontab -l` atual só contém as rotinas descritas nas seções 3.4 e 3.5, mais itens de outro cliente). Portanto, **o gatilho atual de bloco1/bloco2/notificar_pautas/relatorio_diario/relatorio_semanal/monitor_saude não pôde ser confirmado** — pode estar sendo disparado manualmente, por um agente de IA com agendamento próprio, ou por um mecanismo não descoberto neste mapeamento. Marcar como **não identificado** até confirmação.
- **Fluxo**: `monitor_blocoN.py` coleta itens via RSS/scraping (`monitor_base.buscar_rss_feeds`) → filtra por termos do MS → grava `journalist/pautas/blocoN/pautas_DD-MM-YYYY.json` → `notificar_pautas.py` lê esse JSON, formata cada pauta com prioridade (P1–P10) e envia ao Telegram com botões inline (`✅ Publicar agora`, `❌ Rejeitar`, `🔍 Ver fonte`, `⏸ Monitorar`). Pautas **P1** recebem um alerta extra de "BREAKING" mas nunca publicam sozinhas — aprovação humana é sempre necessária.
- **Serviços externos**: feeds RSS de veículos/fontes do MS (lista não detalhada neste mapeamento); Telegram Bot API (local, porta 8081).

### 3.2 Pipeline editorial completo (pauta aprovada → publicação)

- **Objetivo**: transformar uma pauta aprovada em notícia publicada no site e depois em posts nas redes sociais, com pontos de aprovação humana no meio do caminho.
- **Arquivos**: `pipeline.py` (orquestra as 3 etapas), `journalist/redator.py` (redação), `gerar_tudo.py` (design), `publicar_multiplatforma.py` → `publicar_instagram.py` (publicação).
- **Gatilho**: execução manual via comando ao bot Telegram (ex.: `pauta 3 bloco 1 aprovada`, tratado por `orquestrador.py` → ação `publicar_cms`) ou execução direta de `pipeline.py --bloco N --numero M`.
- **Passo a passo**:
  1. **Redator** (`etapa_redator`): chama `journalist/redator.py --bloco N --numero M --data DD-MM-YYYY`, que usa o **Groq (`llama-3.3-70b-versatile`)** para escrever título, subtítulo e corpo em HTML seguindo o padrão editorial (`cms_msconecta.md` — título 60–85 caracteres, 7–9 parágrafos), e publica direto no CMS WEBSG do site (`POST .../websg/lib/actions/noticias.post.php`), retornando a URL publicada.
  2. **Designer** (`etapa_designer`): chama `gerar_tudo.py` (via `--manual` com título/editoria/imagem extraídos do JSON do redator, ou via URL) para gerar `feed.png` + `story.png`; envia preview ao Telegram com botões `Publicar agora` / `Não publicar`.
  3. **Instagram/redes** (`etapa_instagram`): ao aprovar, chama `publicar_multiplatforma.py`, que publica em Instagram (feed + story), Facebook e Threads via `publicar_instagram.py`, e registra o resultado em `historico_publicacoes.json`.
- **Serviços externos**: Groq (redação), CMS WEBSG do site (publicação da notícia), Claude/Anthropic (Vision para posicionamento de imagem, dentro de `gerar_tudo.py`), Instagram/Facebook/Threads Graph API, Cloudinary (hospedagem temporária das imagens para as APIs de rede social), Telegram (aprovações).

### 3.3 Geração automática de design de notícia (feed + story)

- **Objetivo**: gerar as duas imagens padrão de post (`feed.png` 1080×1350, `story.png` 1080×1920) a partir do título, editoria e imagem de capa de uma notícia, com posicionamento de foco automático via visão computacional.
- **Arquivo**: `gerar_tudo.py` (1725 linhas — lógica detalhada e mantida em `CLAUDE.md`), auxiliado por `corrigir_posicao.py` para ajustes incrementais.
- **Gatilho**: chamado por `monitor_noticias.py` (automático), por `pipeline.py` (fluxo aprovado), ou manualmente via `gerar_design.sh URL` / comando ao bot.
- **Passo a passo resumido** (detalhe completo em `CLAUDE.md`):
  1. Extrai a URL da imagem de capa (`scrape_noticia()`) em cascata de prioridade (implementado 2026-08-06): **(a)** procura o `<a href>` que envolve a `<img>` de capa no HTML e aponta direto para o arquivo original no CMS (`msconecta.com.br/images/noticias/...`), sem passar pelo proxy de imagens do site; **(b)** se não achar, extrai o parâmetro `src=`/`url=` de dentro de uma URL do proxy `load.websg.app.br` (em `og:image`, `twitter:image` ou `<img src>`), com URL-decode e tratamento de `&amp;`; **(c)** se nada bater (ex: notícia sem imagem de capa própria, vídeo embutido), cai no comportamento legado (`og:image`/seletores de `<img>`, forçando `w=2400&h=1600` quando a URL é do proxy WEBSG). Depois, baixa essa URL em até 3 estágios de fallback: download direto → proxy público (`codetabs`) → PC Windows via Tailscale (`http://100.108.170.120:8765/fetch-binary`).
     **Nota**: o arquivo original (caminho a/b) costuma ter *menos* pixels nominais que a versão antiga forçada em `w=2400&h=1600` pelo proxy, porque o proxy fazia upscale artificial (interpolação, sem detalhe real) de fotos pequenas do CMS — isso mascarava o tamanho real da imagem. Com a URL original, o pipeline mede o tamanho real da imagem antes do crop.
  2. **Claude Vision** (`claude-haiku-4-5-20251001`) analisa a imagem (reduzida a 512×512) para detectar o ponto focal ideal: texto relevante em placas/cartazes (prioridade máxima) → rosto humano (com heurística de escolha entre múltiplos rostos) → elemento/sujeito principal.
  3. Recorta e redimensiona a imagem para o canvas de cada formato (Lanczos; suavização em 2 etapas + leve blur quando o fator de escala excede 3x), aplicando "seam carving" (recorte com preservação de conteúdo) quando o corte horizontal for muito agressivo no feed.
  4. Sobrepõe o template gráfico (`moldes/feed.svg`, `moldes/story.svg`) com título (quebra de linha balanceada via programação dinâmica), editoria e identidade visual do MSConecta.
  5. Salva `feed.png`, `story.png`, `legenda.txt` e `meta.json` (parâmetros de crop, usados depois por `corrigir_posicao.py` para ajustes sem reprocessar tudo) em `output/YYYY-MM-DD/EDITORIA_titulo-slug/`.
- **Serviços externos**: Anthropic Claude (Vision), PC Windows via Tailscale (fallback de download de imagem), proxies HTTP públicos (fallback de scraping).
- **Upscale via IA (Real-ESRGAN/Replicate) — removido em 2026-08-16, REINTRODUZIDO em 2026-08-19**: existiu entre 2026-07-05 e 2026-08-16 como etapa antes do crop final (acionada por fator de escala > 1.8x ou nitidez baixa medida na imagem), validada em produção em 2026-08-06. Removida em 08-16 porque, em uso real, estava **borrando as imagens em vez de melhorá-las** (efeito oversmoothed, perda de textura) mesmo em chamadas que a própria API reportava como sucesso. Reintroduzida em 2026-08-19 (`upscale_imagem_ia()`/`medir_nitidez()`/`_log_upscale_ia()` restauradas em `gerar_tudo.py`) com três proteções contra o efeito "plástico" original: `face_enhance: False` (era a causa raiz mais provável — GFPGAN interno "restaura"/alisa pele de forma agressiva), limiar de acionamento subido de 1.8x para 2.5x, e nova `imagem_tem_rosto()` (Haar cascade local) que pula o upscale via IA inteiramente quando detecta rosto na imagem fonte. `REPLICATE_API_TOKEN` está presente e válido em `/etc/msconecta-bot.env` desde 2026-08-19 (chave nova fornecida por Saulo). Falha (token ausente ou qualquer exceção na chamada à API) sempre degrada graciosamente para a imagem original sem upscale — nunca interrompe o pipeline (`try/except` completo em `upscale_imagem_ia()`, confirmado em 2026-08-20). Detalhes completos em `HISTORICO_MUDANCAS.md` (2026-08-16 e 2026-08-19) e em `CLAUDE.md`.

**Pulado sempre no caminho manual (`--manual`) desde 2026-08-28:** `replicate.run()` (dentro de `upscale_imagem_ia()`) não tem timeout próprio — só o download do resultado depois tem (90s) — e falhas reais medidas em produção levaram ~2min10s cada (não os poucos segundos esperados), o suficiente para travar um usuário olhando ativamente a tela do editor visual/aguardando um ajuste por texto no Telegram. Novo parâmetro `permitir_upscale_ia` (propagado por `redimensionar_capa()`/`carregar_fundo()`/`gerar_imagem_story()`/`gerar_imagem_feed()`) é `False` sempre que `main()` roda em modo `--manual` — usado tanto por `editor_visual.py` quanto por `corrigir_posicao.py` (que também chama `gerar_tudo.py --manual` internamente). Cai direto no fallback Lanczos+blur, com tempo total agora determinístico (~2min, dominado pelo download da imagem, não pelo upscale) em vez de indeterminado. O caminho automático (`monitor_noticias.py`/`publicar_instagram.py`) não foi alterado — continua chamando com `permitir_upscale_ia=True`. Ver `HISTORICO_MUDANCAS.md` 2026-08-28 para a investigação completa (incluindo o caso real da notícia POLÍTICA) e seção 6 para o padrão de risco associado.

**Unsplash — REMOVIDO em 2026-08-16 (mesma sessão)**: `journalist/buscar_imagem.py` tinha Unsplash como fallback secundário de banco de imagem (usado só pelo pipeline do Redator/pautas automáticas, quando a matéria gerada por IA não tem imagem própria — Pexels é a fonte primária, e continua ativo). Removido junto com o upscale por decisão de não manter integrações de terceiros que não agregam valor suficiente; Pexels sozinho já cobre a maior parte dos casos, com fallback gracioso ("Sem imagem") quando nada é encontrado. `UNSPLASH_KEY` removido de `/etc/environment` e `/etc/msconecta-bot.env`.

### 3.4 Monitoramento automático de notícias novas → geração de design

- **Objetivo**: perceber quando uma notícia é publicada no site (por qualquer via — inclusive manual, fora deste pipeline) e gerar o design de redes sociais automaticamente, sem esperar o fluxo completo do pipeline editorial.
- **Arquivo**: `monitor_noticias.py`.
- **Gatilho**: **cron, a cada 30 minutos** (`*/30 * * * * ... monitor_noticias.py`, confirmado na crontab ativa do sistema).
- **Fluxo**: faz scraping da listagem `https://msconecta.com.br/noticias` (com o mesmo esquema de fallback do item 3.3: PC Windows → proxies públicos) → compara com `noticias_vistas.json` para achar notícias ainda não processadas → para cada notícia nova (limite de 5 por ciclo), chama `gerar_design.sh` → notifica o resultado no Telegram com botões (`✅ Postar agora`, `⏰ Programar`, `🖼 Reposicionar`, `⏭ Pular design`) e enfileira a aprovação em `estado.json` (`fila_aprovacao`). Só marca a notícia como "vista" se o design foi gerado com sucesso (falhas são re-tentadas no próximo ciclo).
- **Serviços externos**: os mesmos de `gerar_tudo.py` (seção 3.3), mais Telegram.

### 3.5 Publicação multiplataforma (Instagram, Facebook, Threads) e agendamentos

- **Objetivo**: publicar o design aprovado nas redes sociais, incluindo posts agendados para um horário futuro.
- **Arquivos**: `publicar_instagram.py` (principal), `publicar_multiplatforma.py` (orquestração/relato por plataforma).
- **Gatilho**: **cron, a cada 1 minuto** (`* * * * * ... publicar_instagram.py --processar`, confirmado na crontab ativa) — processa a fila de `agendamentos.json` e publica o que estiver na hora; publicação imediata também é acionada sob demanda pelo bot/pipeline.
- **Fluxo**: lê pasta de output (`feed.png`, `story.png`, `legenda.txt`, `meta.json`) → faz upload das imagens ao **Cloudinary** (a Graph API do Instagram exige uma URL pública, não aceita upload binário direto) → cria container de mídia na Graph API → aguarda processamento → publica → repete para Story (com link sticker) → publica também em Threads e Facebook Page. Também expõe `postar_link_canal_instagram()`, usada para postar o **link da notícia no canal do Instagram** (recurso "canal" do Instagram, separado do feed/story) — via chamada HTTP `POST http://100.108.170.120:8765/postar-canal` (automação no PC Windows, timeout 70s), disparada por AMBOS os caminhos de publicação: o loop `--processar` do cron E a publicação manual `--pasta ... --agora` (chamada por `orquestrador.py:acao_publicar_feed()` quando o editor aprova "postar agora" no Telegram).
- **Modo de falha conhecido — thread daemon fire-and-forget sem join (identificado e corrigido em 2026-08-11)**: `postar_link_canal_instagram()` era disparada numa `threading.Thread(daemon=True)` sem `.join()`, para não bloquear os passos seguintes (story/Threads/Facebook) enquanto aguarda a resposta do PC Windows. Threads daemon em Python são **encerradas abruptamente, sem exceção nem log, quando o processo principal termina** — se a chamada HTTP ao PC (até 70s) ainda estivesse em andamento no momento em que `publicar_feed_e_story()` retornava e o script `publicar_instagram.py` encerrava, a postagem no canal era descartada silenciosamente: nem sucesso nem erro aparecem em lugar nenhum. Isso é diferente de uma exceção engolida — o `try/except` dentro da função funciona corretamente, o problema é a thread nunca chegar a executá-lo até o fim. Na prática isso quase nunca acontecia (a chamada normalmente responde em poucos segundos, bem menos que o tempo gasto nos passos seguintes), mas se tornou visível durante uma janela de instabilidade do túnel Tailscale com o PC (reconexão apos reboot do PC as 2026-08-10 22:45, recuperação completa confirmada as 2026-08-11 09:37): a notícia 15751 ("Canal Rural premia histórias do campo brasileiro") teve o feed publicado normalmente às 15:15:01 UTC de 2026-08-11 mas o link no canal nunca saiu, sem nenhum log de erro — confirmado comparando `instagram.log` (que mostra "Publicando feed..." seguido direto de "Publicando story..." sem a linha "Link postado no canal Instagram" no meio, presente em todas as outras execuções do dia). **Fix**: a referência da thread agora é guardada e, antes de `publicar_feed_e_story()` retornar, chama-se `canal_thread.join(timeout=75)` — o processo não consegue mais encerrar (e matar a thread) enquanto a postagem no canal ainda está em voo; se mesmo assim ainda estiver viva após o timeout, um aviso é logado (`  Aviso: postagem no canal Instagram ainda em andamento apos timeout de espera`). **Lacuna de observabilidade que permanece**: o caminho manual `--agora` (via Telegram) é chamado por `orquestrador.py:executar()` com `subprocess.run(capture_output=True)` — todo o stdout/stderr do `publicar_instagram.py`, incluindo logs de sucesso/erro do canal, fica só na variável Python e NUNCA é gravado em `instagram.log` nem em nenhum arquivo em disco (só os últimos 400 caracteres aparecem no Telegram, e só se o script retornar código de saída != 0, o que uma falha isolada do canal não causa, já que ela nunca derruba o `sucesso` geral). Ou seja: publicações manuais aprovadas no Telegram têm o status da postagem no canal essencialmente invisível para auditoria posterior — considerar no futuro persistir esse log em disco também no caminho `--agora`.

**Reconfirmação em 2026-08-13 (novo boot/reconexão do PC, sem regressão)**: após um novo ciclo de desligar/religar do PC Windows em 2026-08-10 22:45 (mesma janela de instabilidade do Tailscale já descrita acima, recuperada às 09:37 de 08-11), foi reportado suspeita de que o disparo do link no canal tivesse parado de novo. Reconstruído e auditado `instagram.log` (incluindo rotacionados) de 2026-08-11 00:00 UTC até 2026-08-13 16:00 UTC: das 68 publicações de feed nesse intervalo, 67 tiveram o link do canal confirmado com sucesso (`Link postado no canal Instagram: ...`) — a única exceção é a notícia 15751, já identificada e reenviada com sucesso na correção original de 08-11 (ver acima). Ou seja, o fix do `.join(timeout=75)` continua efetivo e nenhuma nova publicação ficou sem o link do canal desde então; nenhum código foi alterado nesta verificação. **Discrepância registrada, não resolvida**: uma investigação feita do lado do PC Windows (sessão separada) reportou que `publicador.log` (log do `story_listener_novo.ps1`, porta 8765) estava com zero atividade desde 2026-08-11 01:31 — o que é inconsistente com a VPS receber `{"ok": true}` do mesmo endpoint (`/postar-canal`) repetidamente depois desse horário, inclusive minutos antes desta verificação. Não foi possível reconciliar isso a partir da VPS (sem acesso ao PC); hipótese mais provável é que a observação no PC tenha sido sobre um arquivo de log desatualizado/errado, ou capturada antes do Tailscale estabilizar de fato. Vale conferir no PC qual processo realmente atende `/postar-canal` e se `publicador.log` é o log correto desse processo.
- **Serviços externos**: Cloudinary, Graph API do Instagram (`graph.instagram.com`), Graph API do Facebook, API do Threads, PC Windows via Tailscale (postagem no canal do Instagram, automação de browser — `http://100.108.170.120:8765/postar-canal`).

### 3.6 Geração e publicação de Reels

- **Objetivo**: transformar vídeos virais curados (ou enviados manualmente ao bot) em Reels no formato 9:16 com identidade visual do MSConecta (título sobreposto, marca d'água, fontes, logo).
- **Arquivos**: `reel_handler.py` (orquestração, lado Python/VPS), `/opt/msconecta-reels/server.js` (API de renderização Node/Express, roda tanto na VPS quanto no PC Windows — ver ressalva abaixo), `/opt/msconecta-reels/src/compositions/ReelMSConecta.tsx` (template Remotion usado pela renderização local na VPS), `monitor_videos.py` (curadoria de candidatos).
- **Gatilho**: `monitor_videos.py` roda para sugerir vídeos (agendamento não identificado com certeza — ver seção 6); a geração/postagem do Reel em si é disparada por aprovação do editor no Telegram (`acao_programar_reel` em `orquestrador.py`, ou fluxo de vídeo enviado direto ao bot em `reel_handler.iniciar_fluxo_reel`).
- **Cadeia real de fallback (corrigido em 2026-08-13 — a descrição anterior, "VPS primária, PC fallback", estava invertida e usava o motor errado)**: `reel_handler._render_via_windows()` tenta **primeiro** o PC Windows (`http://100.108.170.120:3001`, rotas `/upload` → `/render` → polling em `/status/:jobId` → `/output/:filename`) — **não** é Remotion: o motor de render nesse servidor é **FFmpeg puro** (upload via multer, overlay de título via FFmpeg), apesar do nome do diretório/serviço sugerir Remotion. Só se o PC estiver offline (health-check) ou o render no PC falhar/estourar o timeout de polling, cai para o **fallback 2: Remotion na própria VPS** (`render_local_remotion()`, `http://127.0.0.1:3001`, serviço `msconecta-api`, usa de fato o template `ReelMSConecta.tsx`). Se esse também falhar, cai para o **fallback 3: FFmpeg básico na VPS** (`render_ffmpeg_basico()`) — título sobreposto via `drawtext`, sem overlay/marca d'água/logo do template Remotion; é o vídeo de qualidade visivelmente inferior que sai publicado quando os dois primeiros falham.
- **Timeout de polling do PC Windows (bug corrigido em 2026-08-13)**: `_render_via_windows()` aguardava no máximo 6 minutos (72×5s) pelo `/status/:jobId`. Renders reais desse pipeline levam entre ~7min28s e ~8min42s (confirmado cruzando logs da VPS com logs do PC em duas execuções, 2026-08-12 e 2026-08-13) — ou seja, o timeout estourava sistematicamente antes do render real terminar, o handler desistia sem nunca reconsultar depois, declarava falha falsa do PC no Telegram, acionava o fallback 2 (Remotion na VPS, que está com `ETIMEDOUT` — ver ponto seguinte) e terminava sempre no fallback 3 (FFmpeg básico) — o vídeo de baixa qualidade publicado era, na prática, o resultado normal e não uma exceção. **Fix**: timeout de polling aumentado para 12 minutos (144×5s), com margem sobre o pior caso observado, mais uma reconsulta final a `/status/:jobId` após o timeout estourar (cobre o caso do render terminar poucos segundos depois do limite de polling). Ver `HISTORICO_MUDANCAS.md` 2026-08-13 para detalhes e arquivo afetado.
- **Fallback 2 (Remotion na VPS) — status do bug `ETIMEDOUT` de 2026-08-13 incerto, não reproduzido em 2026-08-25**: `render_local_remotion()` chegou a falhar com `spawnSync /bin/sh ETIMEDOUT` após ~10 minutos em pelo menos duas execuções (12/08 e 13/08). Causa raiz nunca foi investigada. Reinvestigado em 2026-08-25 (contexto: diagnóstico do "PC offline" no fluxo de Reels — ver `HISTORICO_MUDANCAS.md`): nas duas tentativas de Reel desse dia, o PC Windows estava genuinamente offline na porta 3001 e o fallback 2 **funcionou normalmente** (`Render local OK`, ~23s de vídeo, sem ETIMEDOUT). Os logs rotacionados disponíveis só cobrem 08-24/08-25, então não é possível afirmar se o bug foi corrigido em algum momento intermediário ou se é intermitente — só que não reproduziu nos dados disponíveis. Se reaparecer, checar `logs/reel.log` por `ETIMEDOUT` e investigar a causa raiz (nunca foi feito).
- **Queda recorrente da porta 3001 no PC Windows — causa raiz identificada e corrigida em 2026-08-25 (ver seção 6 para o registro completo)**: as quedas repetidas de `render_server_pc` observadas ao longo de 2026-08-25 (diagnóstico inicial e depois confirmadas ao vivo pela ativação do health-check) tinham causa raiz dupla no PC Windows: `EADDRINUSE` não tratado em `server.js` combinado com 4 Scheduled Tasks redundantes (`MSConectaRender`, `MSConectaRenderBoot`, `MSConectaRenderPerm`, `StartRenderAutoStart`) competindo pela mesma porta, mais `MSConectaProcessWatchdog` estruturalmente incapaz de agir (`RunLevel Limited`, não conseguia confirmar identidade de processo `SYSTEM`). `server.js` agora trata `EADDRINUSE` explicitamente (log + `process.exit(1)`, sem crash mudo); as três Scheduled Tasks redundantes foram desativadas, restando só `StartRenderAutoStart` como fonte de verdade (com a config de self-healing migrada e o bug de `ExecutionTimeLimit=PT72H` corrigido para `PT0S`); `MSConectaProcessWatchdog` elevado para `RunLevel Highest`. Validado end-to-end com kill externo real do processo (recuperação automática em ~2min).
- **Publicação: `subprocess.run` sem try/except na chamada a `publicar_reel.py` (bug corrigido em 2026-08-13, terceira ocorrência do mesmo padrão — ver seção 6)**: `reel_handler.publicar_reel_instagram()._pub()` chamava `subprocess.run(['python3', 'publicar_reel.py', ...], timeout=300, ...)` sem nenhum `try/except` ao redor, dentro de uma `threading.Thread(daemon=True)`. Se o processo estourasse o timeout (`subprocess.TimeoutExpired`) ou lançasse qualquer outra exceção, a thread daemon morria silenciosamente — sem log, sem aviso no Telegram — deixando o usuário esperando indefinidamente sem saber se o reel foi publicado ou não. **Fix**: `subprocess.run` envolvido em `try/except` capturando `subprocess.TimeoutExpired` especificamente (e `Exception` genérica como rede de segurança), sempre logando E enviando mensagem ao Telegram em caso de falha — nunca falha em silêncio. Timeout aumentado de 300s para **900s** (publicação real no Instagram inclui upload Cloudinary + criação de container + polling de processamento da Graph API, que pode levar vários minutos). Ver `HISTORICO_MUDANCAS.md` 2026-08-13 para detalhes.
- **Aviso de duração acima do limite de elegibilidade de Reels (2026-08-13)**: a Graph API do Instagram documenta ~90s como limite de elegibilidade para um vídeo ser publicado como Reel (acima disso pode virar post comum). `publicar_reel.py` agora loga a duração do vídeo (via `ffprobe`) antes do upload, com aviso caso exceda 90s (não bloqueia a publicação). `reel_handler.py` faz a mesma checagem logo após o render (`processar_confirmacao_reel()._render()`, sobre o vídeo já renderizado), e se exceder o limite avisa o usuário via Telegram **antes** da etapa de aprovação/publicação — para que ele saiba com antecedência que o vídeo pode não sair como Reel, em vez de só descobrir depois de tentar publicar.
- **Serviços externos**: Anthropic Claude (curadoria de vídeos e geração de título/legenda), Remotion (fallback 2, renderização local na VPS, sem custo externo), Graph API do Instagram (publicação via `publicar_reel.py`).

### 3.7 Editor visual de posicionamento de imagem

- **Objetivo**: interface web para o editor ajustar visualmente o ponto focal/zoom de uma imagem já gerada, sem precisar digitar comandos de texto ao bot.
- **Arquivo**: `editor_visual.py` (FastAPI), front-end em `static/editor.html`.
- **Gatilho**: processo persistente via systemd (`msconecta-editor.service`), acessado via link único com token temporário (`editor_tokens.py`, TTL de 4h) enviado pelo bot; exposto publicamente via nginx em `https://<host>/editor` (proxy para `127.0.0.1:8090`).
- **Fluxo**: editor abre o link → ajusta crop/zoom na UI → o backend chama `corrigir_posicao.py` para reprocessar a imagem sem repetir o scraping/Vision → notifica o resultado de volta ao Telegram.
- **Endpoint `/api/editor/{token}/imagem`**: serve a imagem original para o editor visualizar (cache local em `imagem_preview.jpg`/`.png` na pasta da notícia; se ausente, chama `gerar_tudo.baixar_imagem()` — mesma cadeia de fallback usada na geração: direto → proxy `codetabs` → PC Windows via Tailscale → Playwright como último recurso). O token (TTL de 4h) **não é single-use** — `validar_token()` só invalida por expiração, nunca por uso; página HTML, `/dados` e `/imagem` podem ser chamados quantas vezes forem necessárias com o mesmo token. `nginx` usa `proxy_read_timeout 340s` neste bloco (ver `HISTORICO_MUDANCAS.md` 2026-08-20) para cobrir o pior caso da cadeia de download completa.

**Endpoint `/api/editor/{token}/confirmar`**: dispara `gerar_tudo.py --manual` numa thread daemon em background (pode levar ~2min, dominado pelo download da imagem — ver seção 3.3, upscale via IA sempre pulado nesse caminho desde 2026-08-28). **Trava contra geração concorrente duplicada (desde 2026-08-28)**: `_geracoes_em_andamento`/`_geracoes_lock` em `editor_visual.py` rejeitam (`HTTP 409`) uma segunda confirmação para a mesma pasta enquanto a primeira ainda está rodando, em vez de deixar duas gerações escreverem em paralelo sobre os mesmos `feed.png`/`story.png`/`meta.json` — caso real que causou a notícia POLÍTICA travar em 2026-08-28 (ver `HISTORICO_MUDANCAS.md`). **Efeito colateral conhecido, não corrigido**: `_atualizar_estado()` (chamada ao fim de cada geração bem-sucedida) atualiza `estado.json['ultima_pasta']` mas não `ultima_url`/`ultima_legenda` — os três campos podem ficar dessincronizados entre si, e `orquestrador.py:acao_programar()` (comando "programa para as HH:MM" no Telegram, sem URL explícita) agenda com base em `ultima_pasta` — se o editor visual for usado nos minutos antes de um comando "programar" sem argumento, o agendamento pode acabar mirando a notícia errada. Ver `HISTORICO_MUDANCAS.md` 2026-08-28 para o caso real (agendamento da POLÍTICA por engano do horário pretendido para outra notícia).

### 3.8 Relatórios e monitoramento de saúde

| Automação | Arquivo | Frequência (documentada) |
|---|---|---|
| Relatório diário de publicações | `relatorio_diario.py` | Segundo `MANUAL_OPERACIONAL.md`, 18h Campo Grande (21h UTC) — **não confirmado na crontab atual** |
| Relatório semanal | `relatorio_semanal.py` | Segundas 9h, segundo o próprio docstring — **não confirmado na crontab atual** |
| Monitor de saúde dos serviços (alerta só em mudança de estado) | `monitor_saude.py` | Docstring afirma "a cada 5 minutos via cron" — **não confirmado na crontab atual** |
| Métricas de engajamento do Instagram | `metricas_instagram.py` | Não identificado |
| Dashboard web em tempo real | `dashboard.py` | Processo persistente (systemd `msconecta-dashboard`), porta 8085 |

Ver seção 6 sobre a divergência entre a documentação (`MANUAL_OPERACIONAL.md`) e a crontab real do sistema.

### 3.9 Pipeline de produção editorial (dashboard, Fases 0-1 — desde 2026-08-14)

- **Objetivo**: dar visão consolidada (Kanban por estágio, planner do dia, saúde do sistema) de tudo que os JSONs de produção já registram, sem que o Telegram deixe de ser o painel de aprovação. Arquitetura completa, decisões e justificativas em `ROADMAP_PIPELINE_DASHBOARD.md` neste repositório — aqui só o resumo operacional.
- **Arquivos**: `pipeline_schema.sql` (schema), `pipeline_lib.py` (parsing compartilhado), `migrar_dados_iniciais.py` (bootstrap único, já rodado), `sync_pipeline_db.py` (sincronização incremental contínua), `health_check_pipeline.py` (health-check ativo), `dashboard_pipeline.py` (frontend FastAPI+htmx).
- **`sync_pipeline_db.py`**: cron a cada 5 minutos, lê `agendamentos.json`/`estado.json`/`videos_sugestoes.json` e faz upsert em `msconecta.db` — nunca escreve nos JSONs, nenhum script de produção foi alterado para chamá-lo ou depender dele.
- **`health_check_pipeline.py`**: checa identidade de processo (não só porta) — na VPS via `systemctl` + `/proc/<pid>/cmdline` (msconecta-bot/api/editor/dashboard); no PC Windows via assinatura de resposta HTTP sobre Tailscale (`story_listener_novo.ps1` porta 8765, servidor de render porta 3001), tratando timeout de rede como `desconhecido` (não `falha`) para não confundir instabilidade do Tailscale com processo caído. **Ativado no cron em 2026-08-25** (`*/5 * * * *`, `logs/health_check_pipeline_cron.log`). Nessa mesma ativação foi acrescentado `_cruzar_status_pc()`: como os dois serviços do PC estão no mesmo IP/Tailscale, quando um responde `ok` e o outro dá timeout no mesmo ciclo, o timeout é reclassificado de `desconhecido` para `falha` (rede comprovadamente saudável no instante, então só pode ser o processo) — validado com um caso real (`render_server_pc` caiu de novo às 18:15 UTC no próprio teste de ativação, corretamente marcado como `falha`, não `desconhecido`; ver `HISTORICO_MUDANCAS.md`).
- **Alerta via Telegram na mudança de estado (2026-08-25, item (b) da seção 6, agora implementado)**: `_detectar_transicoes()` compara o status desta checagem com o último status já persistido em `system_health` para cada serviço (consultado antes do insert do ciclo atual) — dispara alerta só quando o estado muda (`ok`→`desconhecido`/`falha` = queda, `desconhecido`/`falha`→`ok` = recuperação), nunca a cada ciclo enquanto o problema persiste no mesmo estado. Mesmo padrão de "alerta só em mudança de estado" que `monitor_saude.py` já usa para os serviços da VPS, reaproveitado em vez de reinventado (`monitor_saude.py` usa `saude_estado.json` como fonte do estado anterior; aqui a própria tabela histórica `system_health` cumpre esse papel, sem precisar de um arquivo companheiro novo). Caso especial: quando `story_listener_pc` **e** `render_server_pc` estão `desconhecido` no mesmo ciclo (timeout simultâneo, não coberto por `_cruzar_status_pc()` porque nenhum dos dois respondeu `ok`), os alertas individuais são suprimidos e um único alerta conjunto "PC Windows/Tailscale possivelmente offline" é disparado no lugar. O alerta de queda de `render_server_pc` menciona que a causa raiz mais comum (corrida de Scheduled Tasks, ver subseção "watchdog vivo mas estruturalmente incapaz de agir" na seção 6) já foi corrigida em 2026-08-25. Corrigido no mesmo passo um falso-positivo real: `checar_story_listener_pc()` exigia o campo `uptime` além de `ok:true` para classificar como `ok`, causando `falha` espúria quando o serviço respondia `{"ok":true}` sem `uptime` (confirmado em 2026-08-16, `/fetch-url` saudável no mesmo instante) — agora `ok:true` sozinho já é suficiente. Validado com testes unitários (mock de HTTP), simulação dos 4 cenários de transição (queda, recuperação, alerta conjunto de PC, não-repetição) e envio real de teste ao Telegram (3 mensagens, `HTTP 200 ok=true`); ver `HISTORICO_MUDANCAS.md`.
- **`dashboard_pipeline.py`**: FastAPI + htmx (sem build step de frontend), somente leitura (conexão SQLite em modo `mode=ro`), bind em `127.0.0.1:8095`, systemd `msconecta-pipeline-dashboard.service`. Exposto em **`https://pipeline.korabot.com.br/pipeline`** (`/etc/nginx/sites-available/pipeline`, TLS via Let's Encrypt/certbot, HTTP redireciona automaticamente para HTTPS), protegido por autenticação básica (`/etc/nginx/.htpasswd_pipeline`) — credenciais informadas a Saulo via Telegram, nunca em texto plano em nenhum arquivo. **O acesso antigo via IP direto (`http://72.60.151.58/pipeline`) foi removido** do bloco `msconecta-designs` em 2026-08-14 (ver `HISTORICO_MUDANCAS.md`) — essa rota hoje cai na SPA (Nexus Panel) que já ocupa a location `/` desse bloco, não expõe mais o dashboard do pipeline. Renovação do certificado via `certbot.timer` (systemd, já habilitado por padrão nesta VPS, cobre todos os domínios `*.korabot.com.br`/`*.nip.io`).

---

## 4. INTEGRAÇÕES E DEPENDÊNCIAS

Não há `requirements.txt` nem `package.json` na raiz de `/root/msconecta` (dependências Python parecem instaladas globalmente/via ambiente do sistema, não em um virtualenv versionado). As bibliotecas abaixo foram identificadas por import direto no código:

### Dependências Python (lado VPS)
| Biblioteca | Uso |
|---|---|
| `requests` | Cliente HTTP para scraping, proxies, APIs |
| `beautifulsoup4` (`bs4`) | Parsing de HTML no scraping de notícias/RSS |
| `Pillow` (`PIL`) | Geração e manipulação das imagens de feed/story |
| `numpy` | Suporte numérico para o `seam-carving` |
| `seam-carving` | Recorte de imagem com preservação de conteúdo (usado no feed) |
| `anthropic` (SDK oficial) | Claude Vision (foco de imagem) e classificação de intenção do bot |
| `fastapi` + `uvicorn` | Servidor do editor visual |

### Dependências Node.js (`/opt/msconecta-reels/package.json`)
| Pacote | Uso |
|---|---|
| `@remotion/cli`, `remotion` | Motor de renderização de vídeo programático |
| `@remotion/google-fonts`, `@remotion/layout-utils` | Suporte de fonte/layout no template de Reel |
| `@anthropic-ai/sdk` | Uso de Claude a partir do lado Node (provavelmente análise de frame/curadoria) |
| `express` | API HTTP interna (`server.js`) |
| `react`, `react-dom` | Runtime de componentes usados pelo Remotion |
| `dotenv` | Carregamento de variáveis de ambiente locais ao app |

### Bancos de dados / storages
- **Os JSONs continuam sendo a fonte de verdade em produção.** O estado é persistido em **arquivos JSON simples** na raiz do projeto: `estado.json` (sessão/fila de aprovação atual), `agendamentos.json` (fila de posts agendados), `historico_publicacoes.json` (log de publicações), `noticias_vistas.json` e `cache_noticiasmetadados.json` (cache do monitor), `metricas_ig.json`, `videos_sugestoes.json`/`videos_vistos.json`, entre outros. Nenhum script de produção foi alterado para escrever num banco — ver ponto abaixo.
- **`msconecta.db` (SQLite, desde 2026-08-14)**: banco de leitura derivado dos JSONs acima, mantido atualizado por `sync_pipeline_db.py` (cron a cada 5 min). Existe para alimentar o dashboard do pipeline (`dashboard_pipeline.py`) sem que nenhum script de produção precise saber que ele existe — arquitetura completa em `ROADMAP_PIPELINE_DASHBOARD.md` (seções 4, 4.1, 4.2) neste mesmo repositório de docs. **Não é a fonte de verdade** — é populado por leitura, nunca o contrário.
- **Cloudinary** é usado como storage de imagens (hospedagem temporária necessária para a Graph API do Instagram).
- **CMS WEBSG** (site) tem seu próprio banco (não acessado diretamente — só via o formulário/endpoint HTTP documentado em `cms_msconecta.md`).

### Variáveis de ambiente esperadas

Definidas em `/root/msconecta/.env` e/ou `/etc/msconecta-bot.env` (carregado como `EnvironmentFile` pelos serviços systemd e pelo cron):

| Variável | Finalidade |
|---|---|
| `ANTHROPIC_API_KEY` | Acesso à API da Anthropic (Claude Vision, classificação de intenção) |
| `GROQ_KEY` | Acesso à API da Groq (redação automática de notícias) |
| `TELEGRAM_BOT_TOKEN` | Token do bot Telegram (também hardcoded em vários scripts — ver seção 6) |
| `TELEGRAM_CHAT_ID` | ID do chat/editor autorizado a interagir com o bot |
| `INSTAGRAM_TOKEN` | Token de acesso à Graph API do Instagram (`graph.instagram.com`) |
| `INSTAGRAM_USER_ID` | ID da conta Instagram do MSConecta |
| `INSTAGRAM_APP_ID` / `INSTAGRAM_APP_SECRET` | Credenciais do app do Instagram (OAuth) |
| `FACEBOOK_SYSTEM_TOKEN` | Token de usuário de sistema do Facebook/Meta (fallback de token para Instagram/Facebook) |
| `FACEBOOK_PAGE_ID` / `FACEBOOK_PAGE_TOKEN` | Página do Facebook e seu token de publicação |
| `THREADS_TOKEN` / `THREADS_USER_ID` | Credenciais de publicação no Threads |
| `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` | Credenciais da conta Cloudinary (hospedagem de imagem) |
| `N8N_API_KEY` / `N8N_BASE_URL` | Acesso à instância n8n usada por outro projeto na mesma VPS |
| `PEXELS_KEY` | Busca de imagem de banco gratuito (fallback quando a notícia não tem imagem própria) |

---

## 5. COMUNICAÇÃO ENTRE VPS E PC SERVIDOR

A VPS (`72.60.151.58`) e o PC Windows (`100.108.170.120`) se comunicam **exclusivamente via HTTP simples sobre a rede privada do Tailscale** (nenhuma fila de mensagens, nenhum banco compartilhado). Não há um único "servidor" do lado do PC — são **dois pequenos serviços HTTP** rodando nele:

1. **`story_listener_novo.ps1`** (PowerShell, `HttpListener` nativo do .NET) — escuta na porta **8765** e expõe rotas como `/status`, `/receber-story` (recebe a imagem e publica o story via `story_poster_windows.py`, que usa Playwright/Chrome com CDP na porta 9222), `/fetch-url` e `/fetch-binary` (proxy de scraping/download usando o IP residencial do PC), e `/buscar-noticia` (extrai `og:image` de uma notícia publicada).
2. **API de renderização de Reels do PC** — servidor Node/Express na porta **3001**, mesmo nome de arquivo (`server.js`) e mesmas rotas (`/upload`, `/render`, `/status/:jobId`, `/output/:filename`) do serviço análogo da VPS (`msconecta-api`), mas **motor de render diferente**: no PC é FFmpeg puro (upload via multer + overlay de título via FFmpeg), não Remotion — apesar do nome sugerir o contrário. É a **primeira via tentada** por `reel_handler.py` (não um fallback de sobrecarga da VPS — ver seção 3.6, corrigido em 2026-08-13).

Além dos serviços HTTP, existe **SSH direto com usuário/senha** do PC (`publicar_via_pc.sh`, via `sshpass`) como via alternativa para publicar no CMS via Playwright, sem depender do listener HTTP — usado como plano B.

### O que roda em cada lado

| VPS (`72.60.151.58`) | PC Windows (`100.108.170.120`) |
|---|---|
| Bot Telegram, orquestrador, pipeline editorial | Chrome headless + Playwright (`publicar.js`, `story_poster_windows.py`) |
| Geração de imagem (`gerar_tudo.py`) | `story_listener_novo.ps1` — servidor HTTP na porta 8765 |
| Publicação nas APIs de rede social | Publicação no CMS via automação de navegador (contorna bloqueio anti-bot do Cloudflare para IPs de datacenter) |
| Renderização de Reels — **fallback 2** de 3, Remotion, porta 3001 local (atualmente com `ETIMEDOUT`, ver seção 3.6); **fallback 3**, FFmpeg básico, sem PC envolvido | Renderização de Reels — **1ª via tentada**, FFmpeg puro (não Remotion), porta 3001 |
| Scraping direto/via proxy público (fallback) | Scraping com IP residencial (via principal, mais confiável) |
| Dashboard, editor visual | — |

---

## 6. PONTOS DE ATENÇÃO

### Padrão recorrente de falha silenciosa em operações assíncronas/bloqueantes (risco sistêmico, registrado em 2026-08-13, 5º caso em 2026-08-16, 6º caso em 2026-08-28)
Seis bugs distintos, todos no mesmo padrão:
- **2026-08-11** — `threading.Thread(daemon=True)` disparada sem `.join()` em `publicar_instagram.py` (postagem do link no canal do Instagram descartada silenciosamente se o processo principal terminasse antes da thread).
- **2026-08-13 (cedo)** — timeout de polling sem reconsulta final em `reel_handler._render_via_windows()` (render real terminava depois do timeout de polling estourar, handler desistia e declarava falha falsa em vez de checar mais uma vez).
- **2026-08-13 (este registro)** — `subprocess.run()` sem `try/except` em `reel_handler.publicar_reel_instagram()._pub()` (timeout ou qualquer outra exceção matava a thread daemon sem log nem aviso ao Telegram).
- **2026-08-16** — `subprocess.run()` sem `try/except` em `orquestrador.py:executar()` (helper compartilhado por quase todas as `acao_*` do bot) — timeout de `gerar_tudo.py` no fluxo manual de link colado no Telegram derrubava `orquestrador.py` inteiro (rc=1, traceback em `logs/orq_stderr.log`), sem nenhuma mensagem chegar ao usuário. Agravado por um segundo bug no mesmo mecanismo: `telegram_bot.py:_worker_texto()` só reportava erro ao Telegram se o `stdout` do processo estivesse vazio, mas os logs (`log()`) já tinham escrito linhas em stdout antes do crash — a condição nunca disparava. Ver `HISTORICO_MUDANCAS.md` (2026-08-16) para os detalhes e evidência (log + traceback).
- **2026-08-16 (5º caso, mesmo dia, no PC Windows)** — `tailscale_watchdog.ps1` chamava `tailscale down`/`tailscale up` sem nenhum timeout; quando `tailscale up` travava, o script ficava preso indefinidamente, segurando o pipe IPC do Tailscale e deixando processos zumbis — de fora (VPS), isso era indistinguível de uma queda real do PC (Tailscale "offline", rx=0 mesmo com tx crescendo). Corrigido por Saulo diretamente no PC: nova função `Invoke-TailscaleComTimeout` (`C:\MSConecta\tailscale_watchdog.ps1`, linhas 145-186), rodando os comandos via `Start-Job`+`Wait-Job -Timeout` (20s down / 25s up) e matando `tailscale.exe` (processo CLI, preciso — não `tailscaled`/`tailscale-ipn`) se o timeout estourar, com log `TIMEOUT:` distinto do caminho feliz. Esta é a causa raiz real por trás do incidente descrito na subseção seguinte — a conclusão inicial de "PC fora do ar" descrevia o sintoma, não a causa.
- **2026-08-28 (6º caso)** — `replicate.run()` dentro de `upscale_imagem_ia()` (`gerar_tudo.py`) nunca teve timeout próprio (só o download do resultado depois de completar tem, 90s); no caminho manual do editor visual (usuário olhando a tela ativamente), uma chamada lenta/degradada do Replicate consumia minutos de forma imprevisível, estourando o timeout de 300s do `subprocess.run()` em `editor_visual.py` sem deixar rastro do motivo real (só "timeout ao gerar design"). Combinado com a ausência de trava contra geração concorrente (mesma sessão, achado relacionado — ver logo abaixo), causou pelo menos 3 notícias travadas num único dia, incluindo uma escrita parcial (`story.png` novo, `feed.png`/`meta.json` velhos) em 2 delas. **Fix**: upscale via IA passou a ser sempre pulado no caminho manual (`permitir_upscale_ia=False`), independente de rosto/nitidez — decisão consciente de não depender de nenhum timeout numa API de terceiros para um fluxo onde o usuário espera ativamente. Ver seção 3.3 e `HISTORICO_MUDANCAS.md` 2026-08-28.

Os seis casos têm a mesma forma: uma operação que roda em paralelo ao fluxo principal (thread daemon) ou tem um limite de tempo (timeout de rede/subprocess) sem que o código trate explicitamente o caminho de "não terminou a tempo" ou "levantou exceção" — o resultado é sempre o mesmo, falha invisível para o usuário, sem log e sem mensagem no Telegram, só perceptível por auditoria manual de logs depois do fato. **Não é uma correção pontual, é um sinal de risco sistêmico**: outras threads daemon e outras chamadas com timeout no projeto (`reel_handler.py`, `gerar_tudo.py`, `monitor_noticias.py`, etc.) devem ser revisadas com essa lente antes de assumir que estão corretas, mesmo sem um bug relatado ainda. Recomendação para revisões futuras: toda `threading.Thread(daemon=True)` que faz uma operação com efeito observável pelo usuário deveria ser aguardada (`.join()`) ou, se realmente precisa ser fire-and-forget, ter seu próprio tratamento de exceção que sempre loga e avisa; toda chamada com `timeout=` (subprocess, requests) deveria estar dentro de um `try/except` que trata o caso de estouro como um evento de primeira classe (log + aviso), não como algo que "não deveria acontecer".

### Ausência de trava contra geração concorrente no editor visual — CORRIGIDO em 2026-08-28

`editor_visual.py:editor_confirmar()` disparava `_gerar_em_background()` numa `threading.Thread(daemon=True)` sem nenhum controle de concorrência — se o usuário reabria o link do editor e confirmava de novo (típico quando a espera parece travada, o que a causa raiz da subseção anterior tornava comum), duas gerações rodavam em paralelo sobre os mesmos `feed.png`/`story.png`/`meta.json`. Caso real: notícia POLÍTICA "TSE e Google..." em 2026-08-28, confirmada duas vezes com 5min14s de diferença (mesmo token) — nenhuma das duas chegou a escrever arquivo algum (ambas mortas pelo timeout de 300s, provavelmente por contenção dobrada na chamada ao Replicate). Duas outras notícias no mesmo dia (CAMPO GRANDE, AGRONEGÓCIO "agro-gera-mais-de-30-mil") tiveram o mesmo tipo de escrita parcial, uma delas também com confirmação dupla confirmada em log. **Fix**: `_geracoes_em_andamento`/`_geracoes_lock` (set + lock em nível de módulo) rejeitam a segunda tentativa para a mesma pasta com `HTTP 409` e mensagem clara, em vez de deixar rodar em paralelo. Validado com teste real via o endpoint de produção (duas confirmações disparadas 0.3s uma da outra: primeira aceita, segunda rejeitada). Ver seção 3.7 e `HISTORICO_MUDANCAS.md` 2026-08-28.

**Efeito colateral descoberto no processo de correção, não resolvido — dessincronismo entre `ultima_pasta`/`ultima_url`/`ultima_legenda` em `estado.json`:** `editor_visual.py:_atualizar_estado()` só atualiza `ultima_pasta` (e `atualizado`) ao fim de uma geração pelo editor visual — nunca `ultima_url`/`ultima_legenda`. `orquestrador.py:acao_programar()` (comando "programa para as HH:MM" sem URL explícita) agenda com base em `ultima_pasta`. Se o editor visual for usado nos minutos antes de um comando "programar" sem argumento — mesmo para uma notícia sem relação com o que o usuário está de fato processando no momento na fila do Telegram —, o agendamento pode acabar mirando a notícia errada, silenciosamente (o bot avisa o nome da pasta agendada via Telegram, mas nada impede o usuário de não notar num fluxo de comandos rápidos). Ocorreu de fato em 2026-08-28 durante o teste de recuperação da POLÍTICA desta sessão (ver `HISTORICO_MUDANCAS.md`). Não corrigido — possível fix futuro: `_atualizar_estado()` também gravar `ultima_url`/`ultima_legenda` a partir do `meta.json` da pasta, mantendo os três campos sempre consistentes entre si.

### Padrão recorrente de risco operacional: PC Windows parecendo fora do ar sem alerta automático (registrado em 2026-08-16, causa corrigida no mesmo dia)
Distinto do padrão de código acima — este é um risco de **infraestrutura/processo** percebido a partir da VPS, não necessariamente um bug corrigível só nela. Pelo menos três incidentes já documentados neste projeto tiveram a mesma causa-gatilho aparente — o PC Windows (`100.108.170.120`) parecer fora do ar do ponto de vista da VPS (desligado, Tailscale caído, ou processo do listener morto) — mas cada um só foi percebido horas depois, por sintomas diferentes e indiretos:
- **2026-08-10/11** — reboot do PC + janela de instabilidade do Tailscale causou perda pontual do link no canal do Instagram (percebido só por auditoria manual de log).
- **2026-08-13** — render de Reel via PC demorando mais que o timeout de polling, mascarado por uma cadeia de fallback que sempre terminava na versão de pior qualidade (percebido pela qualidade do vídeo publicado, não por alerta).
- **2026-08-16** — PC Windows pareceu **offline no Tailscale por pelo menos 19h seguidas** (confirmado via `tailscale status`, "last seen" sem avançar mesmo após novas tentativas de conexão da VPS) sem nenhum alerta chegar a Saulo. Efeito em cascata: (a) o fluxo manual de link colado no Telegram estourava o timeout de `gerar_tudo.py` (ver bug de código acima); (b) `monitor_noticias.py` (cron a cada 30min) ficou **incapaz de sequer ler a listagem do site** desde as 09:00 (PC timeout + os 2 proxies públicos de fallback retornando erro 522/520 do Cloudflare, provavelmente pela mesma janela de instabilidade) — ou seja, o sistema ficou "cego" para notícias novas por horas, sem nenhuma notificação disso; Saulo só soube da queda ao tentar usar o bot manualmente. Adicionalmente, ao tentar validar a recuperação do PC, descoberto que `health_check_pipeline.py` (criado exatamente para detectar esse tipo de situação, Fase 1 do pipeline, 2026-08-14) tinha um bug de schema (`CHECK` do banco não incluía o status `'desconhecido'` que o próprio script gera para timeout de rede) que o fazia **falhar por completo e não gravar nada** sempre que o PC estivesse instável — o pior momento possível para essa ferramenta falhar (corrigido nesta mesma sessão, ver `HISTORICO_MUDANCAS.md`). **Causa raiz real (apurada depois por Saulo direto no PC, não era queda física/de rede):** o próprio `tailscale_watchdog.ps1` travava numa chamada `tailscale up` sem timeout, segurando o pipe IPC e deixando processos zumbis — corrigido no mesmo dia (ver 5º caso do padrão de falha silenciosa, subseção acima). Recuperado e revalidado no mesmo dia: `health_check_pipeline.py` voltou a reportar os serviços do PC, `/fetch-url` respondeu com HTML válido em teste real, e o cron de `monitor_noticias.py` recuperou sozinho o backlog de notícias represado desde as 09:00 assim que a coleta voltou a funcionar.

Nota de ferramenta (achado incidental em 2026-08-16, **corrigido em 2026-08-25**): o `/status` do `story_listener_pc` responde `{"ok":true,"status":"..."}`, às vezes **sem** o campo `uptime` que `health_check_pipeline.py:checar_story_listener_pc()` exigia para classificar como `"ok"` — na prática, o serviço saudável era às vezes classificado como `"falha"` por assinatura incompleta, não por estar realmente fora do ar (confirmado em 2026-08-16: `/status` "falha" ao mesmo tempo em que `/fetch-url` respondia perfeitamente). Corrigido ao implementar o alerta via Telegram (ver seção 3.9): `"ok":true` sozinho já é suficiente para `"ok"`, `uptime` ausente vira só uma nota informativa.

**Recomendação — item (a) e item (b) implementados em 2026-08-25**: até 2026-08-25 não existia nenhum alerta automático quando o PC Windows fica inacessível — a detecção dependia de alguém tentar usar uma funcionalidade dependente dele e notar a falha (ou rodar `health_check_pipeline.py` manualmente). (a) **Feito**: o cron de `health_check_pipeline.py` foi aprovado/ativado a cada 5min como planejado na Fase 1 (ver seção 3.9), com o histórico agora sendo gravado continuamente em `system_health`/visível no dashboard. (b) **Feito** (mesmo dia, sessão seguinte): alerta via Telegram na mudança de estado (`ok`→`desconhecido`/`falha` e vice-versa) para todos os serviços de `health_check_pipeline.py`, reaproveitando o mesmo padrão de "alerta só em mudança de estado" de `monitor_saude.py` — detalhes em seção 3.9 e `HISTORICO_MUDANCAS.md`.

**Nota (2026-08-25):** a causa raiz específica por trás da queda recorrente de `render_server_pc`/porta 3001 que motivou boa parte da investigação deste dia foi identificada e corrigida no PC Windows no mesmo dia — ver subseção "Padrão recorrente: watchdog 'vivo' mas estruturalmente incapaz de agir" logo abaixo. A frequência dessa queda específica deve cair para praticamente zero daqui em diante; o alerta do item (b) acima cobre qualquer falha futura não coberta por esse fix pontual (inclusive uma eventual recorrência por causa diferente).

### Condição de corrida entre process_watchdog.ps1 e boot_reconciliation.ps1 no boot do PC Windows — REGISTRADA E RESOLVIDA em 2026-08-20

No PC Windows existem (pelo menos) duas Scheduled Tasks que agem sobre a mesma Scheduled Task alvo, `MSConectaPublicadorPM2` (supervisão do processo PM2 responsável pela publicação): `process_watchdog.ps1` (trigger `AtLogon`, roda em loop contínuo supervisionando o processo) e `boot_reconciliation.ps1` (trigger `AtStartup`, reconciliação pontual logo após o boot). Detectado em **2026-08-20 às 11:04:34** que os dois disparavam próximo o suficiente do boot para agir **simultaneamente** sobre `MSConectaPublicadorPM2` — condição de corrida entre dois processos concorrentes tentando reconciliar o mesmo estado, mesma categoria de risco já mapeada na subseção acima (operação sem coordenação explícita de quem é responsável por agir em qual janela de tempo).

**Fix aplicado por Saulo diretamente no PC** (arquivo vive só no PC, fora do controle de versão desta sessão — só documentado aqui): `process_watchdog.ps1` agora checa `(Get-Date) - LastBootUpTime` no início de cada ciclo do seu loop principal; se o boot foi há menos de 5 minutos, o script só loga e dorme, sem checar nem agir sobre nenhuma task nesse ciclo — deixando `boot_reconciliation.ps1` como único responsável pela reconciliação inicial nessa janela. Passado esse período de 5 minutos, `process_watchdog.ps1` volta a agir normalmente.

**Testado:** sintaxe válida (parser do PowerShell), lógica do guard validada isoladamente com valores simulados (2.3min desde o boot → pula corretamente a ação; 7.8min → segue o comportamento normal), e confirmado que não interfere no comportamento atual da máquina (boot há 63.8min no momento do teste, fora da janela de guard). Aplicado em produção via reinício da Scheduled Task `MSConectaProcessWatchdog` (não foi necessário esperar o próximo boot para o fix entrar em vigor).

**Status: resolvido.** Ver `HISTORICO_MUDANCAS.md` (2026-08-20) para o registro da mudança.

### Padrão recorrente: watchdog "vivo" mas estruturalmente incapaz de agir (2º caso — causa raiz da queda recorrente da porta 3001/render_server_pc, RESOLVIDO em 2026-08-25)

Segunda ocorrência da mesma classe de bug já registrada com `tailscale_watchdog.ps1` (2026-08-16, subseção "Padrão recorrente de falha silenciosa..." acima): um processo watchdog que continua "vivo" nos logs/Task Manager, dando falsa sensação de proteção ativa, mas que na prática é estruturalmente incapaz de agir quando precisa. No primeiro caso a causa era falta de timeout numa chamada externa (`tailscale up` travando indefinidamente); neste segundo caso a causa é privilégio insuficiente.

**Causa raiz original, dupla:**
1. **`EADDRINUSE` não tratado em `server.js`** (o mesmo binário Node/Express citado na seção 3.6, roda tanto na VPS quanto no PC Windows): 4 Scheduled Tasks redundantes no PC (`MSConectaRender`, `MSConectaRenderBoot`, `MSConectaRenderPerm`, `StartRenderAutoStart`) competiam pela mesma porta 3001, causando uma corrida de inicialização que crashava o processo sem log estruturado.
2. **`MSConectaProcessWatchdog` cego**: parado há 3 dias e, mesmo religado, rodando com `RunLevel Limited` — estruturalmente incapaz de confirmar identidade de processo `SYSTEM` via `CommandLine` (sempre "inconclusiva", nunca agia).

**Correções aplicadas no PC Windows (validadas com evidência de log real):**
1. `server.js`: handler para o evento `'error'` do `http.Server`, distinguindo `EADDRINUSE` de outros erros — sempre `process.exit(1)` em vez de crash mudo. Validado com colisão de porta deliberada.
2. Scheduled Tasks consolidadas: `MSConectaRender`, `MSConectaRenderBoot` e `MSConectaRenderPerm` **desativadas** (não removidas); `StartRenderAutoStart` é agora a única fonte de verdade, com a configuração de self-healing de `RenderPerm` (`RunLevel Highest`, `RestartCount=999`, `RestartInterval=1min`) migrada para ela — corrigindo no caminho um bug latente (`ExecutionTimeLimit` estava em `PT72H`, corrigido para `PT0S`).
3. `MSConectaProcessWatchdog`: `RunLevel` elevado de `Limited` para `Highest`, resolvendo a cegueira estrutural.

**Achado sobre o Task Scheduler do Windows (vale para qualquer watchdog futuro neste PC):** `RestartOnFailure` não dispara quando o processo é morto por `TerminateProcess`/`Stop-Process -Force` — o Scheduler registra isso como "concluído com sucesso, código 4294967295" (Event ID 201), não como falha. O retry nativo do Task Scheduler só cobre o processo se auto-encerrando com erro (o `EADDRINUSE` do item 1); kill externo só é coberto pelo watchdog com privilégio elevado (item 3) — mecanismos complementares, não redundantes.

**Validado end-to-end** com kill externo real do processo: morto às 14:44:20, watchdog agiu às 14:46:09–12, identidade confirmada às 14:46:20 — recuperação automática completa em ~2min, sem intervenção manual, confirmada por 3 fontes independentes (log do watchdog, Event Log do Task Scheduler, PID novo via `Get-NetTCPConnection`).

**Status: resolvido.** Ver `HISTORICO_MUDANCAS.md` (2026-08-25) para o registro completo da mudança e da validação.

### Segurança / credenciais em texto plano no código-fonte
Vários scripts têm **segredos hardcoded diretamente no código**, em vez de lidos exclusivamente de variáveis de ambiente — risco caso o repositório seja exposto ou compartilhado:
- O **token do bot Telegram e o chat ID do editor** estão hardcoded, repetidos, em pelo menos `telegram_bot.py`, `orquestrador.py`, `pipeline.py`, `notificar_pautas.py`, `relatorio_diario.py`, `relatorio_semanal.py`, `monitor_noticias.py`, `monitor_videos.py`, `reel_handler.py`, `editor_visual.py` e `journalist/redator.py` — em vez de um único ponto de configuração (embora `portal_config.py` já defina um mecanismo de fallback centralizado, ele não é usado por todos esses arquivos).
- `post_story_agora.py` define tokens do Instagram/Cloudinary diretamente em `os.environ.setdefault(...)` no topo do arquivo.
- `publicar_via_pc.sh` contém uma **senha em texto plano** para SSH no PC Windows (via `sshpass`).
- `/etc/profile.d/msconecta.sh` exporta tokens/segredos em texto plano para qualquer shell de login na VPS.

Recomenda-se migrar todos esses pontos para `portal_config.py` + variáveis de ambiente centralizadas, e revogar/rotacionar os tokens expostos.

### Documentação desatualizada em relação à automação real
`MANUAL_OPERACIONAL.md` (última atualização registrada: 2026-05-18) descreve um crontab com `monitor_bloco1.py`, `monitor_bloco2.py`, `notificar_pautas.py` e `relatorio_diario.py` agendados em horários fixos. **Esse crontab não existe na crontab ativa do sistema no momento deste mapeamento** — apenas `monitor_noticias.py` (a cada 30 min) e `publicar_instagram.py --processar` (a cada minuto) estão confirmados via cron, além de rotinas de outro cliente (Funky Fresh) e um `systemctl restart msconecta-api` semanal. Não foi possível confirmar se blocos de pauta, relatórios e monitor de saúde ainda rodam automaticamente, se migraram para outro mecanismo (por exemplo, um agente de IA com agendamento próprio, mencionado en passant no crontab para outro cliente — "agora orquestrado pelo n8n"), ou se passaram a ser só de execução manual. Vale confirmar com quem opera o sistema.

### Inconsistência de porta no nginx do dashboard
`dashboard.py` abre `HTTPServer(('0.0.0.0', 8085), ...)`, mas a configuração do nginx (`/etc/nginx/sites-available/msconecta-designs`) faz `proxy_pass` de `/dashboard` para `127.0.0.1:8082` — uma porta diferente. Ou o nginx está desatualizado, ou há outro processo (não localizado neste mapeamento) escutando em 8082.

### Grande volume de arquivos de backup manual no diretório raiz
Dezenas de arquivos `.bak`, `.bak_YYYYMMDD_HHMMSS` de scripts centrais (`gerar_tudo.py.bak_*`, `orquestrador.py.bak_*`, `monitor_noticias.py.bak_*`, `corrigir_posicao.py.bak_*` etc.) se acumulam na raiz do projeto — sinal de que não há controle de versão (git) sendo usado como rede de segurança; os backups manuais cumprem esse papel de forma frágil. Não foi encontrado um repositório git em `/root/msconecta`.

### Pasta compartilhada entre projetos não relacionados
`/root/msconecta` mistura os scripts do portal de notícias com automação de um cliente de restaurante ("Funky Fresh": `atualizar_cardapio.sh`, `cardapio_funky_fresh.json`, `funky_metricas.py`, `renovar_token_funky.py`, entre outros). Isso aumenta o risco de alterações num projeto afetarem acidentalmente o outro (e já é fonte de ruído na leitura da crontab e da listagem de arquivos).

### Retenção de output
`limpar_output.py` remove pastas de `output/` com mais de 30 dias — mas **não há evidência de que esse script esteja agendado** (não aparece na crontab nem em systemd). `output/` já ocupa ~2,5 GB (82 pastas de datas), então vale confirmar se a limpeza está de fato rodando periodicamente.

### Monitoramento e logs existentes
- Logs de aplicação em `logs/` (`telegram_bot.log`, `pipeline.log`, `pautas.log`, `relatorio.log`, `reel.log`, entre outros) e na raiz (`instagram.log`, `monitor.log`), rotacionados via `logrotate` (`/etc/logrotate.d/msconecta` — diário, mantém 14 dias, compressão).
- `dashboard.py` oferece um painel web em tempo real com status dos serviços/integrações (porta 8085, autenticado por token em `.dashboard_token` para o endpoint de ações).
- `monitor_saude.py` implementa alerta de mudança de estado (OK↔falha) dos serviços via Telegram, mas seu agendamento atual não foi confirmado (ver acima).
