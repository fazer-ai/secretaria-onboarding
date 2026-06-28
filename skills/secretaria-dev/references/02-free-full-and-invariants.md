# 02: Separação Free/Full + invariantes

## Disciplina de marcação Free/Full
O monorepo é só Full por enquanto; o strip (Free) é fase futura. Mas **marque desde já**:
- **Blocos inline:** cerque código exclusivo do Full com `// @full-only` … `// @full-only-end`. O Free remove o bloco e deve continuar compilando sem ele.
- **Pastas inteiras Full:** no Free a pasta vira um `README.md` no mesmo caminho explicando a feature.
- **Aditividade:** schema sempre com `tenant_id`; env vars, endpoints e UI são **só aditivos** (o Full adiciona; nunca remove o que o Free tem).

## Invariantes (leia a doc certa ANTES de mexer no subsistema)
Os pontos fixos do projeto (multi-tenancy/RLS, "um core três transportes", roteamento BrowserRouter, CSP, encryption, i18n, UX/skeletons) estão nas instruções do projeto na raiz e detalhados por subsistema em `docs/` (cada doc cobre um subsistema: tenancy, graph, chatwoot, mcp, logs, etc.). Aponte e leia a doc do subsistema **antes** de editá-lo.
