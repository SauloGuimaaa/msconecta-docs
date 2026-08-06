# Histórico de mudanças — MSConecta

Formato de cada entrada:
## YYYY-MM-DD — título curto da mudança
- **O que mudou:** descrição objetiva
- **Arquivos/serviços afetados:** lista
- **Motivo:** por que a mudança foi feita
- **Autor:** Claude Code / Saulo / outro

---

## 2026-08-06 — Validação em produção do upscale via IA após a mudança de extração de imagem
- **O que mudou:** nenhuma mudança de código — validação da mudança anterior (extração da imagem de capa via `<a href>`, commit `4097312`) no ambiente real de produção, motivada pelo risco de que imagens originais menores (abaixo de 1080px de largura) passassem a depender do Real-ESRGAN/Replicate para não sair borradas, caminho que antes praticamente não era acionado. Confirmado: (1) `REPLICATE_API_TOKEN` está presente e válido em `/etc/msconecta-bot.env` (o `.env` real usado pelo cron — `/root/msconecta/.env`, usado em execução manual, não tem essa chave); (2) rodado `gerar_tudo.py` de ponta a ponta nesse ambiente real para a notícia 15636 (imagem original 900×675): Real-ESRGAN concluiu em 9,7s, `feed.png`/`story.png` com qualidade visual boa; (3) o próprio cron, já em produção com a mudança, gerou mais 2 designs reais no mesmo período com upscale bem-sucedido (`logs/upscale_ia.log`, 0 falhas); (4) confirmado que falha do Replicate nunca derruba o pipeline (try/except em `upscale_imagem_ia()`, fallback de suavização Lanczos+blur só acima de `fator_escala > 3.0`), mas NÃO existe alerta ativo (Telegram/monitoramento) quando o upscale falha — só um log passivo (`logs/upscale_ia.log`) que ninguém observa proativamente; o único fator mitigante é a aprovação manual obrigatória no Telegram antes de qualquer publicação.
- **Arquivos/serviços afetados:** nenhum código alterado nesta entrada. Serviço externo confirmado: Replicate (Real-ESRGAN), via `/etc/msconecta-bot.env`.
- **Motivo:** garantir que a mudança de extração de imagem (commit `4097312`) não introduziu um ponto de falha silenciosa em produção, já que ela aumenta a frequência com que o upscale via IA é necessário. Recomendação registrada: a mudança está segura para continuar rodando no cron como está (pipeline nunca quebra, degrada para resize simples em vez de travar), mas é recomendável adicionar um aviso na notificação do Telegram quando `upscale_imagem_ia()` falhar ou o fallback de baixa resolução for acionado, como melhoria de observabilidade (não bloqueante).
- **Autor:** Claude Code

---

## 2026-08-06 — Extração da imagem de capa passa a priorizar o arquivo original (via `<a href>`) em vez do proxy de resize
- **O que mudou:** `scrape_noticia()` em `gerar_tudo.py` agora extrai a URL da imagem de capa em cascata de prioridade: (a) `<a href>` que envolve a `<img>` de capa no HTML, apontando direto pro arquivo original no CMS (`msconecta.com.br/images/noticias/...`); (b) se não achar, o parâmetro `src=`/`url=` de dentro de uma URL do proxy `load.websg.app.br` (og:image/twitter:image/img src), com URL-decode e tratamento de `&amp;`; (c) fallback para o comportamento antigo (og:image/seletores de `<img>`, com `w=2400&h=1600` forçado na URL do proxy) se nada acima bater. Duas funções auxiliares novas: `_extrair_capa_via_ancora()` e `_extrair_capa_via_proxy_param()`. O download em 3 estágios (direto → proxy `codetabs` → PC Windows via Tailscale) em `baixar_imagem()` não foi alterado, só passou a receber uma URL de entrada melhor.
- **Arquivos/serviços afetados:** `gerar_tudo.py` (função `scrape_noticia()` e duas funções novas). `monitor_noticias.py` não precisou de mudança — só dispara `gerar_tudo.py URL` via subprocess, sem lógica própria de extração de imagem.
- **Motivo:** o proxy de imagens do CMS (`load.websg.app.br`) recomprime a foto (conversão para webp/jpg) e, em fotos cujo arquivo original é menor que 2400×1600, faz upscale artificial por interpolação simples para atingir esse tamanho — sem detalhe real adicional. Isso mascarava o tamanho real da imagem e fazia o critério geométrico de acionamento do upscale via IA (Real-ESRGAN, `LIMIAR_UPSCALE_IA = 1.8x`) nunca disparar para essas fotos, já que a versão "grande" do proxy parecia suficientemente grande. Testado em 3 notícias reais (editorias PANTANAL/educação, política e economia): o arquivo original tem, sim, menos pixels nominais e menos bytes que a versão inflada pelo proxy (ex: 900×675/183 KB vs 2400×1600/442 KB), mas ao usar o original o pipeline mede o tamanho real e aciona corretamente o Real-ESRGAN quando necessário — reconstrução de detalhe por IA em vez de upscale genérico do proxy, além de evitar uma recompressão extra (dupla perda de qualidade JPEG/WEBP).
- **Autor:** Claude Code

---

## 2026-08-06 — Criação do contexto inicial e histórico versionado
- **O que mudou:** mapeamento completo do projeto gerado e pasta de documentação estruturada como repositório git.
- **Arquivos/serviços afetados:** nenhum código alterado, apenas documentação nova.
- **Motivo:** ter uma fonte única de verdade sobre a estrutura do projeto, sempre atualizada, para orientar mudanças futuras.
- **Autor:** Claude Code
