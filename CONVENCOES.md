# Convenções obrigatórias para mudanças no MSConecta

Sempre que qualquer mudança for feita neste projeto (código, configuração, cron, systemd, nginx, variáveis de ambiente, infraestrutura):

1. Atualize a seção correspondente em `CONTEXTO_MSCONECTA.md` para refletir o novo estado real (não deixe o documento ficar desatualizado como o MANUAL_OPERACIONAL.md estava).
2. Adicione uma nova entrada no topo de `HISTORICO_MUDANCAS.md`, seguindo o formato definido no próprio arquivo.
3. Faça commit no git desta pasta (`/root/msconecta-docs/`) com mensagem descritiva, ANTES de considerar a tarefa concluída.
4. Nunca inclua credenciais, tokens ou senhas em nenhum desses arquivos, mesmo que estejam em texto plano no código-fonte.
