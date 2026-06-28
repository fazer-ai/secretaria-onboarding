# 01: Layout do repo + porta de qualidade

## Mapa
- `src/api/` — backend Elysia (features/, lib/, middlewares/).
- `src/client/` — frontend React (pages/, components/, contexts/, lib/, locales/).
- `prisma/` — schema e migrations.
- `docs/` — guia por subsistema (uma doc por subsistema; leia a relevante antes de mexer).
- `scripts/` — utilitários (`set-admin.ts`, etc.).
- `public/` — assets e `index.html`.

## Subir local
```sh
bun install
bun dev          # hot reload, porta 3000
```
Antes de configurar `DATABASE_URL` no `.env`, cheque PostgreSQL já em uso (`ss -tlnp | grep 543`); o default é 5432, senão use a próxima porta livre via `POSTGRES_PORT`.

## Porta de qualidade
```sh
bun check        # lint Biome + tsc + i18n + testes
```
Rode **depois de todas as mudanças**; só conclua com `bun check` verde. Nunca rode um `prisma migrate reset` cru (apaga os grants do role runtime); use `bun db:reset`.

As convenções completas (estilo, theming, CSP, encryption, i18n, UX) estão nas instruções do projeto (carregadas automaticamente ao abrir o repo) e nos docs em `docs/`.
