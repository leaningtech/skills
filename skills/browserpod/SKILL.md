---
name: browserpod
description: BrowserPod, the microVM that runs in the browser. Adds a Linux environment to a web app — bash, git, npm, Node.js 22 and Rust programs on a WebAssembly kernel, in a browser tab — that boots with no API key or account. Go is coming soon. Covers booting a pod, writing files, running programs in a terminal, networking with and without BrowserPod Edge, Portals, cross-origin isolation, native npm dependencies and the licence's attribution. Use when a user wants code, a dev server, a CLI or an AI agent to run client-side in the browser, asks whether or how BrowserPod fits, hits a BrowserPod error, or asks which BrowserPod licence or plan applies.
---

# BrowserPod

BrowserPod is the microVM that runs in the browser: a Linux environment in a
browser tab. Programs such as `node`, `bash`, `git` and Rust binaries are
compiled from unmodified source to WebAssembly and run as processes on a
Linux-like kernel, client-side. A whole toolchain works in a pod:
`npm install && npm run dev`, `git clone`, CLIs. Nothing runs on a server, and
the user's files stay on their device.

The package is `@leaningtech/browserpod` on npm. It loads the runtime from
BrowserPod's CDN. A pod boots with no API key and no account.

## Ground rules

- **The docs hold the facts:** https://browserpod.io/docs/ (every page as
  Markdown: https://browserpod.io/docs/llms.txt). This skill names the core
  calls and the decisions; for exact signatures, options and the current
  version, read the Reference before writing code, and use only documented
  options. If this skill and the docs disagree, the docs win.
- **Boot without an API key** unless the user asks for BrowserPod Edge or the
  app needs what only Edge gives (§4).
- **Inspect before changing anything.** Say what you will add and where.
- **Stop and ask the user before you:**
  - create an account or an API key, or put a key in code;
  - change server or production configuration (headers, proxies);
  - copy, vendor or self-host the runtime's files (§6);
  - choose or buy a plan or a licence.

## 1. What runs, and what doesn't

| Runtime                                                         | Notes                                                                                       |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Node.js 22 (the only version)                                   | The real Node binary; JavaScript runs on the browser's engine. npm, npx, pnpm and Yarn work |
| Rust programs                                                   | Unmodified CLIs. A Rust toolchain can be installed in a pod (docs guide)                    |
| bash, git, curl, ripgrep and common command-line tools          | In the base image                                                                           |
| Python                                                          | **Preview only**: no native extension modules. Don't promise real Python apps               |
| Go                                                              | Coming soon. Not available yet: check the docs for the current status                       |
| Ruby, Rails, Java                                               | Not available in BrowserPod. (Java in the browser is CheerpJ, a separate product)           |
| Prebuilt native binaries (`.node` addons, platform executables) | Don't run. Swap for Wasm builds (§5)                                                        |

- **Browsers:** Chromium-based browsers give the best results. Firefox and
  Safari work with some limitations, and mobile devices suit lighter workloads.
- **Memory:** a pod has roughly 1 GB available, which suits most projects; very
  large repositories or builds may need trimming.

## 2. Add BrowserPod to a page

1. `npm install @leaningtech/browserpod`.
2. **Serve the page cross-origin isolated.** Every response for the page needs:
   - `Cross-Origin-Opener-Policy: same-origin`
   - `Cross-Origin-Embedder-Policy: require-corp`

   Without them the pod cannot boot (no `SharedArrayBuffer`). In Vite, set
   them in `server.headers`; in production, on the web server or host (the
   docs guide "Set CORS and COOP headers" has Vite and nginx examples).
   Third-party scripts and images without CORP headers will be blocked on that
   page: check what else the page loads.

3. Boot, open a terminal, write files and run a program:

```js
import { BrowserPod } from "@leaningtech/browserpod";

// No API key: the pod boots on a fetch network (§4)
const network = await BrowserPod.createFetchNetwork({
  fetch: BrowserPod.opinionatedFetch, // lets npm install work
});
const pod = await BrowserPod.boot({ network });

const terminal = await pod.createDefaultTerminal(
  document.getElementById("console"),
);

await pod.createDirectory("/project");
const file = await pod.createFile("/project/main.js", "utf-8");
await file.write('console.log("Hello from a pod");');
await file.close();

const exitCode = await pod.run("node", ["main.js"], {
  terminal,
  cwd: "/project",
});
```

- `run(executable, args, { terminal, env, cwd, echo })` starts one program
  (not a shell command line) and returns a Process; awaiting it gives the exit
  code. It throws if the program cannot start. A terminal is required.
- Files: `createFile` / `openFile` with `"binary"` or `"utf-8"`, then
  `write`, `read`, `getSize`, `close`. Copy a project in file by file.
- `createCustomTerminal` drives another terminal emulator, or captures output
  without one.
- **Persistence:** without `storageKey` the pod is ephemeral (a fresh disk each
  boot). With `storageKey`, the same key resumes the same disk, stored in the
  browser; one tab at a time per key.
- Starter project: `npm create browserpod-quickstart`.

## 3. Credit BrowserPod (licence condition)

BrowserPod is free for any use, including commercial products and production,
under the **BrowserPod Community Licence**, which requires a credit:

- Add a visible **"Powered by BrowserPod"** credit linking to
  https://browserpod.io on the page that hosts the pod (a tool without a UI
  credits it in its README).
- Leave the notice the runtime prints in the browser console.

Do this as part of the integration, and tell the user. Licence:
https://browserpod.io/license.

## 4. Networking: fetch network or Edge

| Need                                                                                     | Use                                                                                                       |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| No network, or HTTP/HTTPS to servers that allow CORS, `npm install`/`pnpm install`       | **Fetch network** (default, no key): `createFetchNetwork({ fetch: BrowserPod.opinionatedFetch })` (below) |
| Servers without CORS headers                                                             | A fetch network with a custom `fetch` that goes through the user's own proxy (ask them), or Edge          |
| A public URL for a server running in the pod (a dev server preview, a webhook)           | **Edge** (Portals)                                                                                        |
| TCP (databases, SSH remotes), or outbound access without CORS limits, under an allowlist | **Edge**                                                                                                  |

### Fetch network (no key)

Requests from the pod (HTTP, HTTPS and WebSocket) go out through the page's
own `fetch()`, without cookies, so the browser's CORS rules apply: a server
must send CORS headers allowing the page's origin, or the browser blocks the
request.

- `BrowserPod.boot({})` (no `apiKey`, no `network`) boots on a fetch network
  with the **plain** browser `fetch`. That is enough for servers that allow
  CORS, but **`npm install` won't work**: it needs the workarounds in
  `opinionatedFetch`. For any app that installs packages, pass it explicitly,
  as in §2:

  ```js
  const network = await BrowserPod.createFetchNetwork({
    fetch: BrowserPod.opinionatedFetch,
  });
  const pod = await BrowserPod.boot({ network });
  ```

- `opinionatedFetch(url, init)` rewrites requests for supported developer
  services so they work without a proxy (today: the npm registry, so
  `npm install` works); every other request is passed unchanged to
  `init.defaultFetch`. More services will be added; the Reference page lists
  them.
- **A custom `fetch`** receives `(url, init)`, where `init` is a `RequestInit`
  plus `defaultFetch` (the browser's `fetch`). Change the request, then hand it
  on. To route hosts without CORS through the user's own reverse proxy (ask
  them for it) and keep the npm workarounds, call `opinionatedFetch` last:

  ```js
  const network = await BrowserPod.createFetchNetwork({
    fetch(url, init) {
      if (new URL(url).hostname === "api.example.com") {
        url = "https://proxy.example.com/" + url; // the user's proxy
      }
      return BrowserPod.opinionatedFetch(url, init);
    },
  });
  ```

- **WebSockets** work with no extra setup through the default handler,
  `opinionatedFetch`, or any custom `fetch` that forwards to
  `init.defaultFetch` (or `opinionatedFetch`). Only a handler that intercepts
  the upgrade itself must return a `101` response carrying the socket on a
  `webSocket` property (see the `createFetchNetwork` Reference).
- No Portals, and no raw TCP, on a fetch network: those need Edge.

### BrowserPod Edge (hosted)

- **BrowserPod Edge** is the optional, hosted network service:
  `BrowserPod.createEdgeNetwork({ apiKey })`, then `boot({ network })`
  (`boot({ apiKey })` is the shorthand). It needs an account and an API key
  from https://console.browserpod.io. Outbound traffic is limited to the
  key's allowlist.
- **Portals** (Edge only): when a process in the pod listens on a port,
  `pod.onPortal(({ url, port }) => …)` gives a public URL for it. Anyone with
  the URL can reach that server: tell the user.
- **API keys are visible to visitors** of a public page. Protect a key by
  restricting its allowed origins in the console. Read it from configuration
  (e.g. `import.meta.env`), not hard-coded.
- **Privacy:** with Edge, traffic passes through BrowserPod's infrastructure.
  On either network, the browser loads the runtime and disk images from
  BrowserPod's CDN. If the user's app has a privacy policy, suggest it mention
  this third-party load.

## 5. Projects with native dependencies

Many npm packages ship platform binaries. Swap them for Wasm builds with
`overrides` (npm/pnpm) or `resolutions`:

- `esbuild` → `esbuild-wasm`; `rollup` → `@rollup/wasm-node`;
  `@parcel/watcher` → `@parcel/watcher-wasm`.
- Next.js: `@next/swc-wasm-nodejs` and webpack (Turbopack is not supported).
- Check the docs guide "Working around native npm dependencies" for the
  current list. If a dependency has no Wasm build, say so; don't work around it
  by other means.

## 6. Licensing and plans: tell the user, never decide

- **The library:** free for any use under the BrowserPod Community Licence,
  with the credit in §3. No account, no key, no metering.
- **Load the runtime from BrowserPod's CDN only.** Copying its JavaScript or
  Wasm files into a build, a repo or a package (vendoring), mirroring them, or
  self-hosting them needs the **BrowserPod Enterprise Licence**. So does
  redistribution (OEM).
- **BrowserPod Edge** plans (console): **Free** (monthly connected-hours
  allowance, default allowlist), **Pro** (more hours, custom allowlists, and
  the BrowserPod Community Licence without attribution: no visible credit
  needed while subscribed), **Enterprise** (the BrowserPod Enterprise Licence:
  self-hosted runtime and network, negotiated individually).
- Names, allowances and prices: https://browserpod.io/pricing. State in one
  line which plan you think applies and why, and let the user decide.

## 7. When something fails

1. Open DevTools before the page loads and read the console.
2. Note the browser and version, the BrowserPod version (from
   `package.json`), the network (fetch or Edge) and the exact error.
3. Check this table, then the docs page "Debugging BrowserPod":

| Symptom                                                     | Likely cause                                                                                                                 |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Pod never boots; `SharedArrayBuffer` missing                | The page isn't cross-origin isolated: COOP/COEP headers missing (§2)                                                         |
| `&&`, pipes or shell builtins fail in `pod.run`             | `run` starts one program (like `execve`), not a shell. Run commands one by one, or write a script and run it                 |
| `The 'terminal' argument is required`, or output disappears | No terminal, or its element was removed. Keep the element mounted (hide it with CSS if needed)                               |
| `Unsupported 'mode' argument`                               | `createFile` / `openFile` take only `"binary"` or `"utf-8"`                                                                  |
| Images, fonts or scripts from other sites stop loading      | `require-corp` blocks resources without CORP/CORS headers                                                                    |
| `run()` asks for Chrome / `Atomics.waitAsync` missing       | Unsupported browser: use a Chromium-based browser                                                                            |
| `npm install` fails on a fetch network                      | Not using `opinionatedFetch`, or a registry or host without CORS                                                             |
| A request fails with a CORS error                           | Fetch network: the server sends no CORS headers. Use the user's proxy or Edge                                                |
| No Portal URL                                               | Fetch network (Portals need Edge), or the server listens on no port                                                          |
| "Cannot create an edge network without an API key"          | `createEdgeNetwork` without `apiKey`                                                                                         |
| Edge boot fails, or requests to a domain are blocked        | Key's allowed origins don't include the page, the domain isn't on the key's allowlist, or the account's allowance is used up |
| Error that the disk is open elsewhere                       | Another tab uses the same `storageKey`                                                                                       |
| `Cannot find module …node` / exec format error              | A native binary: swap for a Wasm build (§5)                                                                                  |
| Out of memory                                               | Pod memory ceiling (~1 GB): a smaller project or build                                                                       |

Issues: https://github.com/leaningtech/browserpod-meta/issues · community:
Discord (linked from the docs).
