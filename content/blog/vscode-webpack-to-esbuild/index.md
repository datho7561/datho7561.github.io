---
layout: post
title:  "Using esbuild for your VS Code Extensions"
date:   2023-06-13 16:35:00 -0400
draft: false
aliases:
- /permalink/vscode-webpack-to-esbuild
comments: true
---

![An arrow pointing from the webpack icon to the esbuild icon, very evidently drawn hastily in a computer graphics program by an amateur](vscode-webpack-to-esbuild.png)

## Motivation

If you develop a VS Code extension, one of the following has probably happened to you:
1. You packaged the extension using `vsce package`
   and received the notification that your extension consists of thousands of JavaScript files.
   Then, you promptly ignored it.
2. Instead of ignoring it, you followed
   [the tutorial from Microsoft](https://code.visualstudio.com/api/working-with-extensions/bundling-extension#using-webpack)
   to set up `webpack`.
   Now, all your code is in one `.js` file, but it takes a long time to compile.
3. You use VS Code's `Webview` feature in order to include one or more web apps in your extension.
   Each of them uses many libraries.
   The framework you use, React, uses `webpack` to bundle its assets.
   Bundling each of them takes around a minute.
   This leads you to get distracted and doomscroll on Mastodon while waiting for the build to finish.

Recently, I ran into the third issue,
and decided to migrate portions of [vscode-openshift-tools](https://github.com/redhat-developer/vscode-openshift-tools)
from `webpack` to [esbuild](https://esbuild.github.io/).
This took bundling times down from 50 seconds down to under a second,
which significantly sped up launching the extension in debug mode and running the test suites.
I got feedback saying it would be helpful to write down my process for doing this,
so I decided to write this blog post.

In this tutorial, I'm going to cover these scenarios:
- you don't use a bundler at all
- you use `webpack` for your extension code, and it's slow
- you have one or more `Webview`s in your extension built using React, and bundling them is slow
    - I'm focusing on React here, since it's the framework I know best, and `esbuild` requires third-party plugins for the other big frameworks (Svelte, Vue, Angular, etc.).
      If you are using a different framework, this blog post may still be helpful for you, since the process is similar.

## Why `esbuild`?

`esbuild` [claims to be 100x faster](https://esbuild.github.io/faq/#benchmark-details)
than other existing solutions.
It began development after many of the other available solutions (`webpack`, `rollup`, `parcel`) had been well established.
The main goal of the project was fixing many of the annoyances and performance issues of those tools.
It supports many features out of the box that other bundlers need plugins for,
such as bundling TypeScript and `.jsx`/`.tsx` files.

## Table of Contents

- [Determine if it's feasible to use esbuild](#feasibility)
- [Install esbuild](#install-esbuild)
- [Figure out your entry points](#identify-entry-points)
- [Set up `esbuild.mjs`](#set-up-esbuildmjs)
- [Update your package.json scripts](#packagejson-scripts)
- [Update your VS Code launch configurations and tasks](#vs-code-launch-and-tasks)
- [Identify broken dependencies and asynchronous imports](#broken-dependencies)
- [Remove webpack configuration](#removing-webpack)
- [Ignore node_modules when packaging the extension](#node_modules)

### Feasibility

Here are some things that could prevent you from bundling your extension using `esbuild`:
- You have a runtime dependency that does not work well with bundlers (even if it works with `webpack`)

    An example of this is [shelljs](https://github.com/shelljs/shelljs/issues/1035).
    Keep in mind that you can probably find a library that provides the same functionality but works with `esbuild`.
    It can be hard to tell if your dependencies don't bundle well until you try bundling them,
    so I've included instructions for this as a part of the tutorial.
- You rely on importing libraries inside of functions or classes instead of at the top of the file. eg:
```typescript
async function activate(): Promise<void> {
    // the new import syntax
    const myCoolLibrary = await import('my-cool-library');
    myCoolLibrary.myFunction();
    // the old require syntax
    const myOtherCoolLibrary = await require('my-other-cool-library');
    myOtherCoolLibrary.myFunction();
}
```

- If you are using a `webpack` configuration that allows you to import `.png` or `.svg` files using asynchronous imports,
that also causes this problem.

- You need to perform unit tests on the exact `.js` file that is being shipped in the extension

    If you trust that `esbuild` will produce code that is equivalent to what the TypeScript compiler outputs,
    then you can compile your tests in TypeScript as you did before bundling, and test that way.
    However, it's very difficult to unit test the `.js` file that is output by `esbuild`.
    All the code will be bundled into one file.
    The only code that that file exports is whatever your main JavaScript/TypeScript file exports,
    which is probably only the `activate` and `deactivate` functions.
    If you try to rewrite imports in this file using `proxyquire` to call fake library implementations,
    nothing will happen, since the bundled JavaScript code doesn't import any modules.

- You rely on a niche `webpack` plugin

    Remember, most of the features that require a plugin in `webpack` work out of the box in `esbuild`.
    It's only more niche situations that aren't supported.
    Double check `npm` to see if someone has already written the plugin you need for `esbuild`.
    For instance, I relied on [@svgr/webpack](https://www.npmjs.com/package/@svgr/webpack) for a PR I was working on,
    and although there is an equivalent `esbuild` plugin,
    it required a bit more work to integrate,
    since not everything works the same.

### Install `esbuild`

In your project folder, run:

```bash
npm i --save-dev esbuild
```

Then, make sure your `package.json` is updated accordingly.
Consider updating the `semver` [version range](https://docs.npmjs.com/about-semantic-versioning#using-semantic-versioning-to-specify-update-types-your-package-can-accept) listed for `esbuild`;
I tend to use the `^` prefix for developer tools.

### Identify Entry Points

The concept behind a bundler (such as `webpack`, `esbuild`, `rollup`, etc.) is to make a version of one of your files
with all the code from your codebase and dependencies copied into that file,
so that the file doesn't need to `import` or `require` anything.
In order to do this, you need to specify which file to bundle.

If you are looking to bundle your entire extension, and you don't have any webviews,
you have one entry point: the file that contains the `activate` and `deactivate` functions.
For the extensions that I work on at Red Hat, this is `src/extension.ts`.

However, if you are looking to bundle your React Webviews,
this means that somewhere in your code base,
you have one or more TypeScript/JavaScript files that will be loaded into the HTML document corresponding to the Webview.
In a React app, it usually looks something like this:

```tsx
import * as React from 'react';
import * as ReactDOM from 'react-dom';
import { MyComponent } from './my-component.tsx';

ReactDOM.render(<MyComponent></MyComponent>, document.getElementById('root'));
```

Then, somewhere in your codebase, you will have an HTML file,
a template for an HTML file, or a string that contains something like:

```html
<html>
  <head>
    <script src="%SCRIPT%"></script>
  </head>
  ...
  <body>
    <div id="root"></div>
  </body>
</html>
```

where `%SCRIPT%` is replaced at runtime with the URI to the transpiled version of the script mentioned above.
You will need to set all the scripts that you load into HTML webviews as entry points.

Make note of which file(s) are entry points as we will use this information in the next step.

### Set up `esbuild.mjs`

You can run `esbuild` from the command line directly, although putting everything in a script file keeps more configuration out of your `package.json`,
which may already be over 1000 lines long and hard to navigate.

Create a file `esbuild.mjs`.
Keep it somewhere under your project folder, such as `./scripts/esbuild.mjs`.
This file needs to have the `.mjs` extension to let node know that it's an ES6 module file.
Writing an ES6 module file allows us to use top-level `await`, which is important for what we are about to do.
As far as I know, at the time of writing, there isn't an easy way to do this in TypeScript.
In the file, add the following:

```javascript
import * as esbuild from 'esbuild';

await esbuild.build(
    entryPoints: [
        './src/extension.ts',
    ],
    bundle: true,
    outfile: './out/extension.js',
    platform: 'node',
    target: 'node16',
    sourcemap: true,
);
```

This code assumes you have one entry point, `./src/extension.ts`, and bundles the code into `./out/extension.js`.
Read more about the options to pass to it [in the docs](https://esbuild.github.io/getting-started/#bundling-for-node).
If you only have this entry point, you can move on to the next step.

If you have webviews, things get more complex.
This is because you need a slightly different configuration for the webview entry points:
- The `platform` field should be set to `browser`
- The `target` field should be set to the browser version you are targetting.
  In the case of a VS Code extension, this is the version of Chromium that the oldest VS Code that you support is using.

I recommend reading more about these settings in the [esbuild documentation](https://esbuild.github.io/api/#target).

Some other things to consider about our `esbuild.mjs` script:
- You may have multiple Webviews
- It would be nice to bundle the main extension code and Webviews in parallel

Taking all that into account, here's what the `esbuild.mjs` might look like:

```javascript
import * as esbuild from 'esbuild';

await Promises.all([
    esbuild.build(
        entryPoints: [
            './src/extension.ts',
        ],
        bundle: true,
        outfile: './out/extension.js',
        platform: 'node',
        target: 'node16',
        sourcemap: true,
    ),
    esbuild.build(
        entryPoints: [
            './src/webviews/webview1.tsx',
            './src/webviews/webview2.tsx',
            './src/webviews/webview3.tsx',
        ],
        bundle: true,
        outfile: './out/',
        platform: 'browser',
        target: 'chrome108',
        sourcemap: true,
    )
]);
```

This generates the following files:
```
out
├── extension.js
├── extension.js.map
├── webview1.js
├── webview1.js.map
├── webview1.css
├── webview2.js
├── webview2.js.map
├── webview2.css
├── webview3.js
├── webview3.js.map
└── webview3.css
```

Note that there are source maps to assist VS Code with debugging the compiled `.js`,
and that a `.css` file is generated for each React app.
Make sure that your Webview HTML templates reference their corresponding `.css` file.
webpack may have been configured to do this automagically for you,
but `esbuild` won't.

We're not done with this script yet, though.
Here are a list of more considerations for bundling your Webviews:

#### Output Folder Structure

You may want a slightly different folder structure for the output files,
such as each Webview getting put into a sub folder. eg:

```
out
├── webview1
|  ├── index.js
|  ├── index.js.map
|  └── index.css
├── webview2
|  ├── index.js
|  ├── index.js.map
|  └── index.css
└── webview3
   ├── index.js
   ├── index.js.map
   └── index.css
```

Unfortunately, the only way I could find to do this is through using multiple calls to `esbuild.build()`.
I'd suggest something like this:
```javascript
await Promise.all(['webview1', 'webview2', 'webview3']
    .map(entrypoint => esbuild.build(
            entryPoints: [
                `./src/webviews/${entrypoint}.tsx`
            ],
            bundle: true,
            outfile: `./out/${entrypoint}/index.js`,
            platform: 'browser',
            target: 'chrome108',
            sourcemap: true,
        )
    )
);
```

#### Images

You might want to be able to reference images in your app using an import statement. eg.

```javascript
import MyProductIcon from '../../../images/icon.png';
import MyProductIconSvg from '../../../images/icon.svg';
```

In order to enable this, you can add this to your `esbuild` configuration:
```javascript
loader: {
    '.png': 'file',
    '.svg': 'file',
},
```

This will copy the images to the output folder,
and will populate the variables with the path to the image as a string.
For instance, in the code snippet from above,
`MyProductIcon` will contain `'./icon.png'`,
and `MyProductIconSvg` will contain `'./icon.svg'`.
If you have `<base>` set up properly in your HTML,
the images should load if you use `<img src={MyProductIcon}></img>`.

If you want to be able to import `.svg` files as React components,
you will need to look into one of the ports of the [svgr plugin](https://www.npmjs.com/package/@svgr/webpack) to esbuild.
Unfortunately, I don't have time to talk about that in this article.

### `package.json` scripts

Next, we need to rewrite the `package.json` to use our script.
The first thing I should mention is that `esbuild` doesn't check types or run linters,
whereas `webpack` was probably configured to do that.
This means calls to `esbuild` will exit normally with code 0 and generate the output file,
even if there are type errors or linter errors in your codebase.
We need to set up the `npm` script to prevent that.

The first step is installing `npm-run-all`.
You could get away with writing node scripts to combine tasks sequentially or in parallel,
but using `npm-run-all` is much easier.

```bash
npm i --save-dev npm-run-all
```

Now, we need to set up a few tasks:
- A task to check types using `tsc` without producing `.js` files, using the `-noEmit` flag
- A task to run our `esbuild.mjs` script
- A task that combines these steps that is run as a part of the package step

Using `npm-run-all`, we can compose scripts in sequence using `run-s` and in parallel using `run-p`.

Here's a script that checks types and bundles the extension code in parallel:

```javascript
{
    // ...
    "scripts": {
        // ...
        "compile": "run-p compile:checkTypes compile:bundle",
        "compile:checkTypes": "tsc -p . -noEmit",
        "compile:bundle": "node ./scripts/esbuild.mjs",
        // ...
    }
    // ...
}
```

If you have multiple entry points (with multiple `tsconfig` files),
you will need to check types for each of them:

```json
{
    // ...
    "scripts": {
        // ...
        "compile": "run-p compile:checkTypes* compile:bundle",
        "compile:checkTypesExt": "tsc -p . -noEmit",
        "compile:checkTypesWebview1": "tsc -p ./src/webview1 -noEmit",
        "compile:checkTypesWebview2": "tsc -p ./src/webview2 -noEmit",
        "compile:bundle": "node ./scripts/esbuild.mjs",
        // ...
    }
    // ...
}
```

Next, make sure that this step is run as a part of your `vscode:prepublish` step,
so that it gets run when `vsce package` is called.
For example, here is a prepublish script that cleans the `out` directory using the `shx` package,
runs `eslint`, then bundles the source code:

```json
{
    // ...
    "scripts": {
        // ...
        "vscode:prepublish": "run-s clean lint compile",
        "clean": "shx rm -rf out",
        "lint": "eslint .",
        "compile": "run-p compile:checkTypes* compile:bundle",
        "compile:checkTypesExt": "tsc -p . -noEmit",
        "compile:checkTypesWebview1": "tsc -p ./src/webview1 -noEmit",
        "compile:checkTypesWebview2": "tsc -p ./src/webview2 -noEmit",
        "compile:bundle": "node ./scripts/esbuild.mjs",
        // ...
    }
    // ...
}
```

Great! Now, our extension should be bundled properly when we run `vsce package`.

### VS Code Launch and Tasks

We need to adjust a few more configuration files so that the extension starts in debug mode when you hit `F5`.

In your `.vscode/launch.json`, find the task called "Extension Host".
Here's an example of what it might look like:

```json
{
    // ...
    "configurations": [
        // ...
        {
            "name": "Extension",
            "type": "extensionHost",
            "request": "launch",
            "runtimeExecutable": "${execPath}",
            "args": [
                "--extensionDevelopmentPath=${workspaceFolder}"
            ],
            "outFiles": [
                "${workspaceFolder}/out/src/**/*.js"
            ],
            "sourceMaps": true,
            "preLaunchTask": "compile"
        },
        // ...
    ]
}
```

Make note of the name of the `preLaunchTask`.
Open `.vscode/tasks.json`, and find the corresponding prelaunch task.
Delete it, and replace it with the following task:

```json
// ...
{
    "label": "compile",
    "type": "npm",
    "script": "compile",
    "problemMatcher": "$tsc",
    "presentation": {
        "reveal": "silent"
    }
}
// ...
```

This task runs the `npm` script we made earlier, and checks the output for any TypeScript errors before launching.
Any errors on the output will prevent launch.

Note that the `npm` script is being run synchronously each time we launch the extension,
and that it's not a background task.
I made this decision, since VS Code doesn't handle running multiple background tasks well.
I also chose not to run a linting task (like `eslint`),
since, from my testing, that adds a lot of time to the task,
meaning it takes longer for the extension to start debugging.

Update the `preLaunchTask` for "Extension Host" in `.vscode/launch.json` to `"compile"`.

Now, you should be able to launch the extension in debug mode
by selecting the "Extension Host" option from the debug menu and pressing `F5`.

### Broken Dependencies

Now that you're able to launch your extension,
make sure that it works.
The main issue that you're likely to run into is that some libraries load
other libraries asynchronously,
using `await require()` or `await import()`.
The technical reason this causes issues is that `esbuild` only uses
the import statements at the top of a file to detect which files are referenced,
and `esbuild` attempts to remove any code that isn't referenced.

Since these asynchronous import calls could happen in library code,
and the import calls only cause failures when the bundled code is being used,
the best way to detect these failures is through integration tests that test against the built `.vsix`.
If you don't have such a test suite, you can manually test the `.vsix`.

If you run into a library that's not loading:
1. If it's in your code, change the import to be at the top of the file.
   When the extension is running, all the code will be in one file,
   so changing the import to be at the top of the file shouldn't decrease performance.
2. If it's in a library that you depend on, look for alternatives for the library.
   As previously mentioned, an example of a library that doesn't support bundling is `shelljs`.
   Many other packages provide similar functions to the ones it provides.
   Also, if the extension has been under development for a few years,
   see if the functionality exists in a newer version of the NodeJS standard library.
   For example, if you only depend on `shelljs` for the `rm()` function,
   you could replace it with the updated `fs.rm()` function in the standard library,
   or consider the `rimraf` library if that's not an option.
3. If neither of the above can solve your problem,
   you may need to reconsider using `esbuild`.
   If you are bundling your source code, but are still interested in doing so,
   you can consider using `webpack`, since it seems to have hacks to handle asynchronous imports.
   If you are currently bundling using `webpack` and are looking for performance improvements,
   look through the `webpack` docs; there are some configuration options that help it run faster.

Make sure to thoroughly test your extension,
and focus your testing on webviews if you are using any.
Check the extension host for error messages.
Once you are confident that your bundled extension works,
move on to the next step (the fun part).

### Removing Webpack

Remove all the `webpack.config.json`, `webpack.config.js`, etc. files.
Uninstall `webpack`, `webpack-cli`, and any webpack extensions using `npm`,
and ensure they are no longer in your `package.json`.
Make sure that none of the `scripts` in your `package.json` reference webpack.
Also check any additional bash scripts, CI/CD configuration and Containerfiles/Dockerfiles
for references to `webpack` or `npm` scripts you've removed.

### node_modules

Add the following line to `.vscodeignore` (create the file if it doesn't exist):

```
node_modules
```

Build the `.vsix`.
Double check the size of the assembled `.vsix`; it should have shrunk.
You can also unzip the assembled `.vsix` to ensure it doesn't contain `node_modules`.
Just like in the [broken dependencies](#broken-dependencies) section,
test the `.vsix` manually or through integration tests to see if anything is broken.
If everything seems to be working, congrats!
Your extension is now being bundled using `esbuild`.
