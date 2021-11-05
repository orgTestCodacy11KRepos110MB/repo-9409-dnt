# dnt - Deno to Node Transform

[![deno doc](https://doc.deno.land/badge.svg)](https://doc.deno.land/https/deno.land/x/dnt/mod.ts)

Prototype for a Deno to npm package build tool.

## What does this do?

It takes a Deno module and creates an npm package for use in Node.js.

There are several steps done in a pipeline:

1. Transforms Deno code to Node/canonical TypeScript including files found by `deno test`.
   - Rewrites module specifiers.
   - Injects a [Deno shim](https://github.com/denoland/deno.ns) for any `Deno` namespace usages.
   - Rewrites Skypack and ESM specifiers to a bare specifier and includes these dependencies in a package.json.
   - When remote modules cannot be resolved to an npm package, it downloads them and rewrites specifiers to make them local.
   - Allows mapping any specifier to an npm package.
1. Type checks the output.
1. Emits ESM, CommonJS, and TypeScript declaration files along with a _package.json_ file.
1. Runs the final output in Node through a test runner calling all `Deno.test` calls.

## Setup

1. Create a build script file:

```ts
// ex. scripts/build_npm.ts
import { build } from "https://deno.land/x/dnt/mod.ts";

await build({
  entryPoints: ["./mod.ts"],
  outDir: "./npm",
  package: {
    // package.json properties
    name: "my-package",
    version: Deno.args[0],
    description: "My package.",
    license: "MIT",
    repository: {
      type: "git",
      url: "git+https://github.com/dsherret/my-package.git",
    },
    bugs: {
      url: "https://github.com/dsherret/my-package/issues",
    },
  },
});

// post build steps
Deno.copyFileSync("LICENSE", "npm/LICENSE");
Deno.copyFileSync("README.md", "npm/README.md");
```

2. Run it and `npm publish`:

```bash
# run script
deno run -A scripts/build_npm.ts 0.1.0

# go to output directory and publish
cd npm
npm publish
```

### Example Build Logs

```
[dnt] Transforming...
[dnt] Running npm install...
[dnt] Building project...
[dnt] Type checking...
[dnt] Emitting declaration files...
[dnt] Emitting ESM package...
[dnt] Emitting CommonJS package...
[dnt] Running tests...

> test
> node test_runner.js

Running tests in ./umd/mod.test.js...

test escapeForWithinString ... ok
test escapeChar ... ok

Running tests in ./esm/mod.test.js...

test escapeForWithinString ... ok
test escapeChar ... ok
[dnt] Complete!
```

## Docs

### Disabling Type Checking, Testing, Declaration Emit, or CommonJS Output

Use the following self-explanatory options to disable any one of these, which are enabled by default:

```ts
await build({
  // ...etc...
  typeCheck: false,
  test: false,
  declaration: false,
  cjs: false,
});
```

### Top Level Await

Top level await doesn't work in CommonJS. If you want to target CommonJS then you'll have to restructure your code to not use any top level awaits. Otherwise, set the `cjs` build options to `false`:

```ts
await build({
  // ...etc...
  cjs: false,
});
```

### Specifier to Npm Package Mappings

By default, dnt will include the remote modules as if they were local in your npm package except in some cases for [skypack](https://www.skypack.dev/) or [esm.sh](https://esm.sh/) remote modules. There are scenarios though where an npm package may exist for one of your dependencies. To use this instead and include it in your package.json file, specify a specifier to npm package mapping.

For example:

```ts
await build({
  // ...etc...
  mappings: {
    "https://deno.land/x/code_block_writer@10.1.1/mod.ts": {
      name: "code-block-writer",
      version: "^10.1.1",
    },
  },
});
```

This will:

1. Change all `"https://deno.land/x/code_block_writer@10.1.1/mod.ts"` specifiers to `"code-block-writer"`
2. Add a package.json dependency for `"code-block-writer": "^10.1.1"`.

Note that if you specify a mapping and it is not found in the code, dnt will error. This is done to prevent the scenario where a remote specifier's version is bumped and the mapping isn't updated.

### Multiple Entry Points

You may wish to distribute a package with multiple entry points. For example, an entry point at `.` and another at `./internal`.

To do this, specify multiple entry points like so:

```ts
await build({
  entryPoints: ["mod.ts", {
    name: "./internal",
    path: "internal.ts",
  }],
  // ...etc...
});
```

This will create a package.json with these as exports:

```jsonc
{
  "name": "my-package",
  // etc...
  "main": "./umd/mod.js",
  "module": "./esm/mod.js",
  "types": "./types/mod.d.ts",
  "exports": {
    ".": {
      "import": "./esm/mod.js",
      "require": "./umd/mod.js",
      "types": "./types/mod.d.ts"
    },
    "./internal": {
      "import": "./esm/internal.js",
      "require": "./umd/internal.js",
      "types": "./types/internal.d.ts"
    }
  }
}
```

Now these entry points could be imported like `import * as main from "my-package"` and `import * as internal from "my-package/internal";`.

### Bin/CLI Packages

To publish an npm [bin package](https://docs.npmjs.com/cli/v7/configuring-npm/package-json#bin) similar to `deno install`, add a `kind: "bin"` entry point:

```ts
await build({
  entryPoints: [{
    kind: "bin",
    name: "my_binary", // command name
    path: "./cli.ts",
  }],
  // ...etc...
});
```

This will add a `"bin"` entry to the package.json and add `#!/usr/bin/env node` to the top of the specified entry point.

### Node and Deno Specific Code

You may find yourself in a scenario where you want to run certain code based on whether someone is in Deno or if someone is in Node and feature testing is not possible. For example, say you want to run the `deno` executable when the code is running in Deno and the `node` executable when it's running in Node.

To do this, it is recommended to use the [`which_runtime`](https://deno.land/x/which_runtime@0.1.0) module. Essentially, import it and then check its exports for if you're in Node or Deno then run specific code based on those scenarios.

### Pre & Post Build Steps

Since the file you're calling is a script, simply add statements before and after the `await build({ ... })` statement:

```ts
// run pre-build steps here

// ex. maybe consider deleting the output directory before build
await Deno.remove("npm", { recursive: true }).catch((_) => {});

await build({
  // ...etc..
});

// run post-build steps here
await Deno.copyFile("LICENSE", "npm/LICENSE");
await Deno.copyFile("README.md", "npm/README.md");
```

### Including Test Data Files

Your Deno tests might rely on test data files. One way of handling this is to copy these files to be in the output directory at the same relative path your Deno tests run with.

For example:

```ts
import { copy } from "https://deno.land/std@x.x.x/fs/mod.ts";

await Deno.remove("npm", { recursive: true }).catch((_) => {});
await copy("testdata", "npm/esm/testdata", { overwrite: true });
await copy("testdata", "npm/umd/testdata", { overwrite: true });

await build({
  // ...etc...
});

// ensure the test data is ignored in the `.npmignore` file
// so it doesn't get published with your npm package
await Deno.writeTextFile(
  "npm/.npmignore",
  "esm/testdata/\numd/testdata/\n",
  { append: true },
);
```

Alternatively, you could also use the [`which_runtime`](https://deno.land/x/which_runtime@0.1.0) module and use a different directory path when the tests are running in Node. This is probably more ideal if you have a lot of test data.

### GitHub Actions - Npm Publish on Tag

1. Ensure your build script accepts a version as a CLI argument and sets that in the package.json object. For example:

   ```ts
   await build({
     // ...etc...
     package: {
       version: Deno.args[0],
       // ...etc...
     },
   });
   ```

   You may wish to removing the leading `v` if it exists (ex. `Deno.args[0]?.replace(/^v/, "")`)

1. In your GitHub Actions workflow, get the tag name, setup node, run your build scripts, then publish to npm.

   ```yml
   # ...setup deno and run `deno test` here as you normally would...

   - name: Get tag version
     if: startsWith(github.ref, 'refs/tags/')
     id: get_tag_version
     run: echo ::set-output name=TAG_VERSION::${GITHUB_REF/refs\/tags\//}
   - uses: actions/setup-node@v2
     with:
       node-version: '16.x'
       registry-url: 'https://registry.npmjs.org'
   - name: npm build
     run: deno run -A ./scripts/build_npm.ts ${{steps.get_tag_version.outputs.TAG_VERSION}}
   - name: npm publish
     if: startsWith(github.ref, 'refs/tags/')
     env:
       NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
     run: |
       cd npm
       npm publish
   ```

   Note that the build script always runs even when not publishing. This is to ensure your build and tests pass on each commit.

## JS API Example

For only the Deno to canonical TypeScript transform which may be useful for bundlers, use the following:

```ts
// docs: https://doc.deno.land/https/deno.land/x/dnt/transform.ts
import { transform } from "https://deno.land/x/dnt/transform.ts";

const outputResult = await transform({
  entryPoints: ["./mod.ts"],
  testEntryPoints: ["./mod.test.ts"],
  shimPackageName: "deno.ns",
  // mappings: {}, // optional specifier mappings
});
```

## Rust API Example

```rust
use std::path::PathBuf;

use deno_node_transform::ModuleSpecifier;
use deno_node_transform::transform;
use deno_node_transform::TransformOptions;

let output_result = transform(TransformOptions {
  entry_points: vec![ModuleSpecifier::from_file_path(PathBuf::from("./mod.ts")).unwrap()],
  test_entry_points: vec![ModuleSpecifier::from_file_path(PathBuf::from("./mod.test.ts")).unwrap()],
  shim_package_name: "deno.ns".to_string(),
  loader: None, // use the default loader
  specifier_mappings: None,
}).await?;
```
