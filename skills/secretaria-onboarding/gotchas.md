# Gotchas: armadilhas conhecidas

Cada uma é uma armadilha conhecida. Leia a da etapa antes de executá-la.

## Infra / Coolify

### FQDN não dirige o Traefik (503 em todo serviço)

Pra um **service** do Coolify, o Traefik lê `service_applications.fqdn` no DB; o env `SERVICE_FQDN_*` é **derivado** dele e NÃO move a rota. Sintoma: o serviço sobe mas dá 503 (cert 000), e a v4 bootou com `publicUrl` sslip.io.

Fix via `scripts/coolify.py` (base64-pipa o psql; o restart não monta curl com token à mão):

```sh
python3 scripts/coolify.py list-apps --ssh root@<VPS_IP>                                          # ache o id
python3 scripts/coolify.py set-fqdn  --ssh root@<VPS_IP> --app-id <id> --fqdn https://agentes.<seu-dominio>
python3 scripts/coolify.py api-post  --base-url http://<VPS_IP>:8000 --token-file coolify.token --path /services/<uuid>/restart
```

**Preserve a porta** quando o template tem (Langfuse: `...:3000`). Verifique por sslip.io enquanto o DNS não propaga.

### `docker_compose_raw` precisa ser base64

`POST /api/v1/services` com o compose cru → 422 "should be base64 encoded". O `scripts/coolify.py create-service` faz o base64 do `--compose-file`; se POSTar à mão via `api-post`, encode antes.

### NÃO sobrescrever `command:` no compose da v4

O boot (`bootstrap → migrate → serve`) é o CMD da imagem. Um `command:` override já derivou pro `./server` obsoleto e crash-loopou (`exec: ./server: not found`). Não declare `command:` no compose da v4.

### Instance Domain do Coolify

Sem setar o Instance Domain (`coolify.<seu-dominio>`) o painel fica só em `http://IP:8000` (HTTP puro, sem TLS). Exige o A-record.

### `prisma migrate reset` quebra a runtime role (local/dev)

Reset recria o schema `public` e leva junto os grants da app role → próximo boot dá `42501 permission denied for schema public`. Nunca rode bare `migrate reset`; use `bun db:reset` (ou re-rode `bun db:bootstrap`).

### Service compose: introspecção é por filesystem + `docker compose`, não pela fila de deploy

Um **service** compose do Coolify NÃO popula `application_deployment_queues` (essa tabela é de *applications*; consultá-la pra um service devolve fila vazia e te faz achar que o deploy não rodou). O estado de um service vive em `/data/coolify/services/<uuid>/` e valida por `docker compose config`/`docker compose ps`. Pra checar se subiu: `docker compose -p <uuid> ps` (ou `ls /data/coolify/services/<uuid>/`), não a fila.

### O `start` da API pode não materializar containers: NÃO suba por `docker compose` à mão

Num Coolify **recém-instalado** (ou logo após reiniciar o container `coolify`/o Docker), o `POST /api/v1/services/<uuid>/start` pode responder `200 "starting request queued"` e a fila preparar projeto/rede/compose em `/data/coolify/services/<uuid>/` **sem criar os containers** (o worker/Horizon do Coolify não processou o job). A tentação é subir na mão (`docker compose -p <uuid> up -d` no diretório do serviço). **Não faça.** Os containers até sobem (`Up healthy` no `docker ps`), mas ficam **fora da gestão do Coolify**: a UI mostra o serviço **Exited/Degraded** (o Coolify não registrou o deploy), e um Restart/Redeploy pela UI depois **conflita** ou re-cria por cima. Sintoma exato (numa run real): `docker ps` mostra tudo `Up (healthy)`, a UI do Coolify mostra `Exited`. Em vez de contornar: confirme que o container `coolify` está `Up (healthy)` e que **nada o reiniciou no meio do deploy** (evite `docker restart coolify`/`systemctl restart docker` durante o deploy, que zera a fila; faça o Instance Domain ANTES de deployar os serviços, não no meio), e **re-dispare o deploy pela API/UI do Coolify**. Se persistir, investigue o worker do Coolify; nunca substitua o deploy do orquestrador por `docker compose` manual.

### Imagem grande: não faça `docker pull` em foreground (estoura o timeout do harness)

Um `docker pull` de imagem grande (Chatwoot Pro) passa de 3 min e o harness mata o comando (exit 124): você acha que falhou e re-tenta à toa. Para **confirmar auth/existência** sem baixar, use `docker manifest inspect <imagem>` (segundos); **deixe o Coolify puxar** a imagem no deploy (assíncrono), não você em foreground.

### Comandos dentro de container (runner/tinker): via helper que é dono do payload

Não monte one-liners de console (Rails runner do Chatwoot, `artisan tinker`/PsySH) à mão por `ssh … "docker exec … --execute='App\Models\User…'"` **dentro do PowerShell**: o `\` de namespace e as aspas são mangled (PHP `T_NS_SEPARATOR` parse error; echo do PsySH polui o stdout). Use o helper que **é dono do payload** e o passa base64 (como o `scripts/coolify.py` já faz pro psql), sem quoting manual atravessando PowerShell→SSH.

### `docker ps --format '{{…}}'` à mão quebra no PowerShell→SSH

O agente improvisa `ssh … "docker ps --format '{{.Names}}\t{{.Status}}'"` pra ver o que subiu, mas no PowerShell→SSH o `{{…}}` e o `\t` são mangled (vira comando quebrado e você lê "nada rodando" num host que TEM containers). Use o helper que é dono do payload: `scripts/docker-status.py --ssh root@<HOST>` (ou `--project <uuid>` p/ um service do Coolify, `--all` p/ incluir parados). Ele roda o ssh por argv direto (chaves intactas) e devolve JSON normalizado.

### Windows/PowerShell: NUNCA payload inline, sempre arquivo + interpretador (a armadilha #1)

A que MAIS quebra a run, sempre num contexto novo (e os helpers de caso específico **não te cobrem nos ad-hoc**): a cada novo (gerar uma key, um restart, uma query) o agente cai no default e monta o comando como se o PowerShell fosse bash. Modos de falha reais já vistos:

- **aspas comidas:** `ssh <host> '… --format "{{.Name}}" | grep -vE "^(a|b)$"'` → o Windows engole as `"` internas ao repassar pro `ssh.exe`; o bash recebe `{{.Name}}`/`^(a|b)$` **sem** aspas → `syntax error near unexpected token '('`.
- **BOM na here-string:** `@'…set -euo pipefail…'@ | ssh … 'bash -s'` → o PowerShell prefixa um **BOM UTF-8** na 1ª linha; o bash lê `﻿set` como comando inexistente, o guard **não arma**, e o resto (um `rm -rf`) roda sem proteção.
- **`\` de continuação:** quebrar um comando com `\` no fim da linha → no PowerShell continuação é `` ` `` (backtick); a 2ª linha vira comando solto (`O termo '\ && docker restart' não é reconhecido`) e o ssh fica com aspas abertas (`unexpected EOF while looking for matching "`).
- **namespace `\`:** `rails runner`/`tinker` com `App\Models\User` inline → o `\` é comido → `T_NS_SEPARATOR`.
- **payload via pipe:** `'{…json…}' | helper` ou `@'…python…'@ | python -` → here-string/pipe carregam o payload com o BOM/encoding do PowerShell.
- **acentos viram `?`:** qualquer texto com acento montado/pipeado inline (uma mensagem de teste `Olá, quais convênios vocês aceitam?`, o nome de uma conta) volta corrompido (`Ol?, quais conv?nios voc?s aceitam?`): o PowerShell re-encoda o pipe/argv na code page do console, não em UTF-8. O fix é o mesmo: o texto vai num **arquivo UTF-8** e a ferramenta/helper aponta pro arquivo (`--data @msg.json`, `--script-file`, `--json-file`), nunca inline.

Não há heredoc (`<<'EOF'` é só POSIX) nem `<` de stdin no PowerShell, por isso o agente é empurrado pra essas traduções frágeis. **A regra que fecha a classe inteira: o shell só ORQUESTRA (variável, loop, `Start-Sleep`, chamar o interpretador); NUNCA carrega o código.** Escreva o payload num **arquivo** (com a ferramenta de edição, zero shell, sem BOM) e rode apontando pro arquivo:

| Você quer… | NÃO (inline) | SIM (payload em arquivo) |
|---|---|---|
| bash remoto | `ssh <host> '…script…'`, `@'…'@ \| ssh` | `remote.py --script-file x.sh` |
| psql num container | `ssh <host> "docker exec … psql -c \"…\""` | `remote.py --in-container <db> --exec "psql -U u -d d -v ON_ERROR_STOP=1" --script-file q.sql` |
| rails runner / tinker | `ssh <host> "docker exec … rails runner …"` | `remote.py --in-container <c> --exec "bundle exec rails runner -" --script-file t.rb` |
| Python local | `@'…'@ \| python -` | escreva o `.py`, rode `python x.py` |
| JSON de API | `'{…}' \| helper`, `-d '{…}'` | `coolify.py api-post … --json-file x.json` |
| criar/editar arquivo | `echo`/`Set-Content`/`Out-File >` | a ferramenta de edição do agente |
| config da v4 | (qualquer inline acima) | tools de **MCP** |

`remote.py` alimenta o `bash -s` (ou `docker exec -i … psql/runner`) remoto via **stdin por argv direto**: aspas, `$()`, `{{…}}`, `(`, `\`, heredoc e múltiplas linhas chegam **byte a byte em qualquer SO**. Saída ao vivo (instalação longa não estoura timeout), exit code propagado; `--capture` devolve `{ok,exit_code,stdout,stderr}` JSON pra parsear; `--dry-run` mostra o argv. **Só** um comando de **uma linha sem** `"`/`$()`/`{{…}}`/`(`/`\` pode ir inline (`ssh <host> 'hostname; docker ps -q'`). Os helpers de caso específico seguem como atalho (`docker-status.py` p/ `docker ps`, `coolify.py` p/ os fluxos de Coolify, `chatwoot-admin.py` p/ ler o admin/token do Chatwoot); pra **qualquer outro** script remoto/console, `remote.py`.

## Edições (imagens Free/Pro)

### Secretária Pro ≠ licença Chatwoot avulsa (precisa da comunidade)

A edição **Pro** da Secretária V4 (`secretariaEdition: "pro"`, marcador) usa imagem privada no Harbor (projeto `secretaria`), liberada **só pra membros da comunidade** (`isCommunityGrant`). Uma licença Chatwoot Pro **avulsa** NÃO desbloqueia a Secretária. A robot do Harbor é **per-user** (cobre a união dos projetos a que o usuário tem acesso): se Chatwoot e Secretária são ambos Pro, é **um único** `docker login`: não logue duas vezes. `free` = imagem pública, **sem** `docker login`.

### Chatwoot OSS não faz `docker login` nem usa Baileys

`chatwootTier: "community"` (OSS) usa a imagem pública `ghcr.io/fazer-ai/chatwoot` (nosso fork), **sem** `docker login` no Harbor e **sem** o `baileys-api` (`COMPOSE_PROFILES` vazio). Só o `pro` faz `docker login` no Harbor + imagem privada `chatwoot-pro` + Baileys. **Não** rode `docker login` nem provisione credencial do Harbor no caminho OSS (não há licença, e o pull público não precisa dela).

## Langfuse

### One-click sem MinIO = traces somem em silêncio

Langfuse v3 exige S3 blob storage na ingestion. O one-click sobe sem MinIO e com `LANGFUSE_S3_*` vazias → `POST /ingestion` dá HTTP 500 e os traces nunca chegam; o `GET /projects` (só Postgres) retorna 200 e mascara. **Use `templates/langfuse/docker-compose.coolify.yml`** (com MinIO) e valide com `scripts/langfuse-verify.py ingestion` (espere 207, não 500). Detalhe em `references/05-langfuse.md`.

## Config da v4 (pós-import)

### MCP/SUPER_ADMIN: mire o tenant com o argumento `tenant`, não crie um tenant

O token MCP do admin do `/setup` é **fleet-level** (`whoami` → `tenantId: null`): ele não tem tenant embutido. Toda tool per-tenant (agent_import, vault, deployment_connect, …) exige o argumento **`tenant`** (slug ou id de `tenant_list`). O erro clássico: a tool reclama *"fleet-level … pass `tenant`"*/*"no tenant target"* e o agente conclui que **falta um tenant** e chama `tenant_create` → cria um tenant **órfão**, e o import cai no lugar errado. Há **um** tenant (o do `/setup`); rode `tenant_list` e passe o `tenant`. Detalhe em `references/06-setup-and-mcp.md`.

### Embedding é por-tenant (senão os docs vão pra FAILED)

Sem `PUT /v1/tenant-settings/embedding {provider, model, credentialRef}`, os docs da KB vão pra FAILED (`embedding credential not configured`). É no nível do **tenant**, não por-KB nem da chave do modelo do agente.

### `reindex` não recupera docs FAILED

Depois de configurar o embedding, `POST /v1/knowledge/bases/:id/reindex` retorna `{queued:0}` se os docs já estão FAILED. Use `POST /v1/knowledge/documents/:id/retry` por doc.

### agent_import resolve credenciais por nome

O export referencia tudo **por nome**, e os nomes não existem no tenant novo. O `agent_import` cria as credenciais faltantes como **pending + deeplink** e emite o aviso `credentialPending`: o usuário só preenche o segredo. Exceções que não viram pending: kinds de OAuth gerenciado (`google_oauth`, `mcp_oauth`) e kinds que exigem `baseURL`/`paramName` → caem em `credentialNotFound`. Detalhe em `references/08-agent-import.md`.

### Chatwoot bind: `POST /deployment` registra o deployment; quem conecta as CONTAS é o `/accounts`

`POST /v1/chatwoot/deployment` valida o token (via `/profile`), **persiste o deployment** (baseUrl + adminToken criptografado na linha do deployment) e retorna as contas alcançáveis. O que ele **não** faz é conectar/sincronizar as contas individuais: isso é o `PUT /v1/chatwoot/deployment/accounts {accountIds:[...]}`. Depois `PATCH /v1/chatwoot/inboxes/:id {agentId}` provisiona o Agent Bot + webhook (não precisa setar `webhook_url` à mão). Detalhe em `references/09-chatwoot-bind.md`.

## Footguns de API (campos exatos)

### `POST /api/v1/api-keys`: o campo é `displayName`

Mintar a API key da v4 com `{ "name": ... }` → 422. O campo é `displayName`. O `token` vem só uma vez.

### vault POST usa `baseUrl` (camelCase)

`POST /v1/vault` espera `baseUrl` (camelCase); mandar `baseURL` faz a entrada nascer sem base URL e o wiring do Langfuse falha ("requires a base URL"). Atenção: o endpoint `PUT /v1/tenant-settings/embedding` usa a forma `baseURL` (maiúsculo), os dois diferem.

## Ambiente do agente (CLI / box)

### Box bun-only: não chame `node`

A máquina do operador pode ter **só `bun`** (sem `node` no PATH): `node helper.js` dá `CommandNotFoundException`. Não escreva helpers Node ad-hoc e não invoque `node`: rode os scripts com **`bun`** (e prefira os helpers vendorados da skill, `scripts/*.py`/`*.ts`, em vez de improvisar). As ops do hub saem pelo proxy `bunx @fazer-ai/secretaria hub …`, não por um helper Node escrito na hora.

## Pendências conhecidas (não bloqueiam o core)

- **TTS:** precisa de chave ElevenLabs real.
- **Visão:** precisa de chave Gemini válida.
- **WhatsApp físico:** opcional; exige um número que o usuário controle. A integração Chatwoot→v4 já é provada sem aparelho via Inbox API (etapa 10); o físico só confirma o transporte real.
- **Kanban:** condicional à licença, **não** "opcional". Com licença disponível (CLI/`hub licenses`), habilitar é **happy-path** (licenciar no hub + Refresh; ver `references/chatwoot-hub-register.md`); imagem Pro sozinha não basta. Sem licença → OSS, sem Kanban.
