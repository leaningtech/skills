---
name: cheerpj
description: Run Java in the browser with CheerpJ. Converts java command lines, JARs, applets, Java Web Start (JNLP) apps, Oracle Forms and Oracle E-Business Suite clients, and Java libraries called from JavaScript, with no local JRE, plug-in or IE mode. Use when a user wants Java to run in a modern browser, asks whether or how CheerpJ fits their app, hits a CheerpJ error, or asks which CheerpJ licence applies.
---

# CheerpJ

CheerpJ is a Java Virtual Machine built on WebAssembly. It runs unmodified
bytecode (JARs) client-side in the browser. Nothing is recompiled, nothing runs
on a server, and nothing is installed on the user's machine.

## Ground rules

- **The docs hold the facts:** https://cheerpj.com/docs/. Supported Java
  versions, the loader script tag, function names, init options, browser support
  and limits all change between releases. Read Getting started and the Reference
  before writing code, copy the loader tag from Getting started, and use only
  documented options. Invented ones (for example a heap size or a container id)
  do not exist. If this skill and the docs disagree, the docs win.
- **Inspect before changing anything.** Say which path you picked and why.
- **Never propose a rewrite or port** as a precondition. CheerpJ runs the
  existing bytecode.
- **Stop and ask the user before you:**
  - install an extension, CLI or package;
  - change server or production configuration, or Oracle Forms/EBS files;
  - self-host or copy the runtime;
  - work around something the docs list as unsupported (report it instead);
  - choose or buy a licence.

## 1. Find how Java is launched today, then convert it

Look for the launch in start scripts (`.bat`, `.sh`, `.cmd`), shortcuts, launcher
config, `META-INF/MANIFEST.MF`, `.jnlp` files or `<applet>` tags. Large apps
usually launch with many JARs on a classpath and an explicit main class, not with
`-jar`. Map each part to its CheerpJ equivalent; exact names are in the
Reference.

| Today                                                                                          | In CheerpJ                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `java -jar app.jar args`                                                                       | Run-JAR call with the JAR's virtual path and the same args                                                                                                                           |
| `java -cp a.jar;lib/* com.x.Main args`                                                         | Run-main call: class name, classpath of virtual paths joined with `:` (expand every wildcard into explicit JARs), args                                                               |
| `-Dkey=value`                                                                                  | System-properties init option                                                                                                                                                        |
| `-Xmx`, `-Xms`, `-XX:…`, `-server`                                                             | Usually no equivalent. Drop them unless the Reference lists one                                                                                                                      |
| `-javaagent`, `-Djava.library.path`, `System.loadLibrary`                                      | Native or agent code. See §2                                                                                                                                                         |
| Target Java version (`JAVA_HOME`, class-file major 52 = 8, 55 = 11, 61 = 17, 65 = 21, 69 = 25) | Version init option. It must be one the docs Overview supports                                                                                                                       |
| Working directory and relative paths                                                           | Resolve in writable browser storage, not next to the JAR (Filesystem guide)                                                                                                          |
| `.jnlp`                                                                                        | `<jar href>` (relative to `codebase`) → classpath · `main-class` + `<argument>` → run-main · `<property>` → system properties · `applet-desc` → applet tag · `<nativelib>` → blocker |
| `<applet code=… archive=…>`                                                                    | §3                                                                                                                                                                                   |
| JavaScript needs to call Java                                                                  | §4                                                                                                                                                                                   |
| Oracle Forms (`frmservlet`, `formsweb.cfg`, `frmall.jar`) or EBS                               | §5                                                                                                                                                                                   |

## 2. What the browser changes

- **Files.** Java sees a virtual filesystem, never URLs or the local disk.
  - Server files appear under a read-only mount.
  - Writable storage is persistent in the browser.
  - Uploads and downloads have dedicated folders, and JS can inject files.
  - A wrong path is the most common first error. Read the Filesystem guide before
    writing any path.
- **Serving.**
  - Serve over HTTP(S), never `file://`, from a server with Range support.
    Python's `http.server` has none; the server-setup guide names some that work.
  - Send caching headers, or JARs download on every visit.
  - 404s for runtime files at startup are normal.
  - Stuck at the "runtime ready" message? Check for `file://`, a non-virtual path,
    or a JAR returning 404.
- **Network.** Same-origin HTTP works. Cross-origin needs CORS on the target.
  Raw sockets need the setup in the Networking guide; ask the user first.
- **OS touch points.**
  - External commands run nothing; they can be intercepted in JS (guide).
  - Opening a file with a desktop app cannot work. Hand the user a blob instead
    (Reference).
  - The PC's host name and IP are invisible. A value can be supplied through an
    init option.
  - The clipboard needs HTTPS.
  - Browser shortcuts win unless overridden with the keyboard init option. Some
    can never be overridden.
- **Blockers.** Pre-built native binaries (`.dll`, `.so`, `.dylib`, also inside
  JARs), OLE/COM automation, and UI toolkits the compatibility page lists as
  unsupported. Report them with file names. Native code with C/C++ source can
  sometimes be compiled to WebAssembly; that is a job for Leaning Technologies.
- **Performance.** First load downloads the runtime and JARs, so measure the
  second launch. Open DevTools slow the runtime a lot. The startup-time guide
  covers preloading.

## 3. Applets and Java Web Start

- **Test first with an extension:** CheerpJ Applet Runner (applet pages) or
  CheerpJ JNLP Runner (`.jnlp`). Each handles only its own launch type, and some
  sites switch per user. Check their docs for supported browsers.
- **Test cleanly:** no local Java, other extensions disabled, and the site not in
  Edge IE mode. CheerpJ cannot run inside IE mode.
- **Deploy:**
  - Extensions managed by IT (group policy, offline licence tokens). They can only
    be configured through their settings page.
  - CheerpJ in the page (Getting started: applet / JNLP). Use the `cheerpj-` tag
    variants if a native plug-in may still be installed.
  - A reverse proxy that injects CheerpJ. Involve Leaning Technologies.
- **Plug-in-era behaviour that is not a CheerpJ defect:**
  - **Page script calls the applet before it is ready:** wait for it, or queue the
    calls.
  - **IE-only DOM** (`document.frames["x"]`, `.applets[].object`): shim it.
  - **`CookieHandler.getDefault()` returns null:** Java SE installs none; the old
    plug-in did. Register one that reads the browser cookies.
  - **`cache_archive` attributes:** they do nothing.
  - **Mobile user agents:** some sites serve a non-applet page to them.
  - **Web Start services:** only some are implemented.
  - **Native DLLs:** a blocker.
- **Licence validation fails behind corporate web filters:** use offline tokens.

## 4. Java and JavaScript (library mode)

Copy the syntax from Getting started: library, the Library mode guide and the
native methods/libraries guides. Rules:

- Await every step across the boundary: class lookup, construction, calls.
- Initialise once per page and create the library object once.
- Objects belong to the context that made them. Inside a native method, use the
  library object the native receives. Build Java objects (for example events) in
  Java, not JS.
- Native method names must match the Java declaration exactly (mangling rule in
  the docs).
- Pass bulk data as typed arrays or files, not many small calls.
- Java runs on one thread in time slices. Use a Web Worker for heavy work if the
  docs allow it.

## 5. Oracle Forms and E-Business Suite

The Forms client (`frmall.jar` plus app JARs) is a Java program talking HTTP to the
Forms server. CheerpJ runs it unchanged; the server, `.fmx` modules and database
stay as they are.

### Classify first, and say which it is

Decide who controls the Forms server, who the users are, and whether the goal is
to keep the app running or to build and ship with it. Then report it in these
words:

> **Running an existing Forms application.** You want <app> to keep working for
> <your staff> without Java on their PCs or IE mode. Nothing in the application
> changes. We test with the CheerpJ browser extension, then either roll it out to
> the fleet or add CheerpJ to the Forms launch page on your server.

> **Integrating CheerpJ into a Forms-based application.** You own <app> and its
> Forms server and want CheerpJ to be part of how it is delivered: <a web edition /
> a product your customers use / the first step of a gradual modernisation>. We
> add CheerpJ to the Forms launch page and your web front end, keep the Forms
> modules as they are, and build the web pieces around them.

If it is unclear, ask: "Do you run this application for your own people, or do you
develop it for others?" Running a vendor's Forms product, including EBS, for your
own staff is **running**.

### Check the Oracle side first (most failed first tests)

1. **`connectMode=socket`.** It fails with
   `FRM-92060 ... Bad machine specification`. Fix: `connectMode=http`/`https`
   with `serverURL` pointing at the listener servlet (usually `/forms/lservlet`).
   A listener 404 is a Forms-side mapping problem.
2. **gzip on `/forms/frmservlet` or `/forms/lservlet`.** The client fails even with
   HTTP 200, and the native plug-in never asked for gzip. Exclude `/forms/*` from
   compression at the proxy or load balancer.
3. **Edge IE mode.** Remove the Forms or EBS host from the Enterprise Mode Site
   List for test users.
4. **Launch mode.** Applet or Web Start decides which extension to use. The server
   route needs applet launch.
5. **EBS only.** Change Forms settings through AutoConfig with the DBA. Patching
   (AD/TXK) overwrites hand edits.

### Running

- Test with the extension against this checklist:
  - login and SSO, menus, a heavy form;
  - LOVs and drop-downs (mouse and keyboard), shortcuts, clipboard;
  - printing and reports, file upload/download, WebUtil;
  - multiple windows, a second session, closing the tab and relaunching, timeout;
  - EBS: View Output/Log (allow pop-ups).
- Judge layout with a licence applied (the evaluation notice shifts it), and speed
  on a second launch with DevTools closed.
- Roll out by group policy with offline tokens, or use the server route for other
  browsers.

### Server route (both cases)

1. Add a new `formsweb.cfg` section (for example `[cheerpj]`) with HTTP connect mode
   and applet launch. Leave the live one alone.
2. Find the base template that section uses: `base.htm`, `basejpi.htm`,
   `basejpi_jnlp.htm`, `webutilbase.htm` or `webutiljpi.htm`.
3. Add CheerpJ to a copy of it as Getting started: applet shows, with only
   documented options (licence key, keyboard overrides, host name).
4. Confirm the loader is in the **delivered** HTML. A different template being
   picked is the usual reason nothing changed.
5. Serve the runtime per the server-setup guide (licensed copy for production),
   with `frmall.jar` and app JARs Range-enabled and cached, over HTTPS.
6. Test with `?config=cheerpj`.
7. EBS: prefer a reverse proxy that injects the loader. Template edits only through
   AutoConfig, re-checked after each patch.
8. **Integrating** adds:
   - the launch page becomes part of your web front end;
   - redistribution terms if customers host it;
   - for gradual modernisation, Forms and new web screens share the session and
     the host page. The Forms JavaScript integration (`WEB.JAVASCRIPT_EVAL_EXPR`,
     `enableJavascriptEvent`) may bridge events. [Check: verify on CheerpJ]

### Forms specifics

- **WebUtil file transfer:**
  - Never pass a virtual path as the client file name. That gives `WUT-131` and
    `WEBUTIL_FILE_TRANSFER` errors. Use `test.txt` or `C:\Temp\test.txt`.
  - Uploads arrive in the upload folder, which is a source only. In the file
    dialog, pick `All Files (*.*)`.
  - Downloads must be saved where the Filesystem guide says they are delivered.
  - Test non-ASCII names.
- **`CLIENT_HOST`:** intercept it or move it server-side. OLE, COM, DDE and client
  DLLs are blockers.
- **PC name or IP in PL/SQL:** on the server route, use the host-name option. With
  the extension it needs Leaning Technologies.
- **Sessions:**
  - Tabs share one session cookie, so a second session fails. Use the "always
    launch a new Forms session" setting or WebLogic cookie settings.
  - Closing the tab without exiting leaves a cookie that breaks the next launch.
  - Switching user needing a browser restart is normal browser behaviour.
- **Signed JARs:** keep the originals. Re-signing has caused `SecurityException`;
  check with `jarsigner -verify`.
- **Truncated or blurry text:** update the extension or runtime. On high-DPI
  Windows, Chrome's "Override high DPI scaling behaviour" is a workaround.
- **Forms 14c:** uses a newer Java client. Say if the docs don't confirm it.
- **Certification:** never claim Oracle certification. Oracle certifies its own
  JRE.

## 6. Assessing an application ("will it work?", "what will it cost?")

Change nothing. Report:

- **Verdict:** runs / runs with work / blocked.
- **Findings:** item · evidence (file path) · verdict · docs page · fix. Judge
  against the current compatibility page, not memory.
- **Path:** the next step to take.
- **Licence:** the likely licence and the open questions.
- **Next step:** a free proof of concept, or a compatibility review
  (https://cheerpj.com/contact) for blockers, Oracle estates or large rollouts.

## 7. Licensing: tell the user, never decide

- Technical evaluation is free for everyone.
- In production, **who uses the application** decides it: the organisation's own
  people, or customers and the public using software built with CheerpJ. Each has
  its own licence family, with a free tier for non-business or non-commercial
  use.
- Whoever integrates CheerpJ holds the licence. A contractor's client needs its
  own.
- Serving the runtime from anywhere but the CheerpJ CDN, including copying it into
  a repo or package, is self-hosting and needs a licence that allows it.
- Names, tiers and prices: https://cheerpj.com/pricing. State in one line which
  licence you think applies and why.

## 8. When something fails

1. Follow the Debugging guide (or the extension's debug page): debug mode on,
   DevTools open **before** the page loads.
2. Save the console log and a HAR. An empty HAR means DevTools opened too late.
3. Note the browser, the runtime or extension version, the launch mode, the exact
   error text and the steps to reproduce.
4. Check this table before blaming CheerpJ:

| Symptom                                      | Likely cause                                            |
| -------------------------------------------- | ------------------------------------------------------- |
| Class or file not found                      | Path not virtual, or classpath incomplete or wildcarded |
| Unsupported class version                    | Wrong version init option                               |
| Range-header error                           | Server lacks Range support                              |
| POST hangs                                   | Chunked response with keep-alive; send `Content-Length` |
| Java, IE or nothing opens instead of CheerpJ | IE mode, local Java or another extension                |
| Extension stopped working after a change     | Launch switched between applet and Web Start            |
| Licence won't validate                       | Web filter; use offline tokens                          |
| "Socket error" popup but the app works       | Non-fatal notice                                        |
| Layout shifted                               | Evaluation notice; apply a licence                      |
| Text cut off                                 | Old build; update                                       |

Issues: https://github.com/leaningtech/cheerpj-meta/issues · community: Discord
(linked from the docs).
