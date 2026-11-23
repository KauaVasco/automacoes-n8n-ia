# Assistente de E-mails com IA e aprovação no Telegram (n8n)

Três workflows n8n que leem e-mails novos do Gmail, ignoram os automáticos, escrevem uma resposta com IA e só enviam depois que você aprova no Telegram. Se você recusar, é só escrever no chat como a resposta deve ser, e o assistente reescreve até você aceitar.

> **Ambiente:** feito e testado em `localhost`, com n8n 2.48 rodando no Docker (`n8nio/n8n` mais `n8nio/runners`) e um túnel ngrok para o Telegram alcançar o webhook. É um projeto pessoal, não uma instância de produção.

## O que mudou nesta versão

A versão anterior tinha dois workflows (agendamento de reuniões e automação de e-mails) e a aprovação ficava num nó de espera do Telegram. Esse nó abre uma página web para quem recusa, o que obrigava a expor o n8n por um link e não funcionava para digitar a instrução direto no chat. Esta versão refaz isso:

- A aprovação acontece inteira no chat do Telegram, com botões **Sim, enviar** e **Não**. Depois do **Não**, o bot pergunta o que mudar e você responde na própria conversa.
- Um workflow de triagem separa o que é automático do que merece resposta.
- O agendamento de reuniões ficou de fora. Os arquivos antigos continuam no histórico do Git.

## Como funciona

Três workflows, cada um com uma tarefa:

| Arquivo | O que faz |
|---|---|
| `triagem.json` | Vigia a caixa de entrada, descarta e-mails automáticos e chama a redação para os outros. |
| `redacao-e-aprovacao.json` | Lê o e-mail e a conversa, escreve a resposta e manda o pedido de aprovação ao Telegram. |
| `telegram.json` | Recebe os toques nos botões e o texto que você digita, envia a resposta ou pede a reescrita. |

### Triagem

O Gmail Trigger consulta a caixa a cada minuto, só na caixa de entrada e sem as abas Promoções, Social e Fóruns. Cada e-mail novo passa por uma checagem por regras, escrita em Python:

- **Sinais fortes**, que descartam o e-mail: cabeçalho `Auto-Submitted`, `Precedence: bulk/list/junk`, `List-Unsubscribe` ou `List-Id`, `X-Auto-Response-Suppress`, relatório de entrega (`multipart/report`), `Return-Path` vazio, remetente igual a um dos seus endereços e endereços do tipo `noreply`, `donotreply`, `mailer-daemon` e `postmaster`.
- **Sinais prováveis**, como remetente com cara de notificação, categoria Atualizações do Gmail ou frases de aviso automático no assunto e no corpo. Esses e-mails seguem para o Telegram com um aviso ⚠️, e você decide.

Toda decisão fica registrada na tabela `emails_processados`, e um e-mail já registrado nunca é lido de novo. Quando o assistente envia uma resposta, o id dessa mensagem também entra na tabela. Sem isso, a resposta que volta para a sua caixa de entrada seria lida como e-mail novo e o assistente responderia a si mesmo.

### Redação e aprovação

Para cada e-mail que passou, o workflow busca as últimas cinco mensagens da conversa e baixa os anexos. PDFs e imagens seguem para o Gemini, que lê o conteúdo, e planilhas viram texto antes de ir para a IA. Anexos em `.docx` não são lidos. Esse caminho dos anexos está montado, mas ainda não foi testado com arquivos reais.

O modelo principal é o Gemini, com o OpenRouter como reserva se o principal falhar. A resposta sai em formato estruturado: tipo do e-mail, se precisa de resposta, o motivo e o texto. O prompt traz regras de estilo (parágrafos curtos, tom natural, resposta logo na primeira frase) e uma lista do que denuncia texto de máquina: contraste "não é X, é Y", frase de efeito no fim, travessão, três itens por ritmo, palavras infladas e fórmulas como "fico à disposição". Ele também traz uma proteção contra instruções escondidas dentro do e-mail. Se a IA decide que não precisa responder, o workflow só registra o motivo.

Quando precisa responder, ele salva a pendência na tabela `aprovacoes` e manda ao Telegram: quem escreveu, o assunto, o que o e-mail diz e "Irei responder assim", com os dois botões.

Quando o remetente pede algo que só o dono decide (confirmar um prazo, aceitar, recusar, passar valor ou data) e o e-mail não traz essa decisão, o rascunho não responde por conta própria. Ele deixa um marcador entre colchetes, como `[confirmar: dia]`, e a mensagem do Telegram começa com um aviso ⚠️. O **Sim** envia o texto exatamente como está, então, se houver colchetes, toque em **Não** e diga o que colocar.

### Telegram

Só o chat configurado consegue aprovar.

- **Sim, enviar:** responde na mesma conversa do Gmail, só ao remetente, com exatamente o texto aprovado.
- **Não:** o bot pergunta como a resposta deve ficar. Você escreve a instrução (ou `ignorar` para descartar), o texto antigo volta para a IA junto com a instrução e uma nova versão chega com os botões. As instruções se acumulam, até o limite de cinco por e-mail.
- Botão de uma mensagem antiga responde "Esse pedido já foi tratado".

O campo `modoSimulacao` do nó `Config` vem ligado: o **Sim** registra o envio mas não manda e-mail. Desligue quando quiser enviar de verdade.

As leituras do Gmail tentam de novo até três vezes, porque o Gmail às vezes derruba a conexão.

## Configuração

Cada workflow tem um nó `Config` logo no início, o único lugar para editar seus dados:

| Workflow | Campo | O que colocar |
|---|---|---|
| Triagem | `meusEmails` | Seus endereços, separados por vírgula. E-mails enviados por eles são ignorados. |
| Redação | `nomeUsuario` | O nome que assina a resposta. |
| Redação e Telegram | `telegramChatId` | O ID do seu chat com o bot. |
| Telegram | `modoSimulacao` | `true` para testar sem enviar, `false` para enviar. |

Os arquivos deste repositório já vêm com placeholders (`voce@exemplo.com`, `Seu Nome`, `SEU_CHAT_ID_TELEGRAM`) no lugar dos dados reais.

## Como importar

Cada workflow também traz um bloco "Como Instalar" na própria tela. O resumo:

1. No n8n, vá em **Workflows**, **Import from File** e importe os três arquivos.
2. Conecte as credenciais: Gmail (OAuth2), Telegram, Google Gemini e OpenRouter. Os arquivos trazem só os nomes que cada nó espera.
3. Crie as duas Data Tables abaixo e escolha cada uma nos nós que usam tabela.
4. Preencha o nó `Config` de cada workflow.
5. Nos nós `Dispara Redação e Aprovação` (triagem) e `Dispara Redação` (telegram), escolha o workflow *Redação e Aprovação* que você importou. O n8n gera um ID novo na importação, então essa referência precisa ser refeita.
6. Ative primeiro a **Redação e Aprovação**, depois a Triagem e o Telegram. O n8n não deixa publicar um workflow cujo sub-workflow está inativo.
7. O n8n precisa de um endereço HTTPS alcançável pelo Telegram, porque o bot entrega as mensagens por webhook. Um túnel como o ngrok resolve para uso local.

### Data Tables

`emails_processados`, todas as colunas em texto, exceto `tentativas` (número):
`message_id`, `thread_id`, `remetente`, `assunto`, `decisao`, `motivo`, `tentativas`.

`aprovacoes`, todas em texto, exceto `tentativa` (número):
`email_id`, `thread_id`, `remetente`, `assunto`, `corpo`, `instrucoes`, `tentativa`, `estado`, `sinal`, `motivo`.

O campo `estado` percorre `aguardando_aprovacao`, `aguardando_instrucao`, `reescrevendo`, `finalizado` e `pendente`.

## Limites conhecidos

- Os Code nodes são em Python nativo, que precisa do task runner com Python (a imagem `n8nio/runners` tem).
- O Gmail Trigger consulta a cada minuto, então a resposta ao Telegram não é instantânea.
- Modelos gratuitos do Gemini e do OpenRouter podem usar o conteúdo enviado para treino. Para e-mails de clientes, use uma conta paga ou revise os termos.
- E-mails que você manda de um endereço listado em `meusEmails` são ignorados. Para testar com um e-mail seu, mande de um endereço que não esteja na lista.

## Aviso

O ciclo de e-mail novo, triagem, rascunho, aprovação, reescrita e envio foi testado com Gmail, Telegram, Gemini e OpenRouter. Compartilhado como referência e portfólio: ajuste os prompts e as regras ao seu caso antes de usar com e-mails de verdade.

## Licença

[MIT](LICENSE). Use, copie e adapte à vontade, mantendo os créditos.
