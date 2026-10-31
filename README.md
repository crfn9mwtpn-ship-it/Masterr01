# Bot WhatsApp + IA (Render)

Servico que mantem a conexao do WhatsApp via **Baileys**, publica o QR code no painel
e faz a ponte com a funcao `wa-webhook` do Backend, que responde com IA.

## Como funciona

```
WhatsApp  -->  este bot (Render)  -->  wa-webhook (Backend)  -->  IA
                    |                        |
                    |                        +-- grava conversa e mensagem
                    +---- le a fila de envio (wa_messages.status = 'queued')
```

1. O bot gera o QR e salva em `wa_bot_status.qr_code` (o painel mostra a imagem).
   O QR tambem aparece direto no endereco do servico no Render, com um diagnostico
   do que esta faltando (`https://SEU-SERVICO.onrender.com/`).
2. Cada mensagem recebida vai para `WEBHOOK_URL` com o header `x-bot-token`.
3. O Backend grava, chama a IA e coloca a resposta na fila (`status = queued`).
4. Este bot le a fila a cada 3 segundos, envia pelo WhatsApp e marca como `sent`.

## Deploy no Render (passo a passo)

1. Descompacte o `.zip`. Suba **os arquivos na raiz** de um repositorio novo no GitHub.
   `package.json`, `Dockerfile` e `render.yaml` precisam ficar na raiz (sem subpasta).
2. No Render: **New +** -> **Blueprint** -> selecione o repositorio.
   O `render.yaml` ja cria o servico, o health check e o disco de 1 GB em `/data`.
3. Abra o servico -> aba **Environment** e cadastre:
   - `SUPABASE_SERVICE_ROLE_KEY` -> a chave de servico do seu Backend (**obrigatoria**)
   - `BOT_TOKEN` -> opcional, so se voce cadastrou esse segredo no Backend
4. **Manual Deploy** -> **Deploy latest commit**.
5. Acesse o painel do projeto -> **Conexao** e escaneie o QR.

As variaveis `SUPABASE_URL` e `WEBHOOK_URL` **ja vem preenchidas** no `render.yaml` e
tambem dentro do `index.js`. Se quiser trocar, basta definir no Render que o valor do
Render tem prioridade.

## O bot subiu mas nao gera QR

Abra o proprio endereco do servico no navegador: `https://SEU-SERVICO.onrender.com/`.
Ele mostra o QR (quando existir) e uma lista de diagnostico com check/cruz:

- SUPABASE_URL configurada
- SUPABASE_SERVICE_ROLE_KEY configurada
- Gravacao no Backend (`wa_bot_status`)
- WEBHOOK_URL configurada
- QR gerado nesta sessao

Se um item estiver com X vermelho, a mensagem logo abaixo explica o motivo exato.
O QR tambem pode ser aberto direto em imagem: `https://SEU-SERVICO.onrender.com/qr.png`.
Se o QR aparece nessa pagina mas nao no painel do site, o problema e permissao/gravacao
no Backend (linha "Gravacao no Backend" vermelha).

Para dados em JSON use `https://SEU-SERVICO.onrender.com/health`. A resposta mostra
`ready: false` e a lista `missing` dizendo exatamente qual variavel falta. Exemplo:

```json
{ "ok": true, "ready": false, "missing": ["SUPABASE_SERVICE_ROLE_KEY"], "status": "error" }
```

O servico **nao derruba mais o deploy** quando falta variavel: ele fica no ar mostrando
o aviso nos logs e no `/health` ate voce cadastrar. Depois de cadastrar, faca um novo
deploy.

## Outros erros comuns

| Sintoma no log | Causa | Solucao |
| --- | --- | --- |
| `Exited with status 1` + faltam variaveis | variaveis nao cadastradas | aba Environment |
| QR aparece e some toda hora | sem disco persistente | crie Disk de 1 GB em `/data` |
| `Sessao do WhatsApp encerrada (logout)` | celular removeu o aparelho | o bot apaga a sessao e gera QR novo sozinho |
| Bot conectado mas IA nao responde | falta `GEMINI_API_KEY` no Backend | cadastre o segredo no Backend |
| Mensagens nao saem | conversa sem telefone ou numero invalido | veja `wa_messages.error` e os logs |

## Teste rapido

1. `https://SEU-SERVICO.onrender.com/health` -> `{ "ok": true, "ready": true }`
2. Painel -> **Conexao** -> escaneie o QR.
3. Mande "oi" de outro numero e acompanhe em **Conversas**.
4. Se nada acontecer, olhe os logs do Render e a tabela `wa_logs`.

## Avisos importantes

- **Baileys nao e oficial.** O QR so existe em bibliotecas nao oficiais. O uso pode
  resultar em bloqueio do numero. Em escala, o caminho e a Cloud API oficial.
- Rode **uma unica instancia** por numero. Duas instancias derrubam a sessao.
- Nunca comite a `SUPABASE_SERVICE_ROLE_KEY`.
- O plano gratuito do Render hiberna e nao tem disco: para uso real, use plano com disco.