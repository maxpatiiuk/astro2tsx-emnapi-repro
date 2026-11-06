# `astro check` hides an incompatible transitive dependency

Astro doesn't declare a required peer dep causing incorrect dep resolution. Then Astro CLI incorrectly reports broken dep resolution as a missing `@astro/check` package

## Reproduction

```sh
git clone https://github.com/maxpatiiuk/astro2tsx-emnapi-repro
cd astro2tsx-emnapi-repro
npm ci
npm run check
```

## Explanation

Astro offers to install `@astrojs/check` even though it is already installed:

```log
To continue, Astro requires the following dependency to be installed: @astrojs/check.

  Astro will run the following command:
  If you skip this step, you can always run it yourself later
╭────────────────────────────────────╮
│  npm i @astrojs/check typescript   │
╰────────────────────────────────────╯
■  Continue?
Yes
10:24:48 [ERROR] [check] The `@astrojs/check` and `typescript` packages are required for this command to work. Please manually install them into your project and try again.
```

The checker actually crashes during import. To see the hidden error:

```sh
npm run repro
```

```log
file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/runtime/dist/emnapi.js:798
        this.bridge.setLastError(this.address, 0, 0, 0);
                    ^

TypeError: this.bridge.setLastError is not a function
    at NodeEnv.clearLastError (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/runtime/dist/emnapi.js:798:21)
    at NodeEnv.callIntoModule (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/runtime/dist/emnapi.js:807:14)
    at Object.init (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:150:35)
    at emnapiInit (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:7613:24)
    at file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:7638:16
    at loadSyncCallback (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:7656:12)
    at loadNapiModuleImpl (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:7574:12)
    at instantiateNapiModuleSync (file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@emnapi/core/dist/emnapi-core.mjs:7674:12)
    at file:///Users/mak13180/e/astro2tsx-emnapi-repro/node_modules/@astrojs/astro2tsx/index.js:13:24
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
```

Its transitive compiler, `@astrojs/astro2tsx@0.1.3`, loads an incompatible pair:
`@emnapi/core@1.11.3` and `@emnapi/runtime@2.0.0-beta.2`. Astro catches any
checker import exception and mistakes it for a missing package, instead of
reporting the real error. Using runtime `1.11.3` makes the import succeed.

## Proposed fixes

The upstream fixes are:

1. In `@astrojs/astro2tsx`, move `@emnapi/runtime@1.11.3` from
   `devDependencies` into `dependencies`, matching its pinned
   `@emnapi/core@1.11.3`. This ensures consumers get the compatible runtime
   instead of relying on an independently resolved peer.
2. In Astro's CLI, distinguish package resolution failures from import failures.

## (optional fresh repro steps) How this happened in a real project

Our real monorepo did not declare either emnapi package directly.
The beta runtime entered its lockfile through Astro's own dependency tree:

```text
astro@7.4.6
  rolldown@1.3.0
    @rolldown/binding-wasm32-wasi@1.3.0 (optional)
      @emnapi/runtime@2.0.0-beta.2
```

The checker could reuse that beta because its compiler pins core `1.11.3`,
but lists the matching runtime only in devDependencies. Its WASM runtime
dependency accepts these peers independently:

```text
@emnapi/core:    ^1.7.1 || ^2.0.0-alpha.4
@emnapi/runtime: ^1.7.1 || ^2.0.0-alpha.4
```

The incompatible combination satisfies both ranges, so installation succeeds.

To recreate this selection in an empty directory, install in this order:

```sh
npm init -y
npm install --save-exact --save-optional @rolldown/binding-wasm32-wasi@1.3.0
npm install --save-exact astro@7.4.7 @astrojs/check@0.9.12 typescript@6.0.3
npx astro check
```

The first install seeds the lockfile with the beta. Although the optional
binding is skipped on ARM64, npm reuses its runtime resolution for the checker.
This explicitly lists the fallback only to isolate the original transitive
source; neither install declares an emnapi package directly.

Installation history matters: a fresh npm install of just Astro, its checker,
and TypeScript selected the working runtime `1.11.3` in testing. The included
lockfile preserves the failing selection so `npm ci` reproduces it reliably.

## System details

Verified with Astro 7.4.7 and `@astrojs/check` 0.9.12, the latest releases
when tested on 2026-11-06. No Astro plugins are configured.

Tested on macOS ARM64 with Node.js 24.18.0 and npm 11.16.0.
Reproduced with pnpm 11 too.
