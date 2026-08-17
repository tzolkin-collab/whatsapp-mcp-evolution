# WhatsApp Evolution API — MCP Server

Servidor **Model Context Protocol (MCP)** que expõe a **Evolution API v2** para assistentes de IA (Claude Desktop, Claude Code, Cursor). Gerencia instâncias de WhatsApp, envia mensagens, configura Typebot e webhooks.

Versão **single-tenant**: uma Evolution API, uma chave global, sem banco de dados, sem billing, sem OAuth. Transporte **SSE** sobre Express.

> Origem: extraído do commit `c4bc066` do repositório `whatsapp-connector`, antes de ele virar produto multi-tenant.

---

## Variáveis de ambiente

| Variável | Obrigatória | Descrição |
| :--- | :--- | :--- |
| `EVOLUTION_API_URL` | ✅ | Endpoint da Evolution API v2, sem barra no final |
| `EVOLUTION_GLOBAL_KEY` | ✅ | Chave global (admin). Dá acesso a **todas** as instâncias |
| `PORT` | ❌ | Porta HTTP. Default `3000` |

Copie `.env.example` para `.env` e preencha. O processo aborta no boot se as duas obrigatórias faltarem.

---

## Rodando local

```bash
npm install
npm run build
npm start
```

Ou em watch, sem build:

```bash
npm run dev
```

Endpoints:

- `GET /health` → `{"status":"ok","transport":"sse","sessions":N}`
- `GET /sse` → abre a conexão SSE (o cliente MCP conecta aqui)
- `POST /messages?sessionId=...` → canal de volta do MCP

---

## Deploy no Easypanel

1. **Criar o serviço** — App → *Create Service* → **App**.
2. **Source** — aponte para este repositório Git (branch `main`).
3. **Build** — método **Dockerfile**, path `./Dockerfile`. O build é multi-stage e a imagem final roda como usuário não-root com apenas as dependências de produção.
4. **Environment** — adicione:
   ```
   EVOLUTION_API_URL=https://evolutionapi.landcriativa.com
   EVOLUTION_GLOBAL_KEY=<sua-chave-global>
   ```
   Não defina `PORT` — o Easypanel injeta a dele e o código respeita.
5. **Domains** — exponha a **porta 3000**, ative HTTPS e anote o domínio gerado.
6. **Deploy.** O healthcheck do container bate em `/health`; se ficar verde, subiu.

### Conferindo

```bash
curl https://SEU-DOMINIO/health
```

### Nota sobre SSE atrás de proxy

O transporte SSE mantém a conexão HTTP aberta. Se o cliente MCP cair sozinho depois de alguns minutos, aumente o timeout de proxy do serviço no Easypanel (Advanced → Proxy) — o default costuma ser curto demais para conexões longas.

---

## Registrando no cliente de IA

### Claude Desktop (`claude_desktop_config.json`)

Local, rodando o build da máquina:

```json
{
  "mcpServers": {
    "whatsapp-evolution": {
      "command": "node",
      "args": [
        "D:/Códigos/Tzolkin/Projetos/Produtos Tzolkin/mcps_personalizados/whatsapp-mcp-evolution/build/index.js"
      ],
      "env": {
        "EVOLUTION_API_URL": "https://evolutionapi.landcriativa.com",
        "EVOLUTION_GLOBAL_KEY": "sua-chave-global"
      }
    }
  }
}
```

Remoto, apontando para o Easypanel:

```json
{
  "mcpServers": {
    "whatsapp-evolution": {
      "url": "https://SEU-DOMINIO/sse"
    }
  }
}
```

### Claude Code

```bash
claude mcp add --transport sse whatsapp-evolution https://SEU-DOMINIO/sse
```

⚠️ Este servidor **não tem autenticação**. Qualquer um que alcance `/sse` controla o WhatsApp com a chave global. Se for expor publicamente, ponha um proxy com auth na frente ou restrinja por IP.

---

## Ferramentas disponíveis

| Categoria | Ferramenta | Descrição |
| :--- | :--- | :--- |
| **Instância** | `list_instances` | Lista instâncias e status de conexão |
| **Instância** | `create_instance` | Cria uma nova instância de WhatsApp |
| **Instância** | `connect_instance` | Retorna QR Code / dados de emparelhamento |
| **Instância** | `get_instance_status` | Status da conexão (CONNECTED, DISCONNECTED…) |
| **Instância** | `logout_instance` | Desconecta a sessão ativa |
| **Instância** | `delete_instance` | Exclui a instância do servidor |
| **Mensagens** | `send_text` | Envia texto para contatos ou grupos |
| **Mensagens** | `send_media` | Envia imagem, vídeo, áudio ou documento (URL ou Base64) |
| **Mensagens** | `get_messages` | Histórico de mensagens da instância (exige banco habilitado na Evolution) |
| **Typebot** | `configure_typebot` | Configura e ativa a integração do Typebot |
| **Typebot** | `get_typebot_settings` | Exibe as configurações atuais do Typebot |
| **Typebot** | `change_typebot_status` | Abre, pausa ou fecha a sessão de atendimento |
| **Typebot** | `start_typebot_flow` | Inicia manualmente um fluxo para um contato |
| **Webhook** | `configure_webhook` | Configura webhooks de notificação/recebimento |
| **Webhook** | `get_webhook_settings` | Consulta os webhooks configurados |
