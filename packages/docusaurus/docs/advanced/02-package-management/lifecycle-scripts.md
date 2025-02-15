---
category: advanced
slug: /advanced/lifecycle-scripts
title: "Lifecycle Scripts"
description: An overview of Yarn's supported lifecycle scripts.
---

Packages can define in the `scripts` field of their manifest various actions that should be executed when the package manager executes a particular workflow.

:::info
Note that we don't support every single lifecycle script originally present in npm. This is a deliberate decision based on the observation that too many lifecycle scripts make it difficult to know which one to use in which circumstances, leading to confusion and mistakes. We are open to add the missing ones on a case-by-case basis if compelling use cases are provided.

In particular, we intentionally don't support arbitrary `pre` and `post` hooks for user-defined scripts (such as `prestart`). This behavior caused scripts to be implicit rather than explicit, obfuscating the execution flow. It also sometimes led to surprising behaviors, like `yarn serve` also running `yarn preserve`.
:::

## `prepack` and `postpack`

Those script are called right at the beginning and the end of each call to `yarn pack`. They are respectively meant to turn your package from development into production, and cleanup any lingering artifact. For instance, a typical `prepack` script would call Babel or TypeScript on the source directory to turn `.ts` files into `.js` files.

:::info
Although rarely called directly, `yarn pack` is a crucial part of Yarn. Each time Yarn has to fetch a dependency from a "raw" source (such as a Git repository), it will automatically run `yarn install` and `yarn pack` to generate the package to use.
:::

## `prepublish`

This script is called before `yarn npm publish` before the package has even been packed. This is the place where you'll want to check that the project is in an ok state.

:::caution
Because it's only called on prepublish, **the prepublish hook shouldn't have side effects.** In particular don't transpile the package sources in `prepublish`, as people consuming directly your repository (such as through the [`git:` protocol](/protocol/git)) wouldn't be able to use your project. Instead, use `prepack`.
:::

## `postinstall`

This script is called after the package dependency tree changed in any way -- usually after a dependency (or transitive dependency) got added, removed, or updated, but also sometimes when the project configuration or environment changed (for example when changing the Node.js version).

It is guaranteed to be called in topological order (in other words, your dependencies' `postinstall` scripts will always run before yours).

For backwards compatibility, the `preinstall` and `install` scripts, if presents, are called right before running the `postinstall` script from the same package. In general, prefer using `postinstall` over those two.

:::caution
Postinstall scripts have should be avoided at all cost, as they make installs slower and riskier. Many users will refuse to install dependencies that have `postinstall` scripts. Additionally, since the output isn't shown out of the box, using them to print a message to the user will not work as you expect.
:::

## Environment variables

When running scripts and binaries, some environment variables are usually made available:

| Variable | Description |
| --- | --- |
| `$INIT_CWD` | Directory from which the script has been invoked. This isn't the same as the cwd, which for scripts is always equal to the closest package root. |
| `$PROJECT_CWD` | Root of the project on the filesystem. |
| `$npm_package_name` | Name of the running package. |
| `$npm_package_version` | Version of the running package. |
| `$npm_package_json` | Absolute path to the `package.json` of the running package. |
| `$npm_execpath` | Absolute path to the Yarn binary. |
| `$npm_node_execpath` | Absolute path to the Node binary. |
| `$npm_config_user_agent` | String defining the Yarn version currently in use. |
| `$npm_lifecycle_event` | Name of the script or lifecycle event, if relevant. |
