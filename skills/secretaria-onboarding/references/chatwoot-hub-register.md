# Licenciar a instância Chatwoot no hub (Kanban/Pro)

Habilita as features **Pro** (Kanban etc.) numa instância Chatwoot **Pro já deployada**. Duas coisas são
necessárias e **distintas**:

- A **imagem Pro** (`harbor.fazer.ai/chatwoot/fazer-ai/chatwoot-pro`) traz o **código** (driver de Kanban,
  Baileys). Sem ela, não há Pro. A edição é escolhida no deploy (ver [`../deploy/chatwoot/README.md`](../deploy/chatwoot/README.md)).
- A **assinatura no hub** dá a habilitação em runtime: **imagem Pro sem assinatura ativa = features
  travadas.** Por isso existe o passo do Refresh (não é restart de container).

## Quando: happy-path se há licença

A edição é decidida **no deploy** pelo marcador do CLI `~/.fazer-ai/onboarding.json`
(`chatwootTier` + `chatwootLicenseId`), com fallback pro `list_licenses` no hub se o marcador faltar:
- **`chatwootTier: "pro"`** (ou, sem marcador, há licença CHATWOOT no hub) → deploy da **imagem Pro** e **estes passos são happy-path**: registrar + atachar (use `chatwootLicenseId`) + Refresh pra ligar o Kanban. Não pule.
- **`chatwootTier: "community"`** (ou, sem marcador, sem licença) → deploy da imagem **OSS** (sem Kanban) e segue; nada a fazer aqui.

> **Sem licença e o usuário quer Kanban?** Sugira virar membro Pro da comunidade do Lucas Moreira
> ([lucasmoreira.ai](https://lucasmoreira.ai)) — ganha licença grátis do Kanban (1 conta no plano mensal,
> 2 ilimitadas no anual). Depois é só rodar o CLI de novo e escolher "já me tornei membro" pra a nova
> licença aparecer; ou seguir em OSS.

> Pré-requisitos: `FRONTEND_URL` setado no container do Chatwoot (vira o host que identifica a instância e
> gateia o Refresh). Hub MCP `app-fazer-ai` autorizado (se o token expirou, re-autorize o OAuth). Writes do
> hub são **dry-run por padrão**; mexa só na sua própria licença/instância (ver `guardrails.md`).

## Passos

1. **Instância no hub.** Confira se já existe (`list_instances` / `get_license <id>`): o
   `generate_install_script` do deploy pode já tê-la provisionado. Se faltar:
   `create_instance { identifier: "<slug>" }` (dry-run, depois apply). Use a identidade que o hub casa com
   a instância (o subdomínio/host); confirme com `get_instance`.
2. **Atacha a licença.** `attach_license { license_id: "<licença CHATWOOT>", instance_id: "<id>" }`
   (dry-run, depois apply). Uma feature por instância; os tipos têm que bater.
3. **Refresh na instância** (o botão "Refresh" do super admin, via rails runner no container do Chatwoot):
   ```sh
   docker exec -i <container-rails-do-chatwoot> bundle exec rails runner \
     'Internal::CheckNewVersionsJob.perform_now(jitter_applied: true)'
   ```
   **`jitter_applied: true` é obrigatório.** Sem ele, o job só se reagenda (janela determinística de até 30
   min) e o sync da assinatura nem roda.
4. **Verifique** (mesmo runner):
   ```ruby
   InstallationConfig.find_by(name: 'FAZER_AI_SUBSCRIPTION_SYNC_ERROR_MESSAGE')&.value  # nil = ok
   InstallationConfig.find_by(name: 'FAZER_AI_SUBSCRIPTION_VERIFIED_AT')&.value         # timestamp recente = ok
   ```
   No super admin (`/super_admin/settings`), "fazer.ai Subscription" fica ativa e o Kanban aparece.

## Erros comuns

- **Kanban não aparece com imagem Pro:** faltou o passo 3. A imagem traz o código; a assinatura libera em
  runtime. Rode o Refresh.
- **`FRONTEND_URL` vazio:** o controller do Refresh recusa, e o `installation_host` enviado ao hub fica
  vazio. Sete antes.
- **403 / inativo no `/api/ping`:** a licença não está atachada à instância certa no hub. Confira
  `get_license` / `get_instance` (o casamento é por identifier/host).
- **Assinatura "out of sync" > 3 dias:** o job auto-desativa (`auto_deactivate_if_stale`). Rode o Refresh
  pra re-sincronizar.

OSS não tem nada disso (sem Kanban). Migrar OSS → Pro = re-deploy com a imagem Pro + estes passos.
