# 05: Deploy do Langfuse (com MinIO obrigatório)

## NÃO use o one-click do Coolify

O template one-click declara os `LANGFUSE_S3_*` mas sobe **sem MinIO e com creds vazias**. Resultado: `POST /api/public/ingestion` dá **HTTP 500** (`Could not load credentials from any providers` → `Failed to upload events to blob storage`) e os traces **somem em silêncio**. Pior: `GET /api/public/projects` lê só o Postgres e retorna 200, então um "test connection" ingênuo **passa** e mascara a ingestion quebrada.

## Use o compose vendorado do repo

`templates/langfuse/docker-compose.coolify.yml`: topologia `langfuse` (web) + `langfuse-worker` + `postgres` + `redis` + `clickhouse` + **`minio`**, com as 3 famílias S3 (`EVENT_UPLOAD`/`MEDIA_UPLOAD`/`BATCH_EXPORT`) apontando pra `http://minio:9000` via as magic vars `SERVICE_USER_MINIO`/`SERVICE_PASSWORD_MINIO`. Deploy via `scripts/coolify.py create-service` (base64) + `set-fqdn` (abaixo). Detalhes e mapa magic-var↔env genérico: `templates/langfuse/README.md`.

## Fluxo user-first-seed (você provisiona as keys; o usuário nunca copia nada)

Validado empiricamente no `langfuse:3`. O Langfuse sobe **sem** org/projeto/usuário. A sequência:

1. **Deploy com signup aberto.** Suba com `AUTH_DISABLE_SIGNUP=false` (Coolify: setar na env do serviço; genérico: no `.env`). Sem isso o usuário não consegue se registrar.
2. **O usuário cria conta + organização no browser** (`https://langfuse.<seu-dominio>:3000`). Entregue o link e **espere**; nunca crie a conta por conta própria. Atenção ao que o teste empírico mostrou: o signup cria **só o usuário**; a **organização é um 2º passo** do onboarding do Langfuse. Espere a **org** existir, não só a conta.
3. **Descubra o `org_id` do usuário** pelo Postgres do Langfuse (você tem SSH + docker na VPS; o container é UUID no Coolify, ache pela imagem `langfuse`+`postgres` como no inventário brownfield):
   ```sh
   docker exec <langfuse-postgres> psql -U <user> -d <db> -tAc "SELECT id, name FROM organizations"
   ```
4. **Semeie o projeto + as keys.** Gere um par `pk-lf-…`/`sk-lf-…` e set na env do serviço: `LANGFUSE_INIT_ORG_ID=<org do usuário>`, `LANGFUSE_INIT_PROJECT_ID`, `LANGFUSE_INIT_PROJECT_NAME`, `LANGFUSE_INIT_PROJECT_PUBLIC_KEY`, `LANGFUSE_INIT_PROJECT_SECRET_KEY` (**sem** `LANGFUSE_INIT_USER_*`, a conta é do usuário), e **redeploy**. O Langfuse faz upsert **por id**: a org e a membership do usuário são **preservadas**, e o projeto + keys nascem **dentro** da org dele.
5. **Feche o signup.** Set `AUTH_DISABLE_SIGNUP=true` e redeploy. O seed é idempotente (re-rodar não duplica), e o signup passa a devolver `422 Sign up is disabled`.

Como **você gerou** as keys no passo 4, elas já estão na sua mão pra ligar na v4: o usuário nunca abre "Settings → API Keys" nem copia segredo nenhum.

## FQDN (preserve a porta)

`scripts/coolify.py set-fqdn --ssh root@<VPS_IP> --app-id <id> --fqdn https://langfuse.<seu-dominio>:3000` (ache o id com `list-apps`). O template mapeia o FQDN pra porta 3000 do container; **dropar o `:3000` quebra o routing**. Ver `gotchas.md`.

## Verifique a ingestion (health verde NÃO basta)

`scripts/langfuse-verify.py` POSTa um batch em `/api/public/ingestion` e exige **207/200** (não 500); as chaves são o par que você semeou, lidas de um arquivo `0600` (a secret key fora do argv):
```sh
echo '{"publicKey":"<pk-lf>","secretKey":"<sk-lf>"}' > langfuse.keys && chmod 600 langfuse.keys
python3 scripts/langfuse-verify.py ingestion --base-url https://langfuse.<seu-dominio>:3000 --keys-file langfuse.keys
```
Status 500 = quase sempre MinIO/S3 ausente.

## Ligue na v4 (por MCP, `langfuse_connect`)

O wiring é **por MCP**, num tool só: `langfuse_connect` recebe `public_key`/`secret_key`/`base_url` **inline** (as keys que você semeou), cria a credencial no vault **já preenchida** (`kind:"langfuse"`, `{publicKey, secretKey}` + `baseUrl`) e liga o tracing no tenant-settings. É dry-run por padrão: revise o preview (keys redigidas) e reenvie com `dry_run:false` pra aplicar. Mesmo padrão do `deployment_connect` do Chatwoot (segredo de infra inline). Como as keys já existem, a credencial nasce **preenchida** (NÃO `pending`): uma entry pending não resolve o segredo e o tenant-settings rejeita com `credential ref not found`. (No vault o campo é `baseUrl` camelCase, ver `gotchas.md`; doc do tool em `docs/mcp.md`.)
