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
Independentemente do evento, a Meta sempre enviará a requisição em um formato padrão.

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
## Mensagem de mídia (imagem, audio, video, documento)
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
>Nota: o payload se adapta dinamicamente de acordo com o campo type recebido. O comportamento de download utilizando o id da mídia é exatamente o mesmo para todos os formatos.

#### Abaixo estão os tipos de mídia que podemos receber:
- audio: Arquivo de áudio gravado ou enviado como MP3/Ogg. O objeto principal será "audio": { "id": "...", "mime_type": "..." }.
- document: Qualquer documento generalizado, como PDFs ou planilhas. O objeto principal será "document": { "id": "...", "filename": "...", "mime_type": "..." }.
- image: Arquivos de imagem (JPEG/PNG). O objeto principal será "image": { "id": "...", "mime_type": "...", "sha256": "..." }.
- video: Vídeos em formato MP4. O objeto principal será "video": { "id": "...", "mime_type": "...", "sha256": "..." }.
- sticker: Figurinhas do WhatsApp. O objeto principal será "sticker": { "id": "...", "mime_type": "...", "animated": true }.

## Mensagens interativas (botões e listas)
**Quando acontece:** O usuário interage com uma mensagem estruturada que nós enviamos, clicando em um botão de resposta rápida ou selecionando uma opção em um menu de lista.

**Regra de Negócio:**
* *(Ex: O sistema deve mapear o `interactive.button_reply.id` ou `interactive.list_reply.id` para avançar a etapa do cliente no fluxo do chatbot de forma exata.)*

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
## Localização e contatos
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

## Reações
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

## Eventos da Conta (WABA)

Aqui entram as notificações globais da conta do WhatsApp Business.

>Nota: O webhook no painel da Meta precisa estar inscrito no evento `message_template_status_update`. Se a caixa do `messages` for a única marcada, esses eventos não vão chegar no servidor.

## Atualização de Status de Templates
**Quando acontece:** A Meta avisa se um template criado pelo nosso time foi aprovado, reprovado ou desativado.

**O que o sistema tem que fazer:**
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
## Segurança e Validação de Origem

Como a URL do webhook precisa ficar exposta na internet para a Meta acessar, precisamos garantir que quem está mandando o `POST` é realmente o WhatsApp e não alguém de fora.

**Como funciona a validação:**
A Meta não envia tokens de autenticação no corpo da mensagem. Em vez disso, ela assina cada payload usando o nosso **App Secret** e manda o resultado no header da requisição.

**O que o sistema deve fazer:**
1. Interceptar a requisição e buscar o header *`X-Hub-Signature-256`* (ele vem no formato *`sha256=HASH_AQUI`*).
2. Pegar o corpo cru da requisição (*raw body* text, antes do parse para JSON).
3. Gerar um hash *`HMAC-SHA256`* desse *raw body* usando o nosso **App Secret** como chave.
4. Comparar o hash gerado com o que veio no header da Meta. 
5. Se os hashes não baterem, o servidor deve rejeitar imediatamente.

## Verificação do Webhook (Hub Challenge)

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

## Idempotência e Resposta Rápida (Lidando com Duplicatas)

A arquitetura de webhooks da Meta garante que a notificação será enviada *pelo menos uma vez*. Isso significa que, por oscilações de rede ou atrasos, **é comum recebermos o mesmo payload repetido**. 

Além disso, a Meta exige que o nosso servidor responda com um *`HTTP 200 OK`* quase que instantaneamente. Se demorarmos para responder (porque estamos salvando um arquivo ou processando uma lógica pesada), a Meta entende que deu "Time Out" e tenta mandar a mesma requisição de novo.

**O que o sistema deve fazer:**
1. **Responder primeiro, processar depois:** Assim que o `POST` bater no webhook e passar na verificação de segurança, devolva um *`HTTP 200 OK`* imediatamente. Jogue o payload para uma fila assíncrona para lidar com ele depois.
2. **Checar a chave única (Idempotência):** Antes de processar a mensagem ou o status, o sistema deve buscar no banco de dados se aquele `id` já foi processado.
   * Para mensagens e status, a chave única é sempre o `id` (que começa com `wamid.`).

---

## Account Update
O webhook de account_update é enviado sempre que há uma alteração no status ou nas informações da sua WhatsApp Business Account (WBA). Ele é crucial para monitorar se a sua conta está ativa, se foi banida ou se passou em revisões da Meta.

#### Quando este evento acontece?
- Mudanças no status de revisão da conta (aprovada/rejeitada).
- Desativação ou restrição da conta por violação de políticas.
- Atualização de detalhes do número de telefone associado.

### Gatilhos:
O webhook account_update é acionado nos seguintes cenários, divididos por categorias:

#### 1. Verificação e Status da Conta
- **Análise de empresa:** quando o envio de verificação conduzido por um parceiro é aprovado, rejeitado ou descartado.
- **Ciclo de vida:** quando uma conta do WhatsApp Business é excluída.
-  **Conformidade:** quando a conta viola políticas ou termos da Meta, ou sofre restrições devido a ações de aplicação (account restrictions).
-  **Termos:** quando a empresa aceita os Termos de Serviço da API de mensagens multi-dispositivo (MM).

#### 2. Acessos e Permissões
- **Parceiros:** quando a conta é compartilhada ou deixa de ser compartilhada com um parceiro.
- **Apps:** quando um cliente empresarial concede ou revoga permissões de aplicativos para a conta.
- **Anúncios:** quando a conta dá acesso ao parceiro para gerenciar contas de anúncios.

#### 3. Configurações Comerciais e Preços
- **Localização:** quando o ponto comercial principal da conta é definido.
- **Financeiro:** quando a conta se qualifica para taxas internacionais de autenticação ou quando o nível de preços baseado em volume é atualizado.

#### 4. Registro e Dispositivos
- **Migração:** quando uma conta é removida devido a uma mudança de dispositivo ou novo registro de número de telefone.
- **Reconexão:** quando a conta é reconectada após essa mudança de dispositivo ou registro.

### Referência de campos
Abaixo estão todos os campos possíveis que podem ser retornados:

| Campo | Tipo | Descrição
| :--- | :--- | :---|
| `event` | string | **Obrigatório**. Indica a ação que disparou o webhook (Ex: `DISABLED`, `REVIEW_PASSED`). |
| `phone_number_details` | object | Contém informações se o evento afetar um número de telefone específico. |
| `ban_info` | object | Detalhes sobre banimentos aplicados à conta. |
| `restriction_info` | object | Informações sobre restrições de integridade ou de envio aplicadas. |
| `violation_info` | object | Detalhes sobre violações de políticas específicas. |

### Detalhes de objetos aninhados
Abaixo estão as integrações de dados para mapear os sub-objetos:

1. **Objeto phone_number_details**
   Usado quando o gatilho envolve a verificação ou atualização de um número.
   - `display_phone_number` (string): o número de telefone formatado.
   - `event` (string): o novo status do número. Valores comuns: `PENDING`, `VERIFIED`, `UPGRADED`.
2. **Objeto ban_info**
   Essencial para monitorar bloqueios de segurança.
   - `event` (string): o status do banimento. Valores: `BAN_ADDED`, `BAN_REMOVED`.
   - `expiration` (string/timestamp): data e hora em que o banimento expira (se for temporário).
3. **Objeto restriction_info**
   Indica limites impostos à conta por baixa qualidade ou spam.
   - `event` (string): status da restrição. Valores: `RESTRICTION_ADDED`, `RESTRICTION_REMOVED`.
   - `type` (string): o tipo de restrição aplicada (Ex: Limite de mensagens por dia).
   - `expiration` (string/timestamp): data prevista para o fim da restrição.

### Lista completa de eventos (event)
Estes são os valores exatos que a string event pode assumir. A aplicação deve estar preparada para:

- `ACCOUNT_RESTRICTED`: A conta sofreu limitações de uso.
- `ACCOUNT_UNRESTRICTED`: As restrições da conta foram removidas.
- `DISABLED`: A conta foi desativada permanentemente ou até ação do suporte.
- `REVIEW_PASSED`: A conta ou número passou na revisão comercial.
- `REVIEW_FAILED`: A conta ou número foi reprovado na revisão.
- `MESSAGE_LIMIT_CHANGED`: O limite de mensagens (Tier) do número foi alterado.
- `ONBOARDING_COMPLETED`: A configuração inicial da conta foi finalizada com sucesso.
- `VERIFIED_ACCOUNT`: A conta recebeu o selo de verificação oficial.

⚠️ Notas de Implementação
1. **Nulidade de Campos:** Nem todos os objetos (`ban_info`, `restriction_info`, etc.) estarão presentes simultaneamente. Sua lógica deve sempre verificar a existência da chave antes de tentar acessar o valor (ex: `if (value.ban_info) { ... }`).
2. **Tratamento de Timestamps:** Os campos de `expiration` seguem o padrão *ISO 8601* ou *Unix Timestamp*. Certifique-se de converter para o fuso horário local da sua operação na documentação.

### Exemplos de payload (account_update)
Abaixo estão os cenários mais críticos que o sistema pode receber.

1. **Conta aprovada:** evento disparado quando a conta passa pela revisão comercial inicial.
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "SEU_WABA_ID",
      "changes": [
        {
          "value": {
            "event": "REVIEW_PASSED"
          },
          "field": "account_update"
        }
      ]
    }
  ]
}
```
2. **Alteração de Limite de Mensagens (Com Detalhes do Número):** evento disparado quando o número ganha ou perde capacidade de envios diários. O campo `phone_number_details` é adicionado.
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "SEU_WABA_ID",
      "changes": [
        {
          "value": {
            "event": "MESSAGE_LIMIT_CHANGED",
            "phone_number_details": {
              "display_phone_number": "5511999999999",
              "event": "UPGRADED"
            }
          },
          "field": "account_update"
        }
      ]
    }
  ]
}
```
3. **Conta Desativada / Banida (Com Informações de Banimento):** evento disparado quando a Meta bloqueia a conta. O sistema deve alertar imediatamente usando o objeto ban_info.
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "SEU_WABA_ID",
      "changes": [
        {
          "value": {
            "event": "DISABLED",
            "ban_info": {
              "event": "BAN_ADDED",
              "expiration": "2026-05-01T12:00:00+0000"
            }
          },
          "field": "account_update"
        }
      ]
    }
  ]
}
```
