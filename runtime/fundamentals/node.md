---
title: "Node.js support"
oldUrl:
- /runtime/reference/node/
- /runtime/manual/npm_nodejs/std_node/
- /runtime/manual/node/
- /runtime/manual/npm_nodejs/cdns/
- /runtime/manual/using_deno_with_other_technologies/node/cdns/
- /runtime/manual/node/node_specifiers
- /runtime/manual/node/package_json
- /runtime/manual/npm_nodejs/compatibility_mode/
- /runtime/manual/node/migrate/
- /runtime/manual/node/compatibility/
- /runtime/manual/references/cheatsheet/
- /runtime/manual/node/cheatsheet/
- /runtime/manual/node/faqs
---

Modern Node.js projects will run in Deno with little to no reworking required.
However, there are some key differences between the two runtimes that you can
take advantage of to make your code simpler and smaller when migrating your
Node.js projects to Deno.

## Node built-in modules

Deno provides a compatibility layer that allows the use of Node.js built-in APIs
within Deno programs. However, in order to use them, you will need to add the
`node:` specifier to any import statements that use them:

```js title=main.mjs
import * as os from "node:os";
console.log(os.cpus());
```

And run it with `deno run main.mjs` - you will notice you get the same output as
running the program in Node.js.

Updating any imports in your application to use `node:` specifiers should enable
any code using Node built-ins to function as it did in Node.js.

To make updating existing code easier, Deno will provide helpful hints for
imports that don't use `node:` prefix:

```js title="main.mjs"
import * as os from "os";
console.log(os.cpus());
```

```sh
$ deno run main.mjs
error: Relative import path "os" not prefixed with / or ./ or ../
  hint: If you want to use a built-in Node module, add a "node:" prefix (ex. "node:os").
    at file:///main.mjs:1:21
```

Same hints and additional quick-fixes are provided by the Deno LSP in your
editor.

## Node.js global objects

In Node.js, there are a number of
[global objects](https://nodejs.org/api/globals.html) available in the scope of
all programs that are specific to Node.js, eg. `process` object.

Here are a few globals that you might enounter in the wild and how to use them
in Deno:

- `process` - Deno provides the `process` global, which is by far the most
  popular global used in popular npm packages. It is available to all code.
  However, Deno will guide you towards importing it explicitly from
  `node:process` module by providing lint warnings and quick-fixes:

```js title="process.js"
console.log(process.versions.deno);
```

```shell
$ deno run process.js
2.0.0
$ deno lint process.js
error[no-process-globals]: NodeJS process global is discouraged in Deno
 --> /process.js:1:13
  |
1 | console.log(process.versions.deno);
  |             ^^^^^^^
  = hint: Add `import process from "node:process";`

  docs: https://lint.deno.land/rules/no-process-globals


Found 1 problem (1 fixable via --fix)
Checked 1 file
```

- `require()` function is only available for CommonJS modules, to use it in Deno
  directly you can either run a file with `.cjs` extension, or create a
  `require` instance explicitly:

```js title="require.cjs"
const lib = require("./lib");
```

```js title="require.js"
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const lib = require("./lib");
```

- `Buffer` - to use `Buffer` API it needs to be explicitly imported from the
  `node:buffer` module:

```js title="buffer.js"
import { Buffer } from "node:buffer";

const buf = new Buffer(5, "0");
```

Prefer using
[`Uint8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array)
or other
[`TypedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray)
subclasses instead.

- `__filename` - use `import.meta.filename` instead.

- `__dirname` - use `import.meta.dirname` instead.

## Runtime permissions in Deno

Consider the following simple [Express](https://expressjs.com/) server:

```js title="server.js"
import express from "npm:express@4";

const app = express();

app.get("/", function (_req, res) {
  res.send("hello");
});

app.listen(3000, () => {
  console.log("Express listening on :3000");
});
```

If you run the above with `deno run server.js`, you will be prompted for
permissions required to execute the code and its dependencies. For example:

```sh
$ deno run server.js
┌ ⚠️  Deno requests net access to "0.0.0.0:8000".
├ Requested by `Deno.listen()` API.
├ Run again with --allow-net to bypass this prompt.
└ Allow? [y/n/A] (y = yes, allow; n = no, deny; A = allow all net permissions) >
```

Deno features [runtime security by default](/runtime/fundamentals/security/),
meaning that you as the developer must opt in to giving your code access to the
filesystem, network, system environment, and more. Doing this prevents supply
chain attacks and other potential vulnerabilities in your code. By comparison,
Node.js has no concept of runtime security, with all code executed with the same
level of permission as the user running the code.

To run your code as you would in Node.js, you can pass the `-A` flag to enable
all permissions.

```sh
deno run -A server.js
```

For more granular control, you can enable access to specific features by opting
in to
[individual permissions](/runtime/fundamentals/security/#permissions-list).

## Running scripts from package.json

Deno supports running npm scripts natively with the
[`deno task`](../tools/task_runner.md) subcommand. Consider the following
Node.js project with a script called `start` inside its `package.json`:

```json title="package.json"
{
  "name": "my-project",
  "scripts": {
    "start": "eslint"
  }
}
```

You can execute this script with Deno by running:

```sh
deno task start
```

## Migrating from Node.js to Deno

Running your Node.js project with Deno is a straightforward process. In most
cases you can expect little to no changes to be required, if your project is
written using ES modules.

Main points to be aware of, include:

1. Importing Node.js built-in modules requires the `node:` specifier:

```js
// ❌
import * as fs from "fs";
import * as http from "http";

// ✅
import * as fs from "node:fs";
import * as http from "node:http";
```

:::tip

It is recommended to change these import specifiers in your existing project
anyway. This is a recommended way to import them in Node.js too.

:::

2. Some [globals available in Node.js](#nodejs-global-objects) need to be
   explicitly imported, eg. `Buffer`:

```js
import { Buffer } from "node:buffer";
```

3. `require()` is only available in files with `.cjs` extension, in other files
   an instance of `require()`
   [needs to be created manually](#nodejs-global-objects). npm dependencies can
   use `require()` regardless of file extension.

## Optional improvements with Deno's built-in tools

One of Deno's core strengths is a unified toolchain that comes with support for
TypeScript out of the box, and tools like a linter, formatter and a test runner.
Switching to Deno allows you to simplify your toolchain and reduces the number
of moving components in your project. Deno also has a more secure runtime, with
[runtime permissions](../basics/permissions.md) that allow you to control what
your code can access.

#### deno.json

Deno has its own config file, `deno.json` or `deno.jsonc`, which can be used to
configure your project.

You can use it to [define dependencies](/runtime/fundamentals/configuration/)
using the `imports` option - you can migrate your dependencies one-by-one from
`package.json`, or elect to not define them in the config file at all and use
`npm:` specifiers inline in your code.

In addition to specifying depenendencies you can use
[`deno.json` to define](/runtime/fundamentals/configuration/) tasks, lint and
format options, path mappings, and other runtime configurations.

#### Linting

Deno ships with a built-in linter that is written with performance in mind. It's
similar to ESlint, though with a limited number of rules. If you don't rely on
ESLint plugins, you can drop `eslint` dependency from `devDependencies` section
of `package.json` and use `deno lint` instead.

Deno can lint large projects in just a few milliseconds. You can try it out on
your project by running:

```sh
deno lint
```

This will lint all files in your project. When the linter detects a problem, it
will show the line in your editor and in the terminal output. An example of what
that might look like:

```sh
error[no-constant-condition]: Use of a constant expressions as conditions is not allowed.
 --> /my-project/bar.ts:1:5
  | 
1 | if (true) {
  |     ^^^^
  = hint: Remove the constant expression

  docs: https://lint.deno.land/rules/no-constant-condition


Found 1 problem
Checked 4 files
```

Many linting issues can be fixed automatically by passing the `--fix` flag:

```sh
deno lint --fix
```

A full list of all supported linting rules can be found on
[https://lint.deno.land/](https://lint.deno.land/). To learn more about how to
configure the linter, check out the [`deno lint` subcommand](../tools/linter/).

#### Formatting

Deno ships with a [built-in formatter](../tools/formatter/) that can optionally
format your code according to the Deno style guide. Instead of adding `prettier`
to your `devDependencies` you can instead use Deno's built-in zero-config code
formatter `deno fmt`.

You can run the formatter on your project by running:

```sh
deno fmt
```

If using `deno fmt` in CI, you can pass the `--check` argument to make the
formatter exit with an error when it detects improperly formatted code.

```sh
deno fmt --check
```

The formatting rules can be configured in your `deno.json` file. To learn more
about how to configure the formatter, check out the
[`deno fmt` subcommand](../tools/formatter/).

#### Testing

Deno encourages writing tests for your code, and provides a built-in test runner
to make it easy to write and run tests. The test runner is tightly integrated
into Deno, so that you don't have to do any additional configuration to make
TypeScript or other features work.

```ts title="my_test.ts"
Deno.test("my test", () => {
  // Your test code here
});
```

```sh
deno test
```

When passing the `--watch` flag, the test runner will automatically reload when
any of the imported modules change.

To learn more about the test runner and how to configure it, check out the
[`deno test` subcommand](../tools/test/) documentation.

## Node to Deno Cheatsheet

| Node.js                                | Deno                          |
| -------------------------------------- | ----------------------------- |
| `node file.js`                         | `deno file.js`                |
| `ts-node file.ts`                      | `deno file.ts`                |
| `nodemon`                              | `deno run --watch`            |
| `node -e`                              | `deno eval`                   |
| `npm i` / `npm install`                | `deno install`                |
| `npm install -g`                       | `deno install -g`             |
| `npm run`                              | `deno task`                   |
| `eslint`                               | `deno lint`                   |
| `prettier`                             | `deno fmt`                    |
| `package.json`                         | `deno.json` or `package.json` |
| `tsc`                                  | `deno check` ¹                |
| `typedoc`                              | `deno doc`                    |
| `jest` / `ava` / `mocha` / `tap` / etc | `deno test`                   |
| `nexe` / `pkg`                         | `deno compile`                |
| `npm explain`                          | `deno info`                   |
| `nvm` / `n` / `fnm`                    | `deno upgrade`                |
| `tsserver`                             | `deno lsp`                    |
| `nyc` / `c8` / `istanbul`              | `deno coverage`               |
| benchmarks                             | `deno bench`                  |

¹ Type checking happens automatically, TypeScript compiler is built into the
`deno` binary.

## Node Compatibility

Deno provides polyfills for a number of built-in Node.js modules and globals.
For a full list of Node built-in modules, see the
[reference](https://docs.deno.com/api/node/).

Node compatibility is an ongoing project - help us identify gaps and let us know
which modules you need by
[opening an issue on GitHub](https://github.com/denoland/deno).

<div class="flex flex-row gap-3 flex-wrap items-center mb-2">
  <div>
    <svg class="status-icon status-good" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="size-6">
      <path stroke-linecap="round" stroke-linejoin="round" d="M9 12.75 11.25 15 15 9.75M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" />
    </svg>
     = Full support
  </div>
  <div>
    <svg class="status-icon status-info" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="size-6">
      <path stroke-linecap="round" stroke-linejoin="round" d="m11.25 11.25.041-.02a.75.75 0 0 1 1.063.852l-.708 2.836a.75.75 0 0 0 1.063.853l.041-.021M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Zm-9-3.75h.008v.008H12V8.25Z" />
    </svg>
    = Partial support</div>
  <div>
    <svg class="status-icon status-stubs" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="size-6">
      <path stroke-linecap="round" stroke-linejoin="round" d="m9.75 9.75 4.5 4.5m0-4.5-4.5 4.5M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" />
    </svg>
     = Stubs only
  </div>
</div>

<div class="module-info compat-status-good">

### [`node:assert`](https://docs.deno.com/api/node/assert/)

</div>

<div class="module-info compat-status-partial">

### [`node:async_hooks`](https://docs.deno.com/api/node/async_hooks/)

`AsyncLocalStorage` is supported. `AsyncResource`, `executionAsyncId`, and
`createHook` are non-functional stubs.

</div>

<div class="module-info compat-status-good">

### [`node:buffer`](https://docs.deno.com/api/node/buffer/)

</div>

<div class="module-info compat-status-good">

### [`node:child_process`](https://docs.deno.com/api/node/child_process/)

</div>

<div class="module-info compat-status-stubs">

### [`node:cluster`](https://docs.deno.com/api/node/cluster/)

All exports are non-functional stubs.

</div>

<div class="module-info compat-status-good">

### [`node:console`](https://docs.deno.com/api/node/console/)

</div>

<div class="module-info compat-status-good">

### [`node:crypto`](https://docs.deno.com/api/node/crypto/)

Missing `Certificate` class, `crypto.Cipheriv.prototype.setAutoPadding`,
`crypto.Decipheriv.prototype.setAutoPadding`, `crypto.publicDecrypt`,
`crypto.ECDH.prototype.convertKey`, `x448` option for `generateKeyPair`,
`crypto.KeyObject`, `safe`, `add` and `rem` options for `generatePrime`,
`crypto.Sign.prototype.sign` and `crypto.Verify.prototype.verify` with non
`BinaryLike` input, `crypto.secureHeapUsed`, `crypto.setEngine`, legacy methods
of `crypto.X509Certificate`.

</div>

<div class="module-info compat-status-partial">

### [`node:dgram`](https://docs.deno.com/api/node/dgram/)

Some `dgram.Socket` instance methods are non-functional stubs:

- `addMembership`
- `addSourceSpecificMembership`
- `dropMembership`
- `dropSourceSpecificMembership`
- `setBroadcast`
- `setMulticastInterface`
- `setMulticastLoopback`
- `setMulticastTtl`
- `setTtl`

</div>

<div class="module-info compat-status-good">

### [`node:diagnostics_channel`](https://docs.deno.com/api/node/diagnostics_channel/)

</div>

<div class="module-info compat-status-partial">

### [`node:dns`](https://docs.deno.com/api/node/dns/)

Missing `dns.resolve*` with `ttl` option.

</div>

<div class="module-info compat-status-stubs">

### [`node:domain`](https://docs.deno.com/api/node/domain/)

All exports are non-functional stubs. This is a deprecated Node module.

</div>

<div class="module-info compat-status-good">

### [`node:events`](https://docs.deno.com/api/node/events/)

</div>

<div class="module-info compat-status-good">

### [`node:fs`](https://docs.deno.com/api/node/fs/)

`node:fs` Missing `utf16le`, `latin1` and `ucs2` encoding for `fs.writeFile` and
`fs.writeFileSync`.

`node:fs/promises` Missing `lchmod`.

</div>

<div class="module-info compat-status-good">

### [`node:http`](https://docs.deno.com/api/node/http/)

`createConnection` option is currently not supported.

</div>

<div class="module-info compat-status-partial">

### [`node:http2`](https://docs.deno.com/api/node/http2/)

Partially supported, major work in progress to enable `grpc-js`.

</div>

<div class="module-info compat-status-partial">

### [`node:https`](https://docs.deno.com/api/node/https/)

Missing `https.Server.opts.cert` and `https.Server.opts.key` array type.

</div>

<div class="module-info compat-status-partial">

### [`node:inspector`](https://docs.deno.com/api/node/inspector/)

`console` is supported. Other APIs are stubs and will throw an error.

</div>

<div class="module-info compat-status-good">

### [`node:module`](https://docs.deno.com/api/node/module/)

The `register()` function is not supported.

</div>

<div class="module-info compat-status-partial">

### [`node:net`](https://docs.deno.com/api/node/net/)

Missing `net.Socket.prototype.constructor` with `fd` option.

</div>

<div class="module-info compat-status-good">

### [`node:os`](https://docs.deno.com/api/node/os/)

</div>

<div class="module-info compat-status-good">

### [`node:path`](https://docs.deno.com/api/node/path/)

</div>

<div class="module-info compat-status-partial">

### [`node:perf_hooks`](https://docs.deno.com/api/node/perf_hooks/)

Missing `perf_hooks.eventLoopUtilization`, `perf_hooks.timerify`,
`perf_hooks.monitorEventLoopDelay`.

</div>

<div class="module-info compat-status-good">

### [`node:punycode`](https://docs.deno.com/api/node/punycode/)

</div>

<div class="module-info compat-status-partial">

### [`node:process`](https://docs.deno.com/api/node/process/)

Missing `multipleResolves`, `worker` events.

</div>

<div class="module-info compat-status-good">

### [`node:querystring`](https://docs.deno.com/api/node/querystring/)

</div>

<div class="module-info compat-status-good">

### [`node:readline`](https://docs.deno.com/api/node/readline/)

</div>

<div class="module-info compat-status-partial">

### [`node:repl`](https://docs.deno.com/api/node/repl/)

`builtinModules` and `_builtinLibs` are supported. Missing
`REPLServer.prototype.constructor` and `start()`.

</div>

<div class="module-info compat-status-good">

### [`node:stream`](https://docs.deno.com/api/node/stream/)

</div>

<div class="module-info compat-status-good">

### [`node:string_decoder`](https://docs.deno.com/api/node/string_decoder/)

</div>

<div class="module-info compat-status-good">

### [`node:sys`](https://docs.deno.com/api/node/util/)

</div>

<div class="module-info compat-status-partial">

### [`node:test`](https://nodejs.org/api/test.html)

Currently only `test` API is supported.

</div>

<div class="module-info compat-status-good">

### [`node:timers`](https://docs.deno.com/api/node/timers/promises/)

</div>

<div class="module-info compat-status-partial">

### [`node:tls`](https://docs.deno.com/api/node/tls/)

Missing `createSecurePair`.

</div>

<div class="module-info compat-status-stubs">

### [`node:trace_events`](https://docs.deno.com/api/node/trace_events/)

All exports are non-functional stubs.

</div>

<div class="module-info compat-status-good">

### [`node:tty`](https://docs.deno.com/api/node/tty/)

</div>

<div class="module-info compat-status-partial">

### [`node:util`](https://docs.deno.com/api/node/util/)

Missing `aborted`, `transferableAbortSignal`, `transferableAbortController`,
`MIMEParams`, `MIMEType` and `getSystemErrorMap`.

</div>

<div class="module-info compat-status-good">

### [`node:url`](https://docs.deno.com/api/node/url/)

</div>

<div class="module-info compat-status-partial">

### [`node:v8`](https://docs.deno.com/api/node/v8/)

`cachedDataVersionTag` and `getHeapStatistics` are supported.
`setFlagsFromStrings` is a noop. Other APIs are not supported and will throw an
error.

</div>

<div class="module-info compat-status-partial">

### [`node:vm`](https://docs.deno.com/api/node/vm/)

Partial support.

</div>

<div class="module-info compat-status-stubs">

### [`node:wasi`](https://docs.deno.com/api/node/wasi/)

All exports are non-functional stubs.

</div>

<div class="module-info compat-status-partial">

### [`node:worker_threads`](https://docs.deno.com/api/node/worker_threads/)

Missing `parentPort.emit`, `parentPort.removeAllListeners`,
`markAsUntransferable`, `moveMessagePortToContext`, `receiveMessageOnPort`,
`Worker.prototype.getHeapSnapshot`.

</div>

<div class="module-info compat-status-good">

### [`node:zlib`](https://docs.deno.com/api/node/zlib/~/Zlib)

</div>

## Globals

This is the list of Node globals that Deno supports. These globals are only
available in the `npm` package scope. In your own code you can use them by
importing them from the relevant `node:` module.

| Global name                                                                                                      | Status                                       |
| ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| [`AbortController`](https://nodejs.org/api/globals.html#class-abortcontroller)                                   | ✅                                           |
| [`AbortSignal`](https://nodejs.org/api/globals.html#class-abortsignal)                                           | ✅                                           |
| [`Blob`](https://nodejs.org/api/globals.html#class-blob)                                                         | ✅                                           |
| [`Buffer`](https://nodejs.org/api/globals.html#class-buffer)                                                     | ✅                                           |
| [`ByteLengthQueuingStrategy`](https://nodejs.org/api/globals.html#class-bytelengthqueuingstrategy)               | ✅                                           |
| [`__dirname`](https://nodejs.org/api/globals.html#__dirname)                                                     | ⚠️ [Info](./migrate/#node.js-global-objects) |
| [`__filename`](https://nodejs.org/api/globals.html#__filename)                                                   | ⚠️ [Info](./migrate/#nodejs-global-objects)  |
| [`atob`](https://nodejs.org/api/globals.html#atobdata)                                                           | ✅                                           |
| [`BroadcastChannel`](https://nodejs.org/api/globals.html#broadcastchannel)                                       | ✅                                           |
| [`btoa`](https://nodejs.org/api/globals.html#btoadata)                                                           | ✅                                           |
| [`clearImmediate`](https://nodejs.org/api/globals.html#clearimmediateimmediateobject)                            | ✅                                           |
| [`clearInterval`](https://nodejs.org/api/globals.html#clearintervalintervalobject)                               | ✅                                           |
| [`clearTimeout`](https://nodejs.org/api/globals.html#cleartimeouttimeoutobject)                                  | ✅                                           |
| [`CompressionStream`](https://nodejs.org/api/globals.html#class-compressionstream)                               | ✅                                           |
| [`console`](https://nodejs.org/api/globals.html#console)                                                         | ✅                                           |
| [`CountQueuingStrategy`](https://nodejs.org/api/globals.html#class-countqueuingstrategy)                         | ✅                                           |
| [`Crypto`](https://nodejs.org/api/globals.html#crypto)                                                           | ✅                                           |
| [`CryptoKey`](https://nodejs.org/api/globals.html#cryptokey)                                                     | ✅                                           |
| [`CustomEvent`](https://nodejs.org/api/globals.html#customevent)                                                 | ✅                                           |
| [`CustomEvent`](https://nodejs.org/api/globals.html#customevent)                                                 | ✅                                           |
| [`DecompressionStream`](https://nodejs.org/api/globals.html#class-decompressionstream)                           | ✅                                           |
| [`Event`](https://nodejs.org/api/globals.html#event)                                                             | ✅                                           |
| [`EventTarget`](https://nodejs.org/api/globals.html#eventtarget)                                                 | ✅                                           |
| [`exports`](https://nodejs.org/api/globals.html#exports)                                                         | ✅                                           |
| [`fetch`](https://nodejs.org/api/globals.html#fetch)                                                             | ✅                                           |
| [`fetch`](https://nodejs.org/api/globals.html#fetch)                                                             | ✅                                           |
| [`File`](https://nodejs.org/api/globals.html#class-file)                                                         | ✅                                           |
| [`File`](https://nodejs.org/api/globals.html#class-file)                                                         | ✅                                           |
| [`FormData`](https://nodejs.org/api/globals.html#class-formdata)                                                 | ✅                                           |
| [`global`](https://nodejs.org/api/globals.html#global)                                                           | ✅                                           |
| [`Headers`](https://nodejs.org/api/globals.html#class-headers)                                                   | ✅                                           |
| [`MessageChannel`](https://nodejs.org/api/globals.html#messagechannel)                                           | ✅                                           |
| [`MessageEvent`](https://nodejs.org/api/globals.html#messageevent)                                               | ✅                                           |
| [`MessagePort`](https://nodejs.org/api/globals.html#messageport)                                                 | ✅                                           |
| [`module`](https://nodejs.org/api/globals.html#module)                                                           | ✅                                           |
| [`PerformanceEntry`](https://nodejs.org/api/globals.html#performanceentry)                                       | ✅                                           |
| [`PerformanceMark`](https://nodejs.org/api/globals.html#performancemark)                                         | ✅                                           |
| [`PerformanceMeasure`](https://nodejs.org/api/globals.html#performancemeasure)                                   | ✅                                           |
| [`PerformanceObserver`](https://nodejs.org/api/globals.html#performanceobserver)                                 | ✅                                           |
| [`PerformanceObserverEntryList`](https://nodejs.org/api/globals.html#performanceobserverentrylist)               | ❌                                           |
| [`PerformanceResourceTiming`](https://nodejs.org/api/globals.html#performanceresourcetiming)                     | ❌                                           |
| [`performance`](https://nodejs.org/api/globals.html#performance)                                                 | ✅                                           |
| [`process`](https://nodejs.org/api/globals.html#process)                                                         | ✅                                           |
| [`queueMicrotask`](https://nodejs.org/api/globals.html#queuemicrotaskcallback)                                   | ✅                                           |
| [`ReadableByteStreamController`](https://nodejs.org/api/globals.html#class-readablebytestreamcontroller)         | ✅                                           |
| [`ReadableStream`](https://nodejs.org/api/globals.html#class-readablestream)                                     | ✅                                           |
| [`ReadableStreamBYOBReader`](https://nodejs.org/api/globals.html#class-readablestreambyobreader)                 | ✅                                           |
| [`ReadableStreamBYOBRequest`](https://nodejs.org/api/globals.html#class-readablestreambyobrequest)               | ✅                                           |
| [`ReadableStreamDefaultController`](https://nodejs.org/api/globals.html#class-readablestreamdefaultcontroller)   | ✅                                           |
| [`ReadableStreamDefaultReader`](https://nodejs.org/api/globals.html#class-readablestreamdefaultreader)           | ✅                                           |
| [`require`](https://nodejs.org/api/globals.html#require)                                                         | ✅                                           |
| [`Response`](https://nodejs.org/api/globals.html#response)                                                       | ✅                                           |
| [`Request`](https://nodejs.org/api/globals.html#request)                                                         | ✅                                           |
| [`setImmediate`](https://nodejs.org/api/globals.html#setimmediatecallback-args)                                  | ✅                                           |
| [`setInterval`](https://nodejs.org/api/globals.html#setintervalcallback-delay-args)                              | ✅                                           |
| [`setTimeout`](https://nodejs.org/api/globals.html#settimeoutcallback-delay-args)                                | ✅                                           |
| [`structuredClone`](https://nodejs.org/api/globals.html#structuredclonevalue-options)                            | ✅                                           |
| [`structuredClone`](https://nodejs.org/api/globals.html#structuredclonevalue-options)                            | ✅                                           |
| [`SubtleCrypto`](https://nodejs.org/api/globals.html#subtlecrypto)                                               | ✅                                           |
| [`DOMException`](https://nodejs.org/api/globals.html#domexception)                                               | ✅                                           |
| [`TextDecoder`](https://nodejs.org/api/globals.html#textdecoder)                                                 | ✅                                           |
| [`TextDecoderStream`](https://nodejs.org/api/globals.html#class-textdecoderstream)                               | ✅                                           |
| [`TextEncoder`](https://nodejs.org/api/globals.html#textencoder)                                                 | ✅                                           |
| [`TextEncoderStream`](https://nodejs.org/api/globals.html#class-textencoderstream)                               | ✅                                           |
| [`TransformStream`](https://nodejs.org/api/globals.html#class-transformstream)                                   | ✅                                           |
| [`TransformStreamDefaultController`](https://nodejs.org/api/globals.html#class-transformstreamdefaultcontroller) | ✅                                           |
| [`URL`](https://nodejs.org/api/globals.html#url)                                                                 | ✅                                           |
| [`URLSearchParams`](https://nodejs.org/api/globals.html#urlsearchparams)                                         | ✅                                           |
| [`URLSearchParams`](https://nodejs.org/api/globals.html#urlsearchparams)                                         | ✅                                           |
| [`WebAssembly`](https://nodejs.org/api/globals.html#webassembly)                                                 | ✅                                           |
| [`WritableStream`](https://nodejs.org/api/globals.html#class-writablestream)                                     | ✅                                           |
| [`WritableStreamDefaultController`](https://nodejs.org/api/globals.html#class-writablestreamdefaultcontroller)   | ✅                                           |
| [`WritableStreamDefaultWriter`](https://nodejs.org/api/globals.html#class-writablestreamdefaultwriter)           | ✅                                           |
