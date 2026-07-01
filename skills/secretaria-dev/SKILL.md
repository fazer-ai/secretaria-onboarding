---
name: secretaria-dev
description: "Modo desenvolvedor da Secretária V4: trabalhar no código-fonte. Clona o repo (Free público ou Pro via git proxy do hub), orienta nas boas práticas e invariantes (separação Free/Full via markers @full-only, convenções de docs/), pergunta proativamente o que o usuário quer implementar e conduz a implementação, e ajuda a gerar a própria imagem de deploy. Use quando o usuário quer DESENVOLVER, estender ou contribuir com o código da Secretária V4, não apenas subir (onboarding) ou operar (operação) uma instância."
---

# Modo desenvolvedor da Secretária V4

Leva um desenvolvedor de "quero mexer no código" até "implementou com as boas práticas do projeto e, se quiser, gerou a própria imagem". Audiência: **desenvolvedor**, não operador. Para subir uma instância do zero use a skill `secretaria-onboarding`; para debugar/ajustar uma instância em produção use `secretaria-operation`.

## ⚠️ Distribuição: o Pro é privado

- O **repositório Pro/Full** (`fazer-ai/secretaria-v4`) e a **imagem Pro** são **privados**. **Nunca** publique o código, a imagem, ou trechos exclusivos do Full em local público (gist, fork público, registry público, post, screenshot).
- O acesso ao Pro é concedido individualmente. Vazar repo/imagem quebra esse modelo.
- **Sugestões e contribuições vão para o repositório Free** (open-source): `fazer-ai/secretaria-ia` (provisório; o nome client-facing ainda não foi decidido). Abra issues/PRs lá.

## Fluxo

1. **Obter o código** (`references/00-get-the-code.md`) — bifurca por edição: Free clona o repo público (sem credencial); Pro clona via git proxy do hub (credencial per-user).
2. **Layout + porta de qualidade** (`references/01-layout-and-bun-check.md`) — mapa do repo, `bun install`/`bun dev`, o ciclo `bun check`.
3. **Free/Full + invariantes** (`references/02-free-full-and-invariants.md`) — markers `@full-only`, aditividade, e onde estão as convenções e os docs por subsistema.
4. **Implementação conduzida** (`references/03-implement.md`) — perguntar o que implementar, desenhar desafiando premissas, implementar no estilo vizinho, validar.
5. **Imagem própria + deploy** (`references/04-own-image-and-deploy.md`) — gerar a própria imagem e plugá-la num deploy (casa com o Tier C da `secretaria-onboarding`).

## Guardrails

Ver `guardrails.md` (Pro privado, segredos fora do repo, `bun check` verde) e `gotchas.md` (armadilhas de DB/RLS/CSP).

## Skills irmãs

- `secretaria-onboarding`: subir uma instância nova num VPS (do zero ao agente).
- `secretaria-operation`: debugar/ajustar uma instância **em produção**.

## Pendências

- Confirmar o **nome client-facing** e a URL definitiva do repo Free (hoje provisório: `fazer-ai/secretaria-ia`).
- **Caminho Pro depende do hub:** o git proxy do código-fonte (`fazer-ai/secretaria-v4`) ainda será exposto; até lá, o clone Pro de `references/00-get-the-code.md` é o contrato assumido.
- **Alerta de não-redistribuição no repo Pro:** montar **antes de gerar os repos separados Free/Full** (wording a aprovar por ser conteúdo público).
