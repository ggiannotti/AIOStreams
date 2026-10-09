# AIOStreams

Stremio "super-addon": agrega addons Stremio upstream + debrid/usenet, depois deduplica, filtra, ordena, formata e (opcional) faz proxy dos streams. Processo Node único; também serve SPA React de configuração.
Requer Node `>=24`, pnpm `>=11` (engines do `package.json` raiz).

## Comandos (da raiz; workspace pnpm `packages/*`)

| Comando | Efeito |
|---|---|
| `pnpm install` | instala tudo |
| `pnpm build` | core → server → frontend → jellyfin-web → seanime-extensions (ordem importa; dependem de `@aiostreams/core`) |
| `pnpm dev` | core, server, frontend, jellyfin-web em watch paralelo |
| `pnpm start:dev` | `tsx watch` de `packages/server/src/server.ts`, `NODE_ENV=development` (só backend) |
| `pnpm start` | `node packages/server/dist/server` (exige build) |
| `pnpm start:frontend:dev` | só dev server rsbuild da SPA |
| `pnpm test` | `vitest run --passWithNoTests` em cada workspace |
| `pnpm -F <pkg> test\|typecheck\|lint` | por pacote (ex. `pnpm -F core test`) |
| `pnpm -F <pkg> exec vitest run path/x.test.ts [-t "nome"]` | teste único |
| `pnpm format` | Prettier em `*.ts`/`*.tsx` |
| `pnpm gen:env-docs` | regenera docs de env a partir do schema; rodar após mudar `packages/core/src/config/schema` |
| `pnpm metadata` | regenera metadata de addons (`scripts/generateMetadata.cjs`) |
| `pnpm docs:dev` / `docs:build` | site de docs |

Frontend usa rsbuild (não Vite/webpack direto): `pnpm -F frontend dev|build|preview`.

## Workspace

| Pacote | Papel |
|---|---|
| `core` | engine: addons, I/O, DB, cache, config, presets, builtins, pipeline de streams (`@aiostreams/core`) |
| `server` | Express 5 fino ligando core ao HTTP: rotas, middleware, rate limit, estáticos, lifecycle |
| `frontend` | SPA React 19 (rsbuild, TanStack Router+Query, Tailwind, Radix); build em `packages/frontend/dist`, servido pelo server |
| `ui` | `@aiostreams/ui`: kit compartilhado (preset Tailwind, `theme.css`, componentes, hooks, format). Só fonte, sem build; importado como `@aiostreams/ui/<componente>` por `frontend` e `jellyfin-web`, cujo Tailwind `content` deve incluir `../ui/src` |
| `jellyfin-web` | app web em `/web` de cada mount Jellyfin (rsbuild, `dist` próprio). Server envia seu `index.html` por mount e serve assets hashed em `/jellyfin-web/static`. Para servers Jellyfin tipo AIOStreams (só direct play, campos opcionais `aiostreams`), não cliente Jellyfin geral. `build:standalone` (`--env-mode standalone`, const `__STANDALONE__`) gera cópia que escolhe o próprio server, em `dist-standalone`; o app desktop serve essa |
| `desktop` | app Windows/Linux/macOS: workspace Cargo fora do build pnpm; serve o standalone do `jellyfin-web` em webview transparente (WebView2 tao/wry; WebKitGTK 6 via GTK 4; WKWebView tao/wry) sobre libmpv; expõe bridge JSON checada ao mpv como `window.aiostreamsDesktop`. Ver README dele |
| `seanime-extensions` | bundles de extensão Seanime, build independente |
| `docs` | site de docs, build separado |

## Fluxo de request

1. `packages/server/src/server.ts` boot: DB → templates → Redis (se configurado) → AnimeDatabase / SeaDex / indexers pré-configurados Prowlarr → jobs `TaskManager` (prune de usuários, evict de cache) → analytics → `app.listen`.
2. `packages/server/src/app.ts` monta:
   - `/api/v{API_VERSION}/*`: JSON da SPA (`user`, `health`, `status`, `format`, `catalogs`, `posters`, `oauth/exchange/gdrive`, `debrid`, `search`, `anime`, `proxy`, `templates`, `sync`, `auth`, `dashboard`); handler 404 restrito ao router da API.
   - `/stremio/...`: protocolo Stremio. manifest/stream/configure públicos direto; autenticados em `/stremio/:uuid/:encryptedPassword` via `userDataMiddleware` (resolve `UserData` p/ o pipeline).
   - `/chilllink/:uuid/:encryptedPassword/*`, `/seanime/*`, `/builtins/*` (este atrás de `internalMiddleware`).
   - Legado `/:config/stream/...` devolve 1 stream "reconfigure" apontando p/ nova URL; `/configure` redireciona p/ `/stremio/configure`.
   - Estático: `/assets/*` hashed, cache `immutable`, ignora static rate limiter; `/logo.png`, favicons, ícones do manifest passam por `staticRateLimiter`; fallback SPA = `index.html`.
3. Stream request cria instância `AIOStreams` (`packages/core/src/main/index.ts`) do `UserData`. Construtor liga singletons: `Proxifier`, `StreamLimiter`, `StreamFilterer`, `StreamPrecomputer`, `StreamFetcher`, `StreamDeduplicator`, `StreamSorter`. Helpers de `setup.ts` (`applyPresets`, `assignPublicIps`, `fetchManifests`, `buildResources`) viram presets do usuário em `Addon` + manifests; `resources.ts`/`catalog.ts` implementam `getStreams`/`getCatalog`/`getMeta`/`getSubtitles`/`getAddonCatalog`.

## Presets vs builtins

- **Presets** (`packages/core/src/presets/*.ts`, ~80, registrados em `presetManager.ts`): como configurar/chamar addon Stremio externo (Torrentio, Comet, MediaFusion, Easynews…). Estendem base `Preset` (`preset.ts`) e expõem metadata p/ a SPA. Addon comunitário novo = arquivo de preset + registro em `presetManager.ts`.
- **Builtins** (`packages/core/src/builtins/*`): addons in-process (gdrive, knaben, prowlarr, torznab/newznab, eztv, torrent-galaxy, seadex, easynews-search, library…). Expostos em `/builtins/<nome>` via `packages/server/src/routes/builtins`; `internalMiddleware` só deixa o próprio engine chamar.

## Configuração

- `packages/core/src/config`: `bootstrap.ts` lê env via `envalid`; `schema/` define user-data com zod; `describe.ts` gera metadata p/ SPA e `pnpm gen:env-docs`. Mudou env var ou schema → `pnpm gen:env-docs` + rebuild.
- Config por usuário em `packages/core/src/db` (repositórios + migrations); driver comum SQLite (`better-sqlite3`) / Postgres (`pg`), escolhido por `DATABASE_URI`.
- Cache: 3 backends, 1 API `Cache`: memória, SQL (DB configurado), Redis (`REDIS_URI`). Tasks de limpeza em `server.ts` refletem isso.

## Frontend

- Rotas: TanStack Router file-based em `packages/frontend/src/routes`; `router.tsx` conecta.
- Dados: TanStack Query em `/api/v{N}/...`; SPA nunca chama `/stremio/*`.
- UI: `@aiostreams/ui` (Tailwind 3, Radix, `class-variance-authority`); animação `framer-motion`/`motion`.

## Release

`release-please-config.json` (conventional commits → versão → changelog). Local: `pnpm release` (commit-and-tag-version). Docker: CI `.github/workflows/deploy-docker.yml`.

## Convenções

- ESM (`"type": "module"`). Imports relativos no pacote levam extensão `.js` (NodeNext), ex. `from '../db/index.js'` mesmo com fonte `.ts`.
- Cross-package: `from '@aiostreams/core'`.
- Logger: `createLogger('<scope>')` de `@aiostreams/core` (Pino); sem `console.log`.
- Erro que impede startup: `ConfigStartupError` (`server.ts` imprime sem stack).
