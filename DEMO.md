

## Vorbereitung

- chatgpt.com: MCP-App installiert -> https://chatgpt.com/plugins?directoryTab=personal
- claude.ai: https://claude.ai/customize/connectors/yours
  - MCP und MCP App connected
  - In Promt: MCP aktiv, MCP App inaktiv
- URLs:
  - https://web.transgourmet.ch/de/webshop/
  - https://mcp-webshop-demo.fly.dev/
  - MCP Spec: https://modelcontextprotocol.io/specification/2026-07-28
  - MCP Apps: https://modelcontextprotocol.io/extensions/apps/overview
  - WebMCP Spec: https://webmachinelearning.github.io/webmcp/
  - WebMCP Chrome-Dokumentation: https://developer.chrome.com/docs/ai/webmcp (Imperative API: https://developer.chrome.com/docs/ai/webmcp/imperative-api, Declarative API: https://developer.chrome.com/docs/ai/webmcp/declarative-api)
  - WebMCP declarative Demo:  https://googlechromelabs.github.io/webmcp-tools/demos/french-bistro/









## WebMCP

Smart Browsers:

- Load: https://mcp-webshop-demo.fly.dev/

- Dia: "Suche nach Milch"

- Claude Extension in Chrome: "Suche nach Milch"

  - Zuerst sicherstellen dass die Connectors in claude.ai disconnected sind!

    https://claude.ai/customize/connectors/yours

- Flag: `chrome://flags/#enable-webmcp-testing`

- Chrome Dev Tools -> Application -> WebMCP

- Developer Console: 

  ```
  var tools = await document.modelContext.getTools();
  document.modelContext.executeTool(tools[4], {term: 'Milch'});
  
  
  document.modelContext.getTools(); // Promise<RegisteredTool[]>
  document.modelContext.executeTool(tool, params); // Promise<string | null>
  document.modelContext.registerTool(tool, { signal }); // Promise<void>
  ```

  

- Claude Extension:  "Suche nach Wurst. Benutze WebMCP"
  - Eigener Skill: https://claude.ai/customize/skills/yours

- Tool Console: `Ctrl+Shift+Alt+W`





## **Code Walk:**

#### Tool-Handler

[`src/lib/tools/handlers.server.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L48), das Objekt `toolHandlers`:

- [`searchProducts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L49)
- [`getCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L69)
- [`addToCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L73)
- [`removeFromCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L89)
- [`checkout`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/handlers.server.ts#L100)

Die Handler sind protokollneutral: Sie kennen weder MCP noch Session, rufen [`lib/shop.server.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/shop.server.ts) auf und geben `{ ok: false, error }` zurück statt zu werfen. MCP-Server und In-App-Chat ([`features/chat/server/ai-tools.server.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/chat/server/ai-tools.server.ts)) nutzen beide diese Handler.

Für `getOrders` gibt es dort keinen eigenen Handler. Das Tool baut sein Resultat aus `toolHandlers.getCart` plus [`cartPayload()`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L100), und `cartPayload()` ruft `shop.getOrders()` direkt auf.

Beschreibungen und Zod-Schemas der Tools sind davon getrennt und liegen in [`src/lib/tools/contracts.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/lib/tools/contracts.ts).

#### Registrierung

[`src/features/mcp/server.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts):

- [`buildMcpServer()`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L116) erzeugt den `McpServer` und ruft `registerShopTools()` auf.
- [`registerShopTools(server, uiMeta?)`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L123) enthält die 6 `server.registerTool(...)`-Aufrufe:
  - [`searchProducts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L128)
  - [`getCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L150)
  - [`addToCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L171)
  - [`removeFromCart`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L197)
  - [`checkout`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L220)
  - [`getOrders`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L250)
- Jeder Callback prüft die `loginId` über [`withAccount()`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L84), ruft den passenden `toolHandlers.*` auf und formt das Resultat mit [`toolResult()`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/server.ts#L54) zu `content` plus `structuredContent`.

#### Anbindung an die Endpunkte

- **`/mcp`:** [`src/routes/mcp.tsx`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/routes/mcp.tsx) → [`features/mcp/handler.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp/handler.ts#L9) (`createMcpHandler(buildMcpServer)`)

- **`/mcp-app`:** [`src/routes/mcp-app.tsx`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/routes/mcp-app.tsx) → [`features/mcp-apps/handler.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp-apps/handler.ts#L7) → [`features/mcp-apps/app-server.ts`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp-apps/app-server.ts#L11). Dort wird derselbe `registerShopTools()` mit UI-Metadaten aufgerufen, zusätzlich [`registerUiResources()`](https://github.com/ivorycode/mcp-baernerjs-2026/blob/main/mcp-webshop-demo/src/features/mcp-apps/register-ui-resources.ts).

  



## Lokal mit dem MCP Inspector testen

Im `mcp-webshop-demo` einmalig `npm ci`. 

Dann jeweils ein eigenes Terminal:

```bash
npm run mock        # Mock-Katalog auf http://localhost:4040 (aus _mock-api)
npm run dev         # Webshop + MCP auf http://localhost:3052
npm run inspector   # MCP Inspector (Web-UI) mit den Zielen aus inspector.json
```

Der Inspector öffnet sich im Browser (`http://127.0.0.1:6274?MCP_INSPECTOR_API_TOKEN=…`; den Link mit Token aus der Terminal-Ausgabe verwenden).

Ziele aus `inspector.json`:

- `webshop-local`: `http://localhost:3052/mcp`, nur Tools (Text), keine Resources
- `webshop-local-app`: `http://localhost:3052/mcp-app`, Tools plus MCP-App-Resources `ui://webshop/search-ui.html` und `ui://webshop/cart-ui.html`
- `webshop-remote` / `webshop-remote-app`: dasselbe gegen https://mcp-webshop-demo.fly.dev

Ablauf im Inspector:

1. Server `webshop-local-app` auswählen und verbinden.
2. Tools auflisten → 6 Tools: `searchProducts`, `getCart`, `addToCart`, `removeFromCart`, `checkout`, `getOrders`.
3. `searchProducts` mit `term = Milch` ausführen → Produktsuche als MCP App (Sandbox auf Port 6275).
4. `addToCart` mit `loginId = restaurant-baeren`, `articleNumber = 022600`, `quantity = 2`.
5. `getCart` / `checkout` / `getOrders` mit `loginId = restaurant-baeren` → Warenkorb-App.
6. Resources: die beiden `ui://webshop/...`-Resources lesen (`text/html;profile=mcp-app`).

Demo-Konten: `restaurant-baeren`, `hotel-alpenblick`, `kantine-campus`. Warenkorb und Bestellungen liegen nur im Speicher und sind nach einem Neustart von `npm run dev` weg.

Hinweis: Dem Inspector fehlen die Host-Capabilities `updateModelContext` und `message`. Die MCP Apps funktionieren trotzdem, nur die Kontextmeldungen bzw. Chat-Nachrichten an den Host entfallen.

Ohne laufenden Mock liefern die Tool-Aufrufe „Katalog-API nicht erreichbar (http://localhost:4040/…)“.

Alternativ ohne UI über die Inspector-CLI:

```bash
npx @modelcontextprotocol/inspector@latest --cli --config inspector.json \
  --server webshop-local --method tools/list

npx @modelcontextprotocol/inspector@latest --cli --config inspector.json \
  --server webshop-local --method tools/call \
  --tool-name searchProducts --tool-arg term=Milch

npx @modelcontextprotocol/inspector@latest --cli --config inspector.json \
  --server webshop-local --method tools/call \
  --tool-name addToCart --tool-arg loginId=restaurant-baeren articleNumber=022600 quantity=2
```

Variante über stdio (ohne `npm run dev`, Mock muss trotzdem laufen):

```bash
npx @modelcontextprotocol/inspector@latest --cli --method tools/list node src/features/mcp/stdio.ts
```
