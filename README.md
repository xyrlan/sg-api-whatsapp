# sg-api-whatapp — Evolution API Infrastructure

Infraestrutura Docker para a [Evolution API](https://github.com/EvolutionAPI/evolution-api) (gateway WhatsApp Web via Baileys). Consumido por uma aplicacao Next.js na Vercel via HTTP REST.

## Arquitetura

```
Vercel (Next.js) --HTTP POST--> Railway (Evolution API :8080) --WhatsApp Web--> Cliente
                                        |
                                   PostgreSQL (sessoes, mensagens)
                                   Redis (cache de conexao)
```

## Pre-requisitos

- [Docker](https://docs.docker.com/get-docker/) e Docker Compose v2

## Desenvolvimento Local

```bash
# 1. Copiar template de variaveis
cp .env.example .env

# 2. Editar .env — trocar AUTHENTICATION_API_KEY por uma chave segura
nano .env

# 3. Subir os servicos
docker compose up -d

# 4. Verificar se a API esta rodando
curl http://localhost:8080/

# 5. Parar os servicos
docker compose down
```

Para apagar os volumes (dados do Postgres/Redis):

```bash
docker compose down -v
```

## Deploy no Railway

### 1. Criar o projeto

- Acesse [railway.app](https://railway.app) e crie um novo projeto

### 2. Adicionar banco de dados

- Clique em **"+ New"** > **"Database"** > **PostgreSQL**
- Repita para **Redis**

### 3. Adicionar a Evolution API

- Clique em **"+ New"** > **"Docker Image"**
- Imagem: `atendai/evolution-api:v2.2.3`

### 4. Configurar variaveis de ambiente

No servico da Evolution API, adicione:

| Variavel | Valor |
|----------|-------|
| `SERVER_URL` | Dominio publico do Railway (ex: `https://seu-servico.up.railway.app`) |
| `SERVER_PORT` | `8080` |
| `AUTHENTICATION_API_KEY` | Uma string aleatoria forte |
| `AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES` | `true` |
| `DATABASE_ENABLED` | `true` |
| `DATABASE_PROVIDER` | `postgresql` |
| `DATABASE_CONNECTION_URI` | `${{Postgres.DATABASE_URL}}` |
| `DATABASE_CONNECTION_CLIENT_NAME` | `evolution_api` |
| `DATABASE_SAVE_DATA_INSTANCE` | `true` |
| `DATABASE_SAVE_DATA_NEW_MESSAGE` | `true` |
| `DATABASE_SAVE_MESSAGE_UPDATE` | `true` |
| `DATABASE_SAVE_DATA_CONTACTS` | `true` |
| `DATABASE_SAVE_DATA_CHATS` | `true` |
| `DATABASE_SAVE_DATA_LABELS` | `true` |
| `DATABASE_SAVE_DATA_HISTORIC` | `true` |
| `CACHE_REDIS_ENABLED` | `true` |
| `CACHE_REDIS_URI` | `${{Redis.REDIS_URL}}` |
| `CACHE_REDIS_PREFIX_KEY` | `evolution` |
| `CACHE_REDIS_SAVE_INSTANCES` | `false` |
| `CACHE_LOCAL_ENABLED` | `false` |

> **Nota:** `${{Postgres.DATABASE_URL}}` e `${{Redis.REDIS_URL}}` sao variaveis de referencia do Railway que apontam automaticamente para os add-ons.

### 5. Gerar dominio publico

- No servico da Evolution API, va em **Settings** > **Networking** > **Generate Domain**
- Copie a URL gerada e atualize `SERVER_URL` com ela

## Conectando do Next.js (Vercel)

Adicione estas variaveis de ambiente no seu projeto Vercel:

| Variavel | Valor |
|----------|-------|
| `EVOLUTION_API_URL` | `https://seu-servico.up.railway.app` |
| `EVOLUTION_API_KEY` | Mesma chave do `AUTHENTICATION_API_KEY` no Railway |
| `EVOLUTION_INSTANCE_NAME` | Nome da instancia criada na Evolution API |

Exemplo de chamada:

```typescript
const response = await fetch(
  `${process.env.EVOLUTION_API_URL}/message/sendText/${process.env.EVOLUTION_INSTANCE_NAME}`,
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      apikey: process.env.EVOLUTION_API_KEY!,
    },
    body: JSON.stringify({
      number: "5511999999999",
      text: "Mensagem de teste",
    }),
  }
);
```

## Endpoints Uteis

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| `GET` | `/` | Status da API |
| `POST` | `/instance/create` | Criar instancia WhatsApp |
| `GET` | `/instance/connect/{instanceName}` | Obter QR Code para conectar |
| `GET` | `/instance/fetchInstances` | Listar todas as instancias |
| `POST` | `/message/sendText/{instanceName}` | Enviar mensagem de texto |
| `POST` | `/message/sendMedia/{instanceName}` | Enviar midia (imagem, PDF, etc.) |
| `DELETE` | `/instance/delete/{instanceName}` | Remover instancia |

Todos os endpoints requerem o header `apikey: SUA_CHAVE`.

## Referencia de Variaveis de Ambiente

| Variavel | Descricao | Default | Obrigatoria |
|----------|-----------|---------|-------------|
| `SERVER_URL` | URL publica da API | `http://localhost:8080` | Sim |
| `SERVER_PORT` | Porta do servidor | `8080` | Nao |
| `AUTHENTICATION_API_KEY` | Chave de autenticacao da API | - | Sim |
| `DATABASE_ENABLED` | Habilitar persistencia no PostgreSQL | `true` | Sim |
| `DATABASE_PROVIDER` | Tipo do banco | `postgresql` | Sim |
| `DATABASE_CONNECTION_URI` | String de conexao do PostgreSQL | - | Sim |
| `CACHE_REDIS_ENABLED` | Habilitar cache Redis | `true` | Sim |
| `CACHE_REDIS_URI` | String de conexao do Redis | - | Sim |
| `WEBHOOK_GLOBAL_ENABLED` | Habilitar webhook global | `false` | Nao |
| `WEBHOOK_GLOBAL_URL` | URL do webhook no Next.js | - | Se webhooks habilitados |

## Troubleshooting

**Porta 8080 em uso:**
```bash
sudo lsof -i :8080
# Pare o processo ou altere SERVER_PORT no .env
```

**PostgreSQL nao conecta:**
- Verifique se o container `evolution_postgres` esta rodando: `docker compose ps`
- Confira a `DATABASE_CONNECTION_URI` — o host deve ser `postgres` (nome do servico)

**Redis recusando conexao:**
- Verifique se o container `evolution_redis` esta rodando: `docker compose ps`
- Confira a `CACHE_REDIS_URI` — o host deve ser `redis` (nome do servico)

**QR Code perdido apos restart:**
- Isso acontece quando `DATABASE_ENABLED=false`. Certifique-se de que o PostgreSQL esta configurado e ativo.
