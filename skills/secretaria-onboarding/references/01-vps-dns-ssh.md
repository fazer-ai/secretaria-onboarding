# 01: VPS + DNS + SSH

## VPS — qual VPS é escolha do usuário (pergunte, não escolha)

O **`<VPS_IP>` vem do usuário**, nunca de um chute. Se ele não disse qual e o MCP lista mais de uma VM na conta (`VPS_getVirtualMachinesV1`), **apresente as opções (id, IP, hostname, plano) e pergunte qual usar** — ter o MCP conectado não autoriza escolher por ele. Confirmada a VPS, **nunca toque em outra** da conta (ver `guardrails.md`). O orquestrador (Coolify, Portainer, outro painel, ou nenhum) já está instalado (brownfield) ou será instalado no deploy do tier (escolhido em 1c).

## SSH: sondar o acesso antes de gerar chave

**Nunca peça "o caminho da chave SSH" de cara.** O acesso ou **já existe** (sonde), ou o operador cadastra uma chave nova **pelo painel do provedor** (Hostinger ou qualquer outro). **Nunca use senha root.** Ordem:

**1. Sonde o acesso** (sem perguntar nada): tente logar com as chaves que o operador já tem (default em `~/.ssh` + agent):
```sh
ssh -o ConnectTimeout=12 -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@<VPS_IP> 'echo OK; hostname'
```
- Saiu `OK` → **há acesso**; siga, **não pergunte nada de chave**. Anote a chave que funcionou para reusar.
- `Permission denied (publickey…)`, exit ≠ 0 → **sem acesso**; vá ao passo 2. (`BatchMode=yes` evita travar pedindo senha; o `dangerouslyDisableSandbox: true` é obrigatório por ser rede.)

**2. Sem acesso: gere a chave e imprima a pública para o operador cadastrar no painel:**
```sh
ssh-keygen -t ed25519 -f ~/.ssh/fazer-ai-<nome> -N "" -C "fazer-ai-onboarding"
cat ~/.ssh/fazer-ai-<nome>.pub
```
Mostre essa linha (`ssh-ed25519 …`) e instrua o operador a colá-la no painel da VPS (você **não** faz isto pela API, ver *Nota MCP*):
- **Hostinger:** painel da VPS → card **"Chave SSH"** → **"Gerenciar"** → **"+ Chave SSH"** → cole a chave pública → **"Salvar"**.
- **Outro provedor:** o equivalente no painel dele ("SSH Keys" / "Add SSH key" da VPS).

**3. Confirme** re-sondando (passo 1) até logar; só então use o **comando de trabalho** (determinístico) no resto do fluxo:
```sh
ssh -o IdentitiesOnly=yes -o IdentityAgent=none -o ConnectTimeout=12 -o BatchMode=yes \
    -o StrictHostKeyChecking=accept-new -i ~/.ssh/<sua-chave> root@<VPS_IP>
```
Bash com rede → `dangerouslyDisableSandbox: true`. Scripts/SQL longos: base64 local → pipe → `base64 -d` no destino.

### Nota MCP: cadastre a chave pelo painel, não pela API
A API de chaves não serve aqui: `attach`/`create` registram a chave mas não a injetam numa VM em execução (só aplicam em provisionamento/`recreate`, que apaga dados). Por isso a skill cadastra a chave **pelo painel da VPS** e confirma o acesso **por sondagem** (passo 3).

## DNS (MCP Hostinger, domínio `<seu-dominio>`)

O **domínio raiz (`<seu-dominio>`) é escolha do usuário**: liste os domínios da conta (`domains_getDomainListV1`) e **pergunte qual usar como raiz** — nunca assuma um porque "estava na conta". Definido o raiz, crie os A-records apontando pra `<VPS_IP>` (os três da app são o contrato; ver 1c):
- `agentes.<seu-dominio>`: Secretária V4
- `chatwoot.<seu-dominio>`: Chatwoot (Pro ou OSS)
- `langfuse.<seu-dominio>`: Langfuse (recomendado)
- **painel do orquestrador** (se houver e você quiser um domínio limpo): `coolify.` (Tier A) / `portainer.` (Tier B); outro painel usa o próprio; no compose genérico (Tier C) pode não haver painel.

Tools do `hostinger-dns`: `DNS_getDNSRecordsV1` (inspecionar), `DNS_updateDNSRecordsV1` (setar). **Monitore a propagação** antes de prosseguir: o ACME (Traefik do Coolify, Caddy do Portainer, ou o proxy do tier) só emite o certificado quando o A-record resolve. Sem isso, os serviços sobem mas ficam 503/sem TLS.

## Outro provider (VPS/DNS fora da Hostinger)

Se o usuário usa outro provider de VPS e/ou DNS, **não há MCP da Hostinger**. Pergunte qual provider ele usa e conduza com base no seu conhecimento dele. Só o **provisionamento de VPS/DNS** muda de ferramenta; do SSH em diante (deploy do tier, v4, branding, bind) o fluxo é idêntico.

- **DNS:** crie os **mesmos A-records** (`agentes.`/`chatwoot.`/`langfuse.` + o painel do tier → IP da VPS) pelo painel/CLI/API do provider do usuário. Monitore a propagação igual (o ACME só emite o cert quando o A-record resolve).
- **VPS:** o usuário cria a VPS no provider dele e fornece **IP + chave SSH**. Confirme que a porta 22 está aberta e que dá pra logar como root (ou com sudo). O resto da `01` (comando SSH, base64-pipe) vale igual.
- **Sem VPS ainda?** Sugira adquirir (recomendado: Hostinger, [link de parceiro fazer.ai](https://www.hostg.xyz/SHJfs), cupom `FAZERAI` = 10% de desconto na primeira compra). Detalhe na `00-prereqs-and-access.md`.
