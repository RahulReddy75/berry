---
category: getting-started
slug: /migration/pnp
title: "Yarn PnP migration"
description: A step-by-step and in-depth migration guide from Yarn 1 (Classic) to Yarn 2 (Berry).
sidebar_position: 3
sidebar_label: "To go further"
---

:::info
This step is **completely optional**!

While we recommend to use [Yarn Plug'n'Play](/features/pnp) for new projects, enabling it on existing projects may require a time investment. Feel free to skip this part if you prefer, and to come back to it whenever you have more time and/or a concrete benefit to get from it.
:::

## Calling the Doctor

Plug'n'Play enforces strict [dependency rules](/advanced/rulebook). You'll get errors should something in your application rely on unlisted dependencies which could cause your application to become unstable.

To quickly detect which places may rely on unsafe patterns, Yarn provides a tool called the Doctor. Just run `yarn dlx @yarnpkg/doctor` in your project and the Doctor will start looking at your source files to detect any potentially problematic pattern.

### Example

For example, here's what the Doctor used to say about `webpack-dev-server`:

```
➤ YN0000: Found 1 package(s) to process
➤ YN0000: For a grand total of 236 file(s) to validate

➤ YN0000: ┌ /webpack-dev-server/package.json
➤ YN0000: │ /webpack-dev-server/test/testSequencer.js:5:19: Undeclared dependency on @jest/test-sequencer
➤ YN0000: │ /webpack-dev-server/client-src/default/webpack.config.js:12:14: Webpack configs from non-private packages should avoid referencing loaders without require.resolve
➤ YN0000: │ /webpack-dev-server/test/server/contentBase-option.test.js:68:8: Strings should avoid referencing the node_modules directory (prefer require.resolve)
➤ YN0000: └ Completed in 5.12s

➤ YN0000: Failed with errors in 5.12s
```

We can see that the Doctor spotted a couple of legitimate issues:

- `testSequencer.js` depends on `@jest/test-sequencer` without listing it as a proper dependency - which would be reported as an error at runtime under Yarn Plug'n'Play, as nothing guarantees that the version of `@jest/test-sequencer` would match what the package has been tested with.

- `webpack.config.js` references a loader without passing its name through `require.resolve` - this is unsafe, as it means the loader will be resolved relative to the `webpack` package, rather than `webpack-dev-server`'s dependencies.

- `contentBase-option.test.js` checks the content of the `node_modules` folder - which wouldn't exist anymore under Plug'n'Play.

## Enabling Yarn PnP

1. Look into your `.yarnrc.yml` file for the [`nodeLinker`](/configuration/yarnrc#nodeLinker) setting.
2. If you don't find it, or if it's set to `pnp`, then it's all good: you're already using Yarn Plug'n'Play!
3. Otherwise, remove it from your configuration file and run `yarn install`.
4. Commit the changes.

### What to look for

Now you should have a working Yarn Plug'n'Play setup, but your repository might still need some extra care. Some things to keep in mind:

- There are no `node_modules` folders. Use `require.resolve` instead.
- There are no `.bin` folders. If you relied on them, use [`yarn run bin`](#call-binaries-using-yarn-run-rather-than-node_modulesbin) instead.
- Replace any calls to `node` that are not inside the `scripts` field by `yarn node`.
- Custom pre-hooks (e.g. `prestart`) need to be called manually now (`yarn prestart`).

All of this and more is documented in the following sections. In general, we advise you at this point to try to run your application and see what breaks, then check here to find out tips on how to correct your install.

### Editor support

:::tip
We only cover VSCode here, but we have a dedicated [documentation page](/getting-started/editor-sdks) covering more IDEs!
:::

:::caution
Make sure that `typescript`, `eslint`, `prettier`, ... all dependencies typically used by your IDE extensions are listed at the *top level* of the project (rather than in an arbitrary workspace).
:::

1. Install the [ZipFS](https://marketplace.visualstudio.com/items?itemName=arcanis.vscode-zipfs) VSCode extension.
2. Run `yarn dlx @yarnpkg/sdks vscode` and commit the changes.
3. For TypeScript, don't forget to select [Use Workspace Version](https://code.visualstudio.com/docs/typescript/typescript-compiling#_using-the-workspace-version-of-typescript) in VSCode.

## General Advices

### Fix dependencies with `packageExtensions`

Packages sometimes forget to list their dependencies. In the past it used to cause many subtle issues, so Yarn now defaults to prevent such unsound accesses. Still, we don't want it to prevent you from doing your work as long as you can do it in a safe and predictable way, so we came up with the [`packageExtensions`](/configuration/yarnrc#packageExtensions) setting.

For example, if `react` was to forget to list a dependency on `prop-types`, you'd fix it like this:

```yaml
packageExtensions:
  "react@*":
    dependencies:
      prop-types: "*"
```

And if a Babel plugin was missing its peer dependency on `@babel/core`, you'd fix it with:

```yaml
packageExtensions:
  "@babel/plugin-something@*":
    peerDependencies:
      "@babel/core": "*"
```

Should you use dependencies or peer dependencies? It depends on the context; as a rule of thumb, if the package is a singleton (for example `react`, or `react-redux` which also relies on the React context), you'll want to make it a peer dependency. In other cases, where the package is just a collection of utilities, using a regular dependency should be fine (for example `tslib`, `lodash`, etc).

### Call binaries using `yarn run bin` rather than `node_modules/.bin`

The `node_modules/.bin` folder is an implementation detail, and the PnP installs don't generate it at all. Rather than relying on its existence, just use the `yarn run bin` command which can start both scripts and binaries:

```bash
yarn run jest
# or, using the shortcut:
yarn jest
```

### Call your scripts through `yarn node` rather than `node`

We now need to inject some variables into the environment for Node to be able to locate your dependencies. In order to make this possible, we ask you to use `yarn node` which transparently does the heavy lifting.

**Note:** this section only applies to the _shell CLI_. The commands defined in your `scripts` are unaffected, as we make sure that `node` always points to the right location, with the right variables already set.

### Setup your IDE for PnP support

Since Yarn Plug'n'Play doesn't generate `node_modules` folders, some IDE integrations may not work out of the box. Check our [guide](/getting-started/editor-sdks) to see how to fix them.

### Take a look at our end-to-end tests

We now run daily [end-to-end tests](https://github.com/yarnpkg/berry#current-status) against various popular JavaScript tools in order to make sure that we never regress - or be notified when third-party project ship incompatible changes.

Consulting the sources for those tests is a great way to check whether some special configuration values have to be set when using a particular toolchain.

## Troubleshooting

### `Cannot find module [...]`

This error **doesn't** come from Yarn: it's emitted by the Node.js resolution pipeline, telling you a package cannot be found on disk.

If you have enabled Plug'n'Play, then the Node.js resolution pipeline is supposed to forward resolution requests to Yarn - meaning that if you get this message, it's that this forwarding didn't occur, and your first action should be to figure out why.

Usually, it'll be because you called a Node.js script using `node ./my-script` instead of `yarn node ./my-script`.

### `A package is trying to access [...]`

Although rare, some packages don't list all their dependencies. Now that we enforce boundaries between the various branches of the dependency tree, this kind of issue is more apparent than it used to be (although it's always been problematic).

The long term fix is to submit a pull request upstream to add the missing dependency to the package listing. Given that it sometimes might take some time before they get merged, we also have a more short-term fix available: create `.yarnrc.yml` in your project, then use the [`packageExtensions` setting](#fix-dependencies-with-packageextensions) to add the missing dependency to the relevant packages. Run `yarn install` to apply your changes, and voilà!

Should you choose to open a PR on the upstream repository, you will also be able to contribute your package extension to our [`plugin-compat` database](https://github.com/yarnpkg/berry/blob/master/packages/plugin-compat/sources/extensions.ts), helping the whole ecosystem move forward.
