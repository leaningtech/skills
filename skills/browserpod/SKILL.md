---
name: browserpod
description: Run Linux applications in a browser tab with BrowserPod. Adds an in-browser sandbox to a web app — Node.js 22 and Rust programs, with bash, git and npm, on a WebAssembly kernel — that boots with no API key or account. Covers booting a pod, writing files, running programs in a terminal, networking with and without BrowserPod Edge, Portals, cross-origin isolation, native npm dependencies and the licence's attribution. Use when a user wants code, a dev server, a CLI or an AI agent to run client-side in the browser, asks whether or how BrowserPod fits, hits a BrowserPod error, or asks which BrowserPod licence or plan applies.
---

# BrowserPod

BrowserPod runs Linux applications in a browser tab. Programs such as `node`,
`bash`, `git` and Rust binaries are compiled from unmodified source to
WebAssembly and run as processes on a Linux-like kernel, client-side. A whole
toolchain works in a pod: `npm install && npm run dev`, `git clone`, CLIs.
Nothing runs on a server, and the user's files stay on their device.

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

| Runtime                                                         | Notes                                                                                                     |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Node.js 22 (the only version)                                   | The real Node binary; JavaScript runs on the browser's engine. npm, npx and pnpm work; Yarn is unreliable |
| Rust programs                                                   | Unmodified CLIs. A Rust toolchain can be installed in a pod (docs guide)                                  |
| bash, git, curl, ripgrep and common command-line tools          | In the base image                                                                                         |
| Python                                                          | **Preview only**: no native extension modules. Don't promise real Python apps                             |
| Go, Ruby, Rails, Java                                           | Not available in BrowserPod. (Java in the browser is CheerpJ, a separate product)                         |
| Prebuilt native binaries (`.node` addons, platform executables) | Don't run. Swap for Wasm builds (§5)                                                                      |

- **Browsers:** Chromium-based browsers are the reliable target. Firefox and
  Safari have limitations; heavy workloads may fail on mobile.
- **Memory:** about 1 GB per pod in practice. A very large repository or build
  can run out.

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
under the **BrowserPod Community Licence**, with attribution:

- Add a visible **"Powered by BrowserPod"** credit linking to
  https://browserpod.io on the page or screen that hosts the pod (a footer, an
  About page or the loading state).
- A tool or agent that uses pods without a UI credits BrowserPod in its README
  or docs.
- The runtime also prints a notice in the browser console. Leave it.

Do this as part of the integration, and tell the user. Licence:
https://browserpod.io/license.

## 4. Networking: fetch network or Edge

| Need                                                                                     | Use                                                                                                                               |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| No network, or HTTP/HTTPS to servers that allow CORS, `npm install`/`pnpm install`       | **Fetch network** (default, free, no key): `BrowserPod.boot({})`, or `createFetchNetwork({ fetch: BrowserPod.opinionatedFetch })` |
| Servers without CORS headers                                                             | A fetch network with a custom `fetch` that goes through the user's own proxy (ask them), or Edge                                  |
| A public URL for a server running in the pod (a dev server preview, a webhook)           | **Edge** (Portals)                                                                                                                |
| TCP (databases, SSH remotes), or outbound access without CORS limits, under an allowlist | **Edge**                                                                                                                          |

- **Fetch network:** requests go out through the page's own `fetch()`, without
  cookies, so the browser's CORS rules apply. Outgoing WebSockets work.
  `opinionatedFetch` adds workarounds for known developer services such as the
  npm registry. A custom `fetch` passed to `createFetchNetwork` can route
  requests through the integrator's own reverse proxy.
- **BrowserPod Edge** is the optional, paid network service:
  `BrowserPod.createEdgeNetwork({ apiKey })`, then `boot({ network })`
  (`boot({ apiKey })` is the shorthand). It needs an account and an API key
  from https://console.browserpod.io. Outbound traffic is limited to an
  allowlist (a default list on Free; custom lists on Pro); there is no
  unrestricted access. Edge is priced per connected pod-hour, with a free
  monthly allowance.
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

- **The library:** free for any use under the Community Licence, with
  attribution (§3). No account, no key, no metering.
- **Load the runtime from BrowserPod's CDN only.** Copying its JavaScript or
  Wasm files into a build, a repo or a package (vendoring), mirroring them, or
  self-hosting them needs an **Enterprise** licence. So do redistribution
  (OEM) and removing the attribution outside the Pro plan.
- **BrowserPod Edge** plans (console): **Free** (monthly connected-hours
  allowance, default allowlist), **Pro** (more hours, custom allowlists, and no
  attribution requirement), **Enterprise** (self-hosted runtime and network,
  contracts).
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
