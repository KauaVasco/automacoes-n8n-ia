# Automação de E-mails e Reuniões com IA (n8n)

Dois workflows n8n que trabalham juntos: um lê e-mails e escreve respostas com IA, o outro identifica pedidos de reunião, negocia o horário e agenda tudo automaticamente. Toda ação que sai pro mundo real (enviar e-mail, marcar reunião) passa por aprovação humana no Telegram antes.

> **Ambiente:** feito e testado em `localhost`, rodando n8n via **Docker** (`n8nio/n8n`, SQLite local, sem domínio público, sem reverse proxy). Não é uma instância de produção — é um projeto pessoal/portfólio rodando na própria máquina. Ver [Se fosse rodar fora do localhost](#se-fosse-rodar-fora-do-localhost) pro que mudaria.

## O que mudou nesta versão

Este repositório começou como só a automação de e-mails, respondendo com rascunho + label `Regerar` no Gmail. Agora são dois workflows conectados:

- **`agendamento-reunioes.json`** é o ponto de entrada: todo e-mail novo passa por ele primeiro. Uma IA decide se é um pedido de reunião.
- Se for reunião, ele mesmo cuida de tudo: pergunta o horário se faltar, consulta a agenda, pede aprovação no Telegram e marca no Google Calendar.
- **Se não for sobre reunião**, ele chama `automacao-emails.json` como **sub-fluxo** (Execute Workflow, non-blocking) — que agora responde direto por e-mail em vez de deixar como rascunho, também com aprovação prévia no Telegram.

Ou seja: a automação de e-mails deixou de ser standalone e virou uma peça de um sistema maior, chamada sob demanda.

## Como funciona

### Agendamento de Reuniões (`agendamento-reunioes.json`)

E-mail novo chega → uma IA decide se é pedido de reunião → se tiver horário definido, consulta a agenda e pede aprovação no Telegram; se não tiver, uma IA escreve a pergunta de horário (com aprovação antes de enviar) e espera a resposta na mesma thread. Se a resposta não deixar um dia e horário claros, o fluxo avisa no Telegram em vez de tentar marcar algo errado na agenda. Aprovado, cria o evento no Google Calendar e confirma por e-mail; recusado, pede outro horário pra pessoa e volta a aguardar resposta na mesma thread (renegociação). Um segundo gatilho, rodando a cada 5 minutos, verifica reuniões já vencidas sem confirmação e avisa a outra pessoa se houve imprevisto.

### Automação de E-mails (`automacao-emails.json`)

Chamado como sub-fluxo quando o e-mail não é sobre reunião. Lê o e-mail inteiro — texto, fotos, PDFs e planilhas anexadas, a IA enxerga imagem e PDF de verdade, não só o texto — e escreve uma resposta com tom natural, em parágrafos curtos, sem jargão. A resposta só é enviada (como reply direto na thread original) depois de aprovação no Telegram; recusada, a IA reescreve na hora guardando a tentativa anterior como contexto, sem precisar mexer em label do Gmail.

**Resiliente a falha de modelo:** os dois workflows rodam a IA em cima do OpenRouter (modelo gratuito), com Gemini como fallback automático caso o principal falhe ou retorne vazio — os dois nodes de Agente têm `retryOnFail` configurado também, pra segurar falhas transitórias antes mesmo de cair no fallback.

## Configuração — um lugar só pra editar

Cada workflow tem um node **`Config`** logo no início (Set node), com os campos que mudam de pessoa pra pessoa: `nomeUsuario`, `telegramChatId` e, no de reuniões, `calendarEmail`. Todo o resto do fluxo — prompts de IA, mensagens do Telegram — referencia esse node dinamicamente (`{{ $('Config').item.json.campo }}`), então reaproveitar este workflow é editar um node só, não caçar valor fixo espalhado.

> Os arquivos deste repositório já vêm com placeholders (`Seu Nome`, `SEU_CHAT_ID_TELEGRAM`, `seu-email@gmail.com`) no lugar dos meus dados reais — preencha com os seus depois de importar.

## Como importar

1. Abra seu n8n → **Workflows** → **Import from File** → importe **os dois arquivos**, `agendamento-reunioes.json` e `automacao-emails.json`.
2. Conecte suas próprias credenciais nos nodes de Gmail (OAuth2), Telegram, OpenRouter, Google Gemini e Google Calendar — os arquivos não trazem nenhuma credencial, só o nome que cada node espera.
3. Preencha o node `Config` de cada workflow com seus dados (nome, chat ID do Telegram, e-mail do calendário).
4. **Reconecte a chamada entre os dois workflows:** o n8n gera um ID novo pra cada workflow importado, então o node `Executa Sub-fluxo de Automação de E-mail` (dentro de `agendamento-reunioes.json`) vai perder a referência pro workflow de e-mails. Abra esse node e selecione o workflow `Automação de Emails` recém-importado na lista.
5. **Crie a label do Gmail `Horário Pendente`** e troque o ID dela nos 4 nodes que a usam (`Thread Tem Label Horário Pendente?`, `Aplica Label Horário Pendente`, `Remove Label Horário Pendente`, `Marca Thread Aguardando Novo Horário`) — o arquivo traz o ID da minha própria label, que só existe na minha conta.
6. **Crie a Data Table `reunioes_agendadas`** (usada pra guardar o estado das reuniões) com as colunas: `pessoa`, `email_pessoa`, `datetime_reuniao`, `assunto`, `status`, `thread_id` — os tipos e nomes exatos estão na nota do node `Registra Reunião na Tabela`. Depois, reconecte o node dessa tabela (e os outros 3 que a usam) pra apontar pra ela.
7. Ative os dois workflows.

## Se fosse rodar fora do localhost

Essa instância roda em `localhost:5678` via Docker, num único container, sem domínio público. Funciona bem pra desenvolver e testar, mas é uma configuração de desenvolvimento — não de produção. Comparando com o que instâncias n8n de produção normalmente têm ([fontes no fim](#fontes)):

| Item | Aqui (localhost) | Numa instância pública/produção |
|---|---|---|
| Banco de dados | SQLite (arquivo local) | PostgreSQL — SQLite trava o arquivo inteiro a cada escrita; com webhooks simultâneos, execuções concorrentes começam a falhar |
| `N8N_ENCRYPTION_KEY` | gerada automaticamente pelo n8n na primeira vez, guardada só localmente | definida explicitamente (`openssl rand -hex 32`) e **backupeada separado do banco** — perdendo essa chave, toda credencial salva fica ilegível pra sempre, mesmo com o banco intacto |
| URL pública / `WEBHOOK_URL` | túnel ngrok com domínio reservado (plano free, `*.ngrok-free.dev`), URL fixa entre reinícios — mas depende do processo do ngrok ficar de pé | domínio real fixo configurado em `N8N_HOST` / `WEBHOOK_URL` |
| HTTPS / reverse proxy | túnel cuida do TLS, sem proxy próprio | Nginx ou Caddy na frente, terminando TLS na porta 443; só 80/443 expostos publicamente, container isolado numa rede interna |
| Autenticação | login padrão do n8n (e-mail + senha) | o mesmo login é o mínimo aceitável; instância exposta à internet geralmente soma SSO na frente (Authelia/Authentik) e rate limit no proxy |
| Execução | um container único, sem fila | modo fila (Redis + workers separados) quando o volume de execuções cresce |
| Recursos | o que a máquina local tiver | mínimo documentado de 2 GB de RAM, 4 GB pra não sentir pressão de swap em produção, disco SSD |

Nada disso muda a lógica dos workflows em si — só o jeito como a instância de n8n por trás deles é hospedada.

## Aviso

Estes workflows foram feitos e testados ponta a ponta em ambiente real (Gmail + Telegram + Google Calendar + OpenRouter + Gemini). Compartilhados como referência/portfólio — ajuste os prompts e regras de negócio pro seu caso de uso antes de usar em produção.

## Licença

[MIT](LICENSE) — use, copie e adapte à vontade, mantendo os créditos.
