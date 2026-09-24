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

## 7. Como usar este documento

Cada sessão que for trabalhar numa fase deve ler este documento inteiro
antes de começar, confirmar qual fase está em andamento via
`HISTORICO_MUDANCAS.md`, e nunca implementar nada que contradiga a seção 2
(aprovação manual intocável), mesmo que pareça uma otimização óbvia de
throughput.
