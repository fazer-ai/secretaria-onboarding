# 03: Deploy do Chatwoot (Pro ou OSS)

## Primeiro: leia o marcador e ramifique (Pro vs OSS)

Leia `~/.fazer-ai/onboarding.json` → `chatwootTier`. Eixo **independente** da edição da Secretária V4 (`secretariaEdition`, etapa 4). Marcador ausente → fallback pelo hub (`bunx @fazer-ai/secretaria hub licenses`): licença CHATWOOT disponível → Pro; senão OSS.

- **`community` (OSS)** → imagem **pública** `ghcr.io/fazer-ai/chatwoot:latest` (nosso fork), `COMPOSE_PROFILES` vazio (sem `baileys-api`). **NÃO** rode `docker login` nem provisione credencial do Harbor (não há licença e o pull é público). Deploy pelo compose genérico (`templates/chatwoot/`, ver `templates/chatwoot/README.md`); no Coolify, setar `CHATWOOT_IMAGE=ghcr.io/fazer-ai/chatwoot:latest` no `templates/chatwoot/docker-compose.coolify.yml` e **remover** o `baileys-api`. **Pule a etapa 9b** (licenciar). O resto deste doc é **só Pro**.
- **`pro`** → siga abaixo (Harbor + Coolify API + `docker login` + etapa 9b).

## Imagem privada (Harbor): credencial per-user via proxy do CLI

`harbor.fazer.ai/chatwoot/fazer-ai/chatwoot-pro:latest`.
- Credencial do Harbor pelo **proxy do hub no CLI** (não há hub MCP na sessão do agente; o CLI tem o OAuth do bootstrap):
  ```sh
  bunx @fazer-ai/secretaria hub registry-credential --apply --out harbor.secret
  ```
  Robot **per-user** (a MESMA cred cobre Chatwoot Pro e Secretária Pro), idempotente; grava o secret em `harbor.secret` (`0600`) e imprime só o `username`; o secret **nunca** sai no output. **Nunca** logar o secret.
- O compose é o vendorado `templates/chatwoot/docker-compose.coolify.yml` (não precisa extrair do hub).

## Deploy via API do Coolify

O `scripts/coolify.py create-service` lê o compose, faz o **base64** (raw → 422 "should be base64 encoded") e POSTa em `/api/v1/services` com `instant_deploy:false`; depois você deploya:
```sh
python3 scripts/coolify.py create-service --base-url http://<VPS_IP>:8000 --token-file coolify.token \
  --name chatwoot --project-uuid <PROJ_UUID> --server-uuid <SRV_UUID> --environment-name production \
  --compose-file templates/chatwoot/docker-compose.coolify.yml   # → {uuid}
python3 scripts/coolify.py api-post --base-url http://<VPS_IP>:8000 --token-file coolify.token --path /services/<uuid>/start
```
- Logue no Harbor com `scripts/harbor-login.py login` **antes** do `start` (o pull da privada precisa do login): roda `docker login --password-stdin` por SSH (secret fora do argv) e protege o `$` do usuário robot. O `username` vem do `hub registry-credential` (acima); o secret está em `harbor.secret` (`0600`):
```sh
python3 scripts/harbor-login.py login --ssh root@<VPS_IP> --username '<robot-user>' --secret-file harbor.secret
```

## Admin + token (Rails runner via SSH)

O **usuário** cria o 1º admin do Chatwoot na própria tela de onboarding do Chatwoot (`https://chatwoot.<seu-dominio>`): você entrega o link e **espera** ele criar a conta + o admin. Com o admin já criado, `scripts/chatwoot-admin.py provision` **lê** esse admin (pelo email) e devolve o `api_access_token` dele: roda o Rails runner **dentro** do container (base64-piped por SSH, então o email e as aspas do script não tocam o shell) e **nunca** cria conta nem usuário. O token é o `AccessToken` polimórfico do usuário (idempotente: reusa o existente ou minta um pelo `AccessToken` do owner, `find_or_create_by!`).
```sh
python3 scripts/chatwoot-admin.py provision --ssh root@<VPS_IP> --container <chatwoot-rails-container> \
  --email <email-do-admin> --out chatwoot-admin.json
```
Grava `api_access_token` num arquivo `0600`; só metadados são impressos. Se o email ainda não existe, o helper erra claro (`the user must create the admin … first`) → espere o usuário criar e re-rode. Esse `api_access_token` vai no header `api-access-token: <token>` (hífen: sobrevive a proxies, ver `deploy-b-portainer.md`) das chamadas REST do Chatwoot **e** no `deployment_connect` da etapa 9 (transitório, nunca persistido em repo/log).

## FQDN + 503

Ver `gotchas.md`: setar `service_applications.fqdn` no `coolify-db` + restart (o `SERVICE_FQDN_*` env **não** dirige o Traefik).

## Inbox API (pro E2E)

`POST https://chatwoot.<seu-dominio>/api/v1/accounts/1/inboxes` (header `api-access-token`) body:
```json
{"name":"Validação (API)","channel":{"type":"api","webhook_url":""}}
```
→ inbox `Channel::Api`. O bind do agente (etapa 9) provisiona o webhook do bot; **não** precisa setar `webhook_url` à mão.
