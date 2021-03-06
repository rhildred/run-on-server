# babel-plugin-run-on-server

This is the companion babel plugin for run-on-server.

It does two things:

* Replaces `require` with `eval("require")` within `runOnServer` calls
* Compiles "id mappings" to prevent run-on-server from running arbitrary code.

See the documentation below for an explanation of each functionality and the rationale behind it.

## Replace `require` with `eval("require")`

If you try to compile this using webpack:

```js
runOnServer(
  (code) => {
    const prettier = require("prettier");
    return prettier.format(code);
  },
  [code]
);
```

Then webpack will include the "prettier" module in your clientside code.

Since you're only using it serverside, this is unnecesary.

If `eval("require")` is used instead of `require`, then webpack won't include the module in your clientside code:

```js
runOnServer(
  (code) => {
    const prettier = eval("require")("prettier");
    return prettier.format(code);
  },
  [code]
);
```

This plugin can replace `require` with `eval("require")` for you.

## Compiles "ID Mappings"

[`run-on-server`](https://npm.im/run-on-server) lets you run arbitrary code written on the client (browser, node) on the server (node). It can be great for prototyping, but it's insecure out of the box because its server will eval any JavaScript code you throw at it. That makes it unsuitable for production use because any deployed server would be wide open for attackers to run code on.

This babel plugin solves that issue by making `run-on-server/server` refuse to run any code except for the code that appeared in your source.

Here's how it works:

* The plugin reads through your client source code to find all the places you call `runOnServer`
* It replaces the code you pass into `runOnServer` with an autogenerated ID
* It then outputs an "id mappings file" mapping each ID to the code you passed into `runOnServer`.

This id mappings file can then be fed into `run-on-server/server`, which configures the server so that it _only_ runs code appearing in the id mappings file.

It might be easier to visualize with an example:

Normally, `run-on-server/client` code looks like this:

```js
// src/client.js

import createClient from "run-on-server/client";

const runOnServer = createClient("http://localhost:3001");

runOnServer(() => process.version).then((version) => console.log(version));
```

The babel plugin compiles it to this:

```js
// dist/client.js

import createClient from "run-on-server/client";

const runOnServer = createClient("http://localhost:3001");

runOnServer({
  id: "073f3eb62747a1dbb64a5527c79224ad",
}).then((version) => console.log(version));
```

And the babel plugin also creates an id mappings file with this content:

```js
// run-on-server-id-mappings.js

/*
This file was generated by the run-on-server babel plugin. It should not
be edited by hand.
*/ module.exports = {
  "073f3eb62747a1dbb64a5527c79224ad": () => process.version,
};
```

Note that the autogenerated id (`073f3eb62747a1dbb64a5527c79224ad`) is the same in the code and the id mappings file.

Now, if you feed the id mappings into `run-on-server/server`:

```js
const createServer = require("run-on-server/server");

const server = createServer({
  idMappings: require("./run-on-server-id-mappings"),
});

server.listen(3001, () => {
  console.log("Server is listening on port 3001");
});
```

Then the server will be locked down so that it can only ever run `() => process.version`.

By leveraging this babel plugin, `run-on-server` becomes feasible for use in production-facing libraries and applications.

## Installation

Install the package `babel-plugin-run-on-server`:

```sh
$ npm install --save-dev babel-plugin-run-on-server
# OR
$ yarn add --dev babel-plugin-run-on-server
```

Add it to your `.babelrc`:

```json
{
  "plugins": ["run-on-server"]
}
```

By default, the plugin doesn't do anything. You need to enable each feature by passing options to the plugin in your `.babelrc`:

```json
{
  "plugins": [
    [
      "run-on-server",
      {
        "evalRequire": {
          "enabled": true
        },
        "idMappings": {
          "enabled": true,
          "outputPath": "./server/idMappings.js"
        }
      }
    ]
  ]
}
```

`evalRequire.enabled`

Whether to compile `require` into `eval("require")` within `runOnServer` calls. Defaults to `false`.

`idMappings.enabled`

Whether to compile id mappings and replace the code in `runOnServer` calls with id mappings. Defaults to `false`.

`idMappings.outputPath`

By default, the id mappings file will be written to `run-on-server-id-mappings.js` in whatever directory you run babel in. You can configure it by setting the `idMappings.outputPath` option.

## Notes/Gotchas

### ID mappings file contains old entries until recreated

The plugin never removes entries from the id mappings file, it only adds them. This means that your id mappings file will grow over time and contain entries for code you no longer have. To clean the entries up, remove the id mappings file and re-run babel. It's best practice to clear out all entries before releasing a new production-facing build.

If you are building content from a `src` directory into a `dist` directory, my recommendation is to output the id mappings file into `dist`, and then clear the contents of `dist` before running each build (as part of your build script).

### outputPath resolution may not work as expected

The outputPath in `.babelrc` is resolved relative to where you run `babel` (`process.cwd()`). This usually works, but if your id mappings file is getting put in the wrong place, you might want to use this workaround to give an absolute path to the plugin:

1. Change your `.babelrc` to have this content:

```json
{
  "presets": ["./.babelrc.js"]
}
```

2. Create a new file named `.babelrc.js` in the same directory as `.babelrc`, and make it export a config object containing everything your `.babelrc` had in it before:

```js
module.exports = {
  plugins: [
    [
      "run-on-server",
      {
        idMappings: {
          enabled: true,
          outputPath: "./server/idMappings.js",
        },
      },
    ],
  ],
};
```

3. Import the `path` module and wrap your `outputPath` like so:

```js
const path = require("path");

module.exports = {
  plugins: [
    [
      "run-on-server",
      {
        idMappings: {
          enabled: true,
          outputPath: path.resolve(__dirname, "./server/idMappings.js"),
        },
      },
    ],
  ],
};
```

> Note: If you are using Babel 7, you don't need to do the `"presets": ["./.babelrc.js"]` `.babelrc` workaround, as `.babelrc.js` will be loaded by babel without any additional config.
