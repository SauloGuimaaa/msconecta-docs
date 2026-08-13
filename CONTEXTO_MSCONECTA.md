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
| IA — imagem (upscale) | Real-ESRGAN via Replicate (modelo `nightmareai/real-esrgan`, versão fixada) |
| Web scraping | `requests` + `BeautifulSoup4` |
| CMS do site | WEBSG v5.6.3 (CMS de terceiros, PHP, acessado via automação de formulário/POST — ver `cms_msconecta.md`) |
| APIs de redes sociais | Graph API do Instagram (`graph.instagram.com`), Graph API do Facebook (`graph.facebook.com`), API do Threads |
| Hospedagem de imagem para posts | Cloudinary (upload das imagens geradas antes de enviar para a Graph API) |
| Editor visual web | FastAPI + Uvicorn (`editor_visual.py`), servido na porta 8090 |
| Dashboard de monitoramento | Servidor HTTP nativo do Python (`http.server`), porta 8085 |
| Automação de terceiros | n8n (self-hosted, usado para pelo menos um fluxo — calendário editorial de outro cliente hospedado na mesma VPS, ver observação abaixo) |
| Proxy/servidor web | nginx |
| Processos persistentes | systemd (4 serviços dedicados, ver seção 2) |

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
│   ├── buscar_imagem.py           # Busca imagem via Pexels/Unsplash quando a pauta não tem imagem própria
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
├── logs/                         # Logs das automações (telegram_bot.log, pipeline.log, pautas.log, relatorio.log, upscale_ia.log, reel.log)
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
├── .env                             # Variáveis de ambiente locais ao projeto (não versionado — ver seção 4)
├── CLAUDE.md                        # Notas técnicas profundas sobre a lógica de geração de imagem (algoritmo de foco, quebra de linha, upscale)
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
| `monitor_noticias.py` | Cron, a cada 30 minutos |
| `publicar_instagram.py --processar` | Cron, a cada 1 minuto |
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
- **Serviços externos**: Groq (redação), CMS WEBSG do site (publicação da notícia), Claude/Anthropic (Vision para posicionamento de imagem, dentro de `gerar_tudo.py`), Replicate (upscale de imagem via Real-ESRGAN, quando necessário), Instagram/Facebook/Threads Graph API, Cloudinary (hospedagem temporária das imagens para as APIs de rede social), Telegram (aprovações).

### 3.3 Geração automática de design de notícia (feed + story)

- **Objetivo**: gerar as duas imagens padrão de post (`feed.png` 1080×1350, `story.png` 1080×1920) a partir do título, editoria e imagem de capa de uma notícia, com posicionamento de foco automático via visão computacional.
- **Arquivo**: `gerar_tudo.py` (1725 linhas — lógica detalhada e mantida em `CLAUDE.md`), auxiliado por `corrigir_posicao.py` para ajustes incrementais.
- **Gatilho**: chamado por `monitor_noticias.py` (automático), por `pipeline.py` (fluxo aprovado), ou manualmente via `gerar_design.sh URL` / comando ao bot.
- **Passo a passo resumido** (detalhe completo em `CLAUDE.md`):
  1. Extrai a URL da imagem de capa (`scrape_noticia()`) em cascata de prioridade (implementado 2026-08-06): **(a)** procura o `<a href>` que envolve a `<img>` de capa no HTML e aponta direto para o arquivo original no CMS (`msconecta.com.br/images/noticias/...`), sem passar pelo proxy de imagens do site; **(b)** se não achar, extrai o parâmetro `src=`/`url=` de dentro de uma URL do proxy `load.websg.app.br` (em `og:image`, `twitter:image` ou `<img src>`), com URL-decode e tratamento de `&amp;`; **(c)** se nada bater (ex: notícia sem imagem de capa própria, vídeo embutido), cai no comportamento legado (`og:image`/seletores de `<img>`, forçando `w=2400&h=1600` quando a URL é do proxy WEBSG). Depois, baixa essa URL em até 3 estágios de fallback: download direto → proxy público (`codetabs`) → PC Windows via Tailscale (`http://100.108.170.120:8765/fetch-binary`).
     **Nota**: o arquivo original (caminho a/b) costuma ter *menos* pixels nominais que a versão antiga forçada em `w=2400&h=1600` pelo proxy, porque o proxy fazia upscale artificial (interpolação, sem detalhe real) de fotos pequenas do CMS — isso mascarava o tamanho real da imagem e fazia o critério geométrico de upscale via IA (`LIMIAR_UPSCALE_IA = 1.8x`, passo 3 abaixo) nunca disparar para essas fotos. Com a URL original, o pipeline mede o tamanho real e aciona corretamente o Real-ESRGAN quando necessário, em vez de depender só do upscale genérico do proxy.
  2. **Claude Vision** (`claude-haiku-4-5-20251001`) analisa a imagem (reduzida a 512×512) para detectar o ponto focal ideal: texto relevante em placas/cartazes (prioridade máxima) → rosto humano (com heurística de escolha entre múltiplos rostos) → elemento/sujeito principal.
  3. Se a imagem for pequena/borrada demais para o crop necessário, roda **upscale via IA** (Real-ESRGAN/Replicate) antes do crop final.
  4. Recorta e redimensiona a imagem para o canvas de cada formato, aplicando "seam carving" (recorte com preservação de conteúdo) quando o corte horizontal for muito agressivo no feed.
  5. Sobrepõe o template gráfico (`moldes/feed.svg`, `moldes/story.svg`) com título (quebra de linha balanceada via programação dinâmica), editoria e identidade visual do MSConecta.
  6. Salva `feed.png`, `story.png`, `legenda.txt` e `meta.json` (parâmetros de crop, usados depois por `corrigir_posicao.py` para ajustes sem reprocessar tudo) em `output/YYYY-MM-DD/EDITORIA_titulo-slug/`.
- **Serviços externos**: Anthropic Claude (Vision), Replicate (Real-ESRGAN), PC Windows via Tailscale (fallback de download de imagem), proxies HTTP públicos (fallback de scraping).
- **Validação em produção da extração via `<a href>` (2026-08-06)**: confirmado que `REPLICATE_API_TOKEN` está presente e válido em `/etc/msconecta-bot.env` (o `.env` real sourceado pelo cron do `monitor_noticias.py`/`publicar_instagram.py` — `/root/msconecta/.env`, usado só em execução manual, NÃO tem essa chave). Rodado o pipeline completo nesse ambiente real: Real-ESRGAN acionado com sucesso (`900×675 → 1800×1350` em 9,7s) para a notícia 15636 (imagem original abaixo de 1080px de largura), `story.png`/`feed.png` resultantes com qualidade visual boa (sem blur/pixelização perceptível). O próprio cron, já rodando com a mudança em produção desde o commit, gerou mais 2 designs reais no mesmo intervalo com upscale bem-sucedido (ver `logs/upscale_ia.log`) — 0 falhas registradas até o momento.
  - **Tratamento de falha do Replicate**: `upscale_imagem_ia()` nunca derruba o pipeline (try/except cobre toda chamada à API) — em caso de falha, retorna a imagem original e grava uma linha "FALHA" em `logs/upscale_ia.log`. Só existe suavização de fallback (Lanczos 2 passos + blur leve) quando `fator_escala > 3.0`; entre 1.8x–3.0x, uma falha do Replicate resulta em resize Lanczos simples sem tratamento extra (equivalente à qualidade de antes desta mudança).
  - **Gap identificado**: essa falha não gera alerta ativo (Telegram ou monitoramento de log) — `monitor_noticias.py` roda `gerar_tudo.py` com `subprocess.run(capture_output=True)` e só repassa 4 linhas específicas de stdout (`OUTPUT_PATH`/`IMG_URL`/`EDITORIA`/`LEGENDA_B64`); avisos de upscale ficam só no log passivo. O único fator mitigante é que nada publica automaticamente: toda geração passa por aprovação manual no Telegram (botões "Postar agora"/"Pular design") antes de ir pro Instagram/Facebook/Threads, então um humano vê a imagem antes da publicação — mas essa checagem depende de reparar visualmente o blur numa preview pequena, não é um alerta direcionado. Recomendado como melhoria futura (não bloqueante): incluir aviso na notificação do Telegram quando `upscale_imagem_ia()` falhar ou o fallback de baixa resolução (`fator_escala > 3.0`) for acionado.

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
- **Fallback 2 (Remotion na VPS) atualmente quebrado — `ETIMEDOUT` (pendência não corrigida, 2026-08-13)**: `render_local_remotion()` falha com `spawnSync /bin/sh ETIMEDOUT` após ~10 minutos, consistentemente (confirmado em pelo menos duas execuções, 12/08 e 13/08, `logs/reel.log`). Causa raiz não investigada — problema separado do timeout do PC Windows, de menor prioridade, registrado aqui só para não ser confundido com o fix acima. Enquanto não for corrigido, todo render cujo PC Windows falhar de verdade (não só timeout curto) cai direto no fallback 3 (FFmpeg básico, sem overlay/logo).
- **Publicação: `subprocess.run` sem try/except na chamada a `publicar_reel.py` (bug corrigido em 2026-08-13, terceira ocorrência do mesmo padrão — ver seção 6)**: `reel_handler.publicar_reel_instagram()._pub()` chamava `subprocess.run(['python3', 'publicar_reel.py', ...], timeout=300, ...)` sem nenhum `try/except` ao redor, dentro de uma `threading.Thread(daemon=True)`. Se o processo estourasse o timeout (`subprocess.TimeoutExpired`) ou lançasse qualquer outra exceção, a thread daemon morria silenciosamente — sem log, sem aviso no Telegram — deixando o usuário esperando indefinidamente sem saber se o reel foi publicado ou não. **Fix**: `subprocess.run` envolvido em `try/except` capturando `subprocess.TimeoutExpired` especificamente (e `Exception` genérica como rede de segurança), sempre logando E enviando mensagem ao Telegram em caso de falha — nunca falha em silêncio. Timeout aumentado de 300s para **900s** (publicação real no Instagram inclui upload Cloudinary + criação de container + polling de processamento da Graph API, que pode levar vários minutos). Ver `HISTORICO_MUDANCAS.md` 2026-08-13 para detalhes.
- **Aviso de duração acima do limite de elegibilidade de Reels (2026-08-13)**: a Graph API do Instagram documenta ~90s como limite de elegibilidade para um vídeo ser publicado como Reel (acima disso pode virar post comum). `publicar_reel.py` agora loga a duração do vídeo (via `ffprobe`) antes do upload, com aviso caso exceda 90s (não bloqueia a publicação). `reel_handler.py` faz a mesma checagem logo após o render (`processar_confirmacao_reel()._render()`, sobre o vídeo já renderizado), e se exceder o limite avisa o usuário via Telegram **antes** da etapa de aprovação/publicação — para que ele saiba com antecedência que o vídeo pode não sair como Reel, em vez de só descobrir depois de tentar publicar.
- **Serviços externos**: Anthropic Claude (curadoria de vídeos e geração de título/legenda), Remotion (fallback 2, renderização local na VPS, sem custo externo), Graph API do Instagram (publicação via `publicar_reel.py`).

### 3.7 Editor visual de posicionamento de imagem

- **Objetivo**: interface web para o editor ajustar visualmente o ponto focal/zoom de uma imagem já gerada, sem precisar digitar comandos de texto ao bot.
- **Arquivo**: `editor_visual.py` (FastAPI), front-end em `static/editor.html`.
- **Gatilho**: processo persistente via systemd (`msconecta-editor.service`), acessado via link único com token temporário (`editor_tokens.py`, TTL de 4h) enviado pelo bot; exposto publicamente via nginx em `https://<host>/editor` (proxy para `127.0.0.1:8090`).
- **Fluxo**: editor abre o link → ajusta crop/zoom na UI → o backend chama `corrigir_posicao.py` para reprocessar a imagem sem repetir o scraping/Vision → notifica o resultado de volta ao Telegram.

### 3.8 Relatórios e monitoramento de saúde

| Automação | Arquivo | Frequência (documentada) |
|---|---|---|
| Relatório diário de publicações | `relatorio_diario.py` | Segundo `MANUAL_OPERACIONAL.md`, 18h Campo Grande (21h UTC) — **não confirmado na crontab atual** |
| Relatório semanal | `relatorio_semanal.py` | Segundas 9h, segundo o próprio docstring — **não confirmado na crontab atual** |
| Monitor de saúde dos serviços (alerta só em mudança de estado) | `monitor_saude.py` | Docstring afirma "a cada 5 minutos via cron" — **não confirmado na crontab atual** |
| Métricas de engajamento do Instagram | `metricas_instagram.py` | Não identificado |
| Dashboard web em tempo real | `dashboard.py` | Processo persistente (systemd `msconecta-dashboard`), porta 8085 |

Ver seção 6 sobre a divergência entre a documentação (`MANUAL_OPERACIONAL.md`) e a crontab real do sistema.

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
| `replicate` (SDK oficial) | Upscale de imagem via Real-ESRGAN |
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
- **Não há banco de dados relacional/NoSQL tradicional.** O estado é persistido em **arquivos JSON simples** na raiz do projeto: `estado.json` (sessão/fila de aprovação atual), `agendamentos.json` (fila de posts agendados), `historico_publicacoes.json` (log de publicações), `noticias_vistas.json` e `cache_noticiasmetadados.json` (cache do monitor), `metricas_ig.json`, `videos_sugestoes.json`/`videos_vistos.json`, entre outros.
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
| `UNSPLASH_KEY` | Idem, fallback secundário de banco de imagem |

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

### Padrão recorrente de falha silenciosa em operações assíncronas/bloqueantes (risco sistêmico, registrado em 2026-08-13)
Três bugs distintos, todos no mesmo padrão, corrigidos em menos de uma semana:
- **2026-08-11** — `threading.Thread(daemon=True)` disparada sem `.join()` em `publicar_instagram.py` (postagem do link no canal do Instagram descartada silenciosamente se o processo principal terminasse antes da thread).
- **2026-08-13 (cedo)** — timeout de polling sem reconsulta final em `reel_handler._render_via_windows()` (render real terminava depois do timeout de polling estourar, handler desistia e declarava falha falsa em vez de checar mais uma vez).
- **2026-08-13 (este registro)** — `subprocess.run()` sem `try/except` em `reel_handler.publicar_reel_instagram()._pub()` (timeout ou qualquer outra exceção matava a thread daemon sem log nem aviso ao Telegram).

Os três casos têm a mesma forma: uma operação que roda em paralelo ao fluxo principal (thread daemon) ou tem um limite de tempo (timeout de rede/subprocess) sem que o código trate explicitamente o caminho de "não terminou a tempo" ou "levantou exceção" — o resultado é sempre o mesmo, falha invisível para o usuário, sem log e sem mensagem no Telegram, só perceptível por auditoria manual de logs depois do fato. **Não é uma correção pontual, é um sinal de risco sistêmico**: outras threads daemon e outras chamadas com timeout no projeto (`reel_handler.py`, `gerar_tudo.py`, `orquestrador.py`, etc.) devem ser revisadas com essa lente antes de assumir que estão corretas, mesmo sem um bug relatado ainda. Recomendação para revisões futuras: toda `threading.Thread(daemon=True)` que faz uma operação com efeito observável pelo usuário deveria ser aguardada (`.join()`) ou, se realmente precisa ser fire-and-forget, ter seu próprio tratamento de exceção que sempre loga e avisa; toda chamada com `timeout=` (subprocess, requests) deveria estar dentro de um `try/except` que trata o caso de estouro como um evento de primeira classe (log + aviso), não como algo que "não deveria acontecer".

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

### Calibração pendente do critério de upscale por qualidade
Documentado no próprio `CLAUDE.md`: o limiar `LIMIAR_NITIDEZ_BAIXA = 15.0` (usado para decidir se uma imagem "naturalmente borrada" deve passar por upscale via IA mesmo sem necessidade geométrica) foi calibrado sobre um espaço de medição diferente do que o código realmente usa em produção (crop final vs. imagem fonte completa) — o próprio código admite que esse critério "dificilmente vai disparar" como planejado. Não tratar esse valor como validado. Ver [[msconecta_upscale_ia]] para o histórico da implementação do upscale via IA.

### Retenção de output
`limpar_output.py` remove pastas de `output/` com mais de 30 dias — mas **não há evidência de que esse script esteja agendado** (não aparece na crontab nem em systemd). `output/` já ocupa ~2,5 GB (82 pastas de datas), então vale confirmar se a limpeza está de fato rodando periodicamente.

### Monitoramento e logs existentes
- Logs de aplicação em `logs/` (`telegram_bot.log`, `pipeline.log`, `pautas.log`, `relatorio.log`, `upscale_ia.log`, `reel.log`, entre outros) e na raiz (`instagram.log`, `monitor.log`), rotacionados via `logrotate` (`/etc/logrotate.d/msconecta` — diário, mantém 14 dias, compressão).
- `dashboard.py` oferece um painel web em tempo real com status dos serviços/integrações (porta 8085, autenticado por token em `.dashboard_token` para o endpoint de ações).
- `monitor_saude.py` implementa alerta de mudança de estado (OK↔falha) dos serviços via Telegram, mas seu agendamento atual não foi confirmado (ver acima).
