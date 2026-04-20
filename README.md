# Documentação de Webhooks - WhatsApp

## 📌 Visão Geral
Este documento mapeia os eventos recebidos via Webhook da Meta para o WhatsApp Business API. Aqui é detalhado os payloads esperados e as regras de negócio associadas a cada evento.

---

### 📋 Guia de Referência Rápida
| Field | O que significa? | 
| :--- | :--- |
| `messages` | Mensagens do usuário (texto, mídia, local) |
| `statuses` | Confirmação de entrega/leitura |
| `message_template_status_update` | Alterações em templates |

---

## Estrutura Padrão
Independentemente do evento, a Meta sempre enviará a requisição em um formato padrão. A diferença entre uma mensagem recebida.

``` json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "SEU_WABA_ID",
      "changes": [
        {
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "5511999999999",
              "phone_number_id": "SEU_PHONE_NUMBER_ID"
            },
            // AQUI DENTRO VEM O OBJETO "messages" ou "statuses"
          },
          "field": "messages"
        }
      ]
    }
  ]
}
```
### Mensagem de mídia (imagem, audio, video, documento)
**Quando acontece:** O usuário envia um arquivo.

**Exemplo de Payload recebido (Exemplo de Imagem):**
```json
{
  "messages": [
    {
      "from": "5511988887777",
      "id": "wamid.HBgLNTUxMTk5...",
      "timestamp": "1713589120",
      "type": "image",
      "image": {
        "mime_type": "image/jpeg",
        "sha256": "hash_do_arquivo_aqui",
        "id": "ID_DA_MIDIA_PARA_DOWNLOAD"
      }
    }
  ]
}
```
>Nota: o acordo muda de acordo com o tipo de media em *type*. Se for áudio, virá "type": "audio" e um objeto "audio": { "id": "..." }. Se for documento, "type": "document" e assim por diante. O comportamento de download via ID é o mesmo para todos.

### Mensagens interativas (botões e listas)
**Quando acontece:** O usuário interage com uma mensagem estruturada que nós enviamos, clicando em um botão de resposta rápida ou selecionando uma opção em um menu de lista.

**Regra de Negócio:**
* *(Ex: O sistema deve mapear o `interactive.button_reply.id` ou `interactive.list_reply.id` para avançar a etapa do cliente no fluxo do chatbot de forma exata.*

**Exemplo de Payload recebido (Exemplo de clique em Botão):**
```json
{
  "messages": [
    {
      "from": "5511988887777",
      "id": "wamid.HBgLNTUxMTk5...",
      "timestamp": "1713589250",
      "type": "interactive",
      "interactive": {
        "type": "button_reply",
        "button_reply": {
          "id": "btn_falar_atendente",
          "title": "Falar com Atendente"
        }
      }
    }
  ]
}
```
### Localização e contatos
**Quando acontece:** O usuário compartilha uma localização (mapa) ou um cartão de contato (vCard) pelo anexo do WhatsApp.

**Regra de Negócio:**
* *(Ex: Se for localização, extrair `latitude` e `longitude` para verificar a viabilidade de entrega ou sugerir a loja física mais próxima. Se for contato, processar os dados e salvar no banco de dados como uma possível indicação).*

**Exemplo de Payload (Exemplo de Localização):**
```json
{
  "messages": [
    {
      "from": "5511988887777",
      "id": "wamid.HBgLNTUxMTk5...",
      "timestamp": "1713589300",
      "type": "location",
      "location": {
        "latitude": -25.4284,
        "longitude": -49.2733,
        "name": "Nome do Local (Opcional)",
        "address": "Endereço completo (Opcional)"
      }
    }
  ]
}
```
>Nota: Se o usuário enviar um contato, o type será "contacts", e o JSON trará um array com os dados (nome, telefones, etc.).

### Reações
**Quando acontece:** O usuário reage a uma mensagem nossa com um emoji.

**Regra de Negócio:**
* *(Ex: O sistema deve atualizar a reação na interface do atendente ou, se for um bot, pode ignorar reações para não gerar processamento desnecessário).*

**Exemplo de Payload recebido:**
```json
{
  "messages": [
    {
      "from": "5511988887777",
      "id": "wamid.HBgLNTUxMTk5...",
      "timestamp": "1713589500",
      "type": "reaction",
      "reaction": {
        "message_id": "wamid.ID_DA_MENSAGEM_QUE_RECEBEU_A_REACAO",
        "emoji": "👍"
      }
    }
  ]
}
```
---

### Eventos da Conta (WABA)

Aqui entram as notificações globais da conta do WhatsApp Business.

>Nota: O webhook no painel da Meta precisa estar inscrito no evento `message_template_status_update`. Se a caixa do `messages` for a única marcada, esses eventos não vão chegar no servidor.

### Atualização de Status de Templates
**Quando acontece:** A Meta avisa se um template criado pelo nosso time foi aprovado, reprovado ou desativado.

**O que sistema tem que fazer:**
* Pegar o `message_template_id` e atualizar o status no banco de dados.
* Se vier `APPROVED`, liberar o template
* Se vier `REJECTED` ou `FLAGGED`, bloquear o uso e avisar.

**Payload de Exemplo:**
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "SEU_WABA_ID",
      "changes": [
        {
          "value": {
            "event": "APPROVED", 
            "message_template_id": "9876543210123",
            "message_template_name": "boas_vindas_v1",
            "message_template_language": "pt_BR",
            "reason": "NONE"
          },
          "field": "message_template_status_update"
        }
      ]
    }
  ]
}
```
### Segurança e Validação de Origem

Como a URL do webhook precisa ficar exposta na internet para a Meta acessar, precisamos garantir que quem está mandando o `POST` é realmente o WhatsApp e não alguém de fora.

**Como funciona a validação:**
A Meta não envia tokens de autenticação no corpo da mensagem. Em vez disso, ela assina cada payload usando o nosso **App Secret** e manda o resultado no header da requisição.

**O que o sistema deve fazer:**
1. Interceptar a requisição e buscar o header *`X-Hub-Signature-256`* (ele vem no formato *`sha256=HASH_AQUI`*).
2. Pegar o corpo cru da requisição (*raw body* text, antes do parse para JSON).
3. Gerar um hash *`HMAC-SHA256`* desse *raw body* usando o nosso **App Secret** como chave.
4. Comparar o hash gerado com o que veio no header da Meta. 
5. Se os hashes não baterem, o servidor deve rejeitar imediatamente.

### Verificação do Webhook (Hub Challenge)

Antes da Meta começar a enviar as mensagens (via `POST`), ela precisa ter certeza de que a URL que colocamos no painel é válida. Isso é feito através de uma requisição de verificação.

**Como funciona a validação:**
No momento em que clicamos em "Salvar" no painel da Meta, o WhatsApp faz uma única requisição `GET` para a nossa URL do webhook.

Nesse `GET`, a Meta envia os seguintes parâmetros na query string (URL):
* *`hub.mode`*: Sempre virá com o valor `"subscribe"`.
* *`hub.verify_token`*: Uma senha em texto que nós mesmos inventamos e digitamos lá no painel da Meta na hora de configurar.
* *`hub.challenge`*: Um código numérico aleatório gerado pela Meta.

**O que o sistema deve fazer:**
1. A nossa rota de webhook precisa aceitar requisições `GET` (além do `POST` que já usamos para as mensagens).
2. O servidor deve verificar se o *`hub.verify_token`* recebido é igual ao token que configuramos.
3. Se for igual, o servidor deve responder com um *`HTTP 200 OK`* retornando **apenas** o valor exato do *`hub.challenge`* em texto puro.
4. Se o token for diferente, retornar um *`HTTP 403 Forbidden`*.

### Idempotência e Resposta Rápida (Lidando com Duplicatas)

A arquitetura de webhooks da Meta garante que a notificação será enviada *pelo menos uma vez*. Isso significa que, por oscilações de rede ou atrasos, **é comum recebermos o mesmo payload repetido**. 

Além disso, a Meta exige que o nosso servidor responda com um *`HTTP 200 OK`* quase que instantaneamente. Se demorarmos para responder (porque estamos salvando um arquivo ou processando uma lógica pesada), a Meta entende que deu "Time Out" e tenta mandar a mesma requisição de novo.

**O que o sistema deve fazer:**
1. **Responder primeiro, processar depois:** Assim que o `POST` bater no webhook e passar na verificação de segurança, devolva um *`HTTP 200 OK`* imediatamente. Jogue o payload para uma fila assíncrona para lidar com ele depois.
2. **Checar a chave única (Idempotência):** Antes de processar a mensagem ou o status, o sistema deve buscar no banco de dados se aquele `id` já foi processado.
   * Para mensagens e status, a chave única é sempre o `id` (que começa com `wamid.`).


