# Dynatrace MCP Server — copilot instructions

Short, actionable guidance for AI coding agents working on this repo.

1. Big picture
  - This is a TypeScript Node.js MCP (Model-Context-Protocol) server that exposes a set of "tools" (MCP capabilities) to consumers.
  - `src/index.ts` boots the MCP server, wires transports (stdio or HTTP), registers tools and handles startup checks (Dynatrace connection test, telemetry, graceful shutdown).
  - Each capability lives in `src/capabilities/*.ts`. Capabilities typically call into Dynatrace via an authenticated `HttpClient` created by `src/authentication/dynatrace-clients.ts`.

2. Key patterns and examples
  - Tool registration uses a small helper in `src/index.ts`: each tool is defined with a Zod params schema and a callback that returns text content. Example: the `execute_dql` tool (see `src/index.ts`) expects `{ dqlStatement: string }` and calls `executeDql` in `src/capabilities/execute-dql.ts`.
  - Always use Zod shapes for tool params (the server relies on these for validation). Example snippet: tool('list_vulnerabilities', ..., { riskScore: z.number().optional().default(8.0) }, async ({ riskScore }) => ... )
  - Create authenticated clients via `createDtHttpClient(environment, scopes, clientId, clientSecret, dtPlatformToken)` from `src/authentication/dynatrace-clients.ts`. When adding a tool, add only the minimal extra scopes required by that tool (see how `execute_dql` concatenates many `storage:*:read` scopes).
  - Error handling: check for Dynatrace SDK errors using `isClientRequestError(error)` imported from `@dynatrace-sdk/shared-errors` and return user-friendly messages using the handler in `src/index.ts`.
  - Environment detection: `src/getDynatraceEnv.ts` enforces required env vars (DT_ENVIRONMENT) and defaulting logic (e.g., grail budget). Read it before assuming envs are present.

3. Build, test & run (developer flows)
  - Build: `npm run build` (runs `tsc --build`). Output: `dist/`.
  - Run locally (compiled): `npm start` will run `dist/index.js` (binary `mcp-server-dynatrace`).
  - Dev/watch: `npm run watch` (tsc watch).
  - Tests: `npm test` (jest). Integration tests: `npm run test:integration` (runs in-band). Unit tests: `npm run test:unit`.
  - Run the server in HTTP mode for remote testing: `node ./dist/index.js --http --port 3000` (or use the `--server` alias). By default the server runs in stdio mode.

4. Project-specific conventions
  - Tool callbacks should return user-facing text via the `tool` helper (not raw objects). The helper wraps the response into MCP `CallToolResult` and handles telemetry and errors.
  - When creating a `dtClient`, use `scopesBase.concat(...)` to extend minimum scopes. Keep scope changes minimal and document them in `README.md` and in `CHANGELOG.md` when adding features that require new scopes.
  - DQL guidance: capabilities that execute DQL must handle cost/scan-size reporting (see `execute_dql` which warns based on `scannedBytes`). Preserve similar cost-awareness in new DQL-capable tools.
  - Proxy configuration: network proxies are configured by `src/utils/proxy-config.ts` (calls `setGlobalDispatcher` for undici). Call it early if you add networking logic that must respect system proxies.

5. Integration points & external dependencies
  - Dynatrace: uses OAuth or platform token via `createDtHttpClient`. Many tools call `@dynatrace-sdk/*` clients (query, automation, davis-copilot). Respect token scopes and auth flow.
  - MCP SDK: `@modelcontextprotocol/sdk` is used for server/transport and typing. Understand the difference between `StdioServerTransport` and `StreamableHTTPServerTransport` as used in `src/index.ts`.
  - Telemetry: `src/utils/telemetry-openkit.ts` handles OpenKit telemetry initialization and error tracking.

6. When adding features or fixing bugs (practical checklist)
  - Update `src/capabilities/*` and register new tool in `src/index.ts` using the existing `tool(...)` pattern (Zod schema + async handler).
  - Add any new Dynatrace scopes only when necessary; update `README.md` with required scopes and add an entry to `CHANGELOG.md` under `## Unreleased Changes` describing the addition.
  - Run `npm run build` and `npm test` (and integration tests if your change touches network/auth) before submitting a PR.
  - Preserve user-friendly error messages by leveraging `isClientRequestError` and the existing `handleClientRequestError` helper.

7. Useful file references
  - Startup / registration: `src/index.ts`
  - Auth / clients: `src/authentication/dynatrace-clients.ts`
  - Env validation: `src/getDynatraceEnv.ts`
  - Capabilities examples: `src/capabilities/execute-dql.ts`, `src/capabilities/list-problems.ts`, `src/capabilities/list-vulnerabilities.ts`
  - Proxy handling: `src/utils/proxy-config.ts`
  - Tests: see `integration-tests/` and `src/*/*.test.ts` for patterns

If any of these areas are unclear or you'd like more examples (e.g., a minimal new-tool scaffolding file), tell me which part to expand and I'll iterate. 
