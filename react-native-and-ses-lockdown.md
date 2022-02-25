# React Native + SES lockdown

<!-- MarkdownTOC -->

- Preliminaires
    - Setup steps
    - Look at the bundle
- Add `SES` library file
    - Find out the current version
    - Download the lockdown.umd.js file at the directory root
    - Add the reference to this file at index.js
    - Add the file to the .prettierignore file
    - Prevent lockdown.umd.js to be transformed - with .babelignore
    - Everything OK
- Add `lockdown()` call
    - New Error - extends2.default
- Issues with `@babel/runtime`
    - Causes of Error
    - Debugging Pro-tip
    - Blacklisting @babel/plugin-transform-regenerator
    - Setting up custom plugin arrangement at `babel.config.js`
    - New Error - intrinsics.Promise.resolve.prototype
- Removing the polyfill of `Promise`
    - Comment one line at React Native
    - New Error - intrinsics.%FunctionPrototype%.toString.prototype
- Who is reassigning `Function.prototype.toString`?
    - Disable default integrations in sentry
    - New Error - Attempted to assign to readonly property.
- Where is this `readonly property` ?
    - Patching some `eth` libraries
    - New Error - Exception eventually appears
- The next `readonly property`
    - The `Object.defineProperty` workaround
    - It's working!

<!-- /MarkdownTOC -->

## Preliminaires

### Setup steps

```bash
# Stop!
# Make sure that you installed node, sentry, react native, xcode, etc etc.
# See the README of MM Mobile.

git clone git@github.com:bentobox19/metamask-mobile.git mobile
cd mobile

# It's good to keep up with the original repo
git remote add upstream https://github.com/MetaMask/metamask-mobile.git
git fetch upstream
git merge upstream/main

git checkout -b your-experimental-branch

# Do
npx browserslist@latest --update-db

# And we do
yarn setup

# Other things
cp .ios.env.example .ios.env && \
  cp .android.env.example .android.env && \
  cp .js.env.example .js.env

# Make sure you setup your .js.env file. (without the #)
# export MM_OPENSEA_KEY="123-API-TOKEN"
# export MM_INFURA_PROJECT_ID="123-API-TOKEN"

# Run each command on a separate terminal window
yarn watch
yarn start:ios

# Alternatives to `yarn watch`
## `react-native start`
## `yarn watch:clean ` equivalent to `react-native start --reset-cache`
```
Everything should be working here, provided you followed the above steps to the letter.

### Look at the bundle

    http://localhost:8081/index.bundle?platform=ios&dev=true&minify=false

## Add `SES` library file

### Find out the current version

At https://www.npmjs.com/package/ses

### Download the lockdown.umd.js file at the directory root

```bash
curl -O https://npmfs.com/download/ses/0.15.9/dist/lockdown.umd.js
```

`0.15.9` is 2 days old as of `2022.02.21`.

### Add the reference to this file at index.js

```js
import './shim.js';
import './lockdown.umd.js';

import 'react-native-gesture-handler';
import 'react-native-url-polyfill/auto';
```

### Add the file to the .prettierignore file

`.prettierignore`

```
metro.config.js
jest.preprocessor.js
scripts/metamask-bot-build-announce.js
CHANGELOG.md
lockdown.umd.js
```

### Prevent lockdown.umd.js to be transformed - with .babelignore

Create the file `.babelignore` with the line

```
lockdown.umd.js
```

### Everything OK

👌

## Add `lockdown()` call

At `index.js`

```js
// eslint-disable-next-line
lockdown({consoleTaming: 'unsafe'});

/**
 * Application entry point responsible for registering root component
 */
AppRegistry.registerComponent(name, () => Root);
```

If you don't use the option `{consoleTaming: "unsafe"}`, your app will fail silently.

### New Error - extends2.default

```bash
TypeError: An error was thrown when attempting to render log messages via LogBox.

(0, _extends2.default) is not a function. (In '(0, _extends2.default)({}, parseInterpolation(argsWithoutComponentStack), {
      componentStack: componentStack
    })', '(0, _extends2.default)' is undefined)

parseLogBoxLog
    index.bundle?platform=ios&dev=true&minify=false&modulesOnly=false&runModule=true&app=io.metamask.MetaMask:34632:33
registerWarning
    index.bundle?platform=ios&dev=true&minify=false&modulesOnly=false&runModule=true&app=io.metamask.MetaMask:32199:46
```

## Issues with `@babel/runtime`

### Causes of Error

* `_extends` polyfill at `node_modules/@babel/runtime/helpers/extends.js `.
* `_getPrototypeOf` polyfill at `node_modules/@babel/runtime/helpers/getPrototypeOf.js`.

### Debugging Pro-tip

If you make changes in babel, make sure you re-run react-native with the `--reset-cache` flag.

```bash
react-native start --reset-cache
```

### Blacklisting @babel/plugin-transform-regenerator

Plugins cannot be individually disabled from a preset.

From https://github.com/babel/babel/discussions/14168#discussioncomment-1985730

> Note that the reason that we don't allow it is because plugins are an implementation detail of presets: what matters is the overall result of the preset, and not how it's internally accomplished.

We need either to set up our own `babel.config.js`, or `patch-package` the preset. Let's try the former.

### Setting up custom plugin arrangement at `babel.config.js`

* We derive this configuration from `metro-react-native-babel-preset`

```js
// Borrowing from metro-react-native-babel-preset
const lazyImports = require('metro-react-native-babel-preset/src/configs/lazy-imports');

function isTypeScriptSource(fileName) {
  return !!fileName && fileName.endsWith(".ts");
}

function isTSXSource(fileName) {
  return !!fileName && fileName.endsWith(".tsx");
}

module.exports = {
  comments: false,
  compact: true,
  overrides: [
    {
      plugins: [
        // TODO
        // Check whether we need all this plugins
        // within this block
        '@babel/plugin-transform-flow-strip-types',
        '@babel/plugin-syntax-flow',
        '@babel/plugin-transform-block-scoping',
        ['@babel/plugin-proposal-class-properties', {
           loose: true,
        }],
        '@babel/plugin-syntax-dynamic-import',
        '@babel/plugin-syntax-export-default-from',

        '@babel/plugin-proposal-nullish-coalescing-operator',
        '@babel/plugin-proposal-optional-chaining',

        '@babel/plugin-transform-unicode-regex',
      ]
    },
    {
      test: isTypeScriptSource,
      plugins: [
        [
          '@babel/plugin-transform-typescript',
          {
            isTSX: false,
            allowNamespaces: true
          }
        ]
      ]
    },
    {
      test: isTSXSource,
      plugins: [
        [
          '@babel/plugin-transform-typescript',
          {
            isTSX: true,
            allowNamespaces: true
          }
        ]
      ]
    },
    {
      plugins: [
        '@babel/plugin-transform-react-jsx',
        '@babel/plugin-proposal-export-default-from',
        [
          '@babel/plugin-transform-modules-commonjs',
          {
            strict: false,
            strictMode: false,
            lazy: importSpecifier => lazyImports.has(importSpecifier),
            allowTopLevelThis: true
          }
        ],
        '@babel/plugin-transform-classes',

        'transform-inline-environment-variables',
        'react-native-reanimated/plugin'
      ]
    }
  ]
};
```

### New Error - intrinsics.Promise.resolve.prototype

```bash
failed to delete intrinsics.Promise.resolve.prototype
```

## Removing the polyfill of `Promise`

### Comment one line at React Native

* node_modules/react-native/Libraries/Core/InitializeCore.js

```js
require('./setUpSystrace');
require('./setUpErrorHandling');
// require('./polyfillPromise');
require('./setUpRegeneratorRuntime');
require('./setUpTimers');
```

Updating this patch is _tricky_:

```bash
rm -rf node_modules
# 1. Get the dependencies
yarn

# 2. Apply the changes at /patches
npx patch-package

# 3. Here you do the change to the file, then

# 4. Record your changes
npx patch-package react-native

# 5. commit, push, etc here

# 6. Complete the setup
yarn setup
```

### New Error - intrinsics.%FunctionPrototype%.toString.prototype

```bash
 ERROR  failed to delete intrinsics.%FunctionPrototype%.toString.prototype [TypeError: Unable to delete property.]
```

## Who is reassigning `Function.prototype.toString`?

`sentry` is the one assigning to `Function.prototype.toString`. This is a _System Integration_.

```js
FunctionToString.prototype.setupOnce = function () {
    // eslint-disable-next-line @typescript-eslint/unbound-method
    originalFunctionToString = Function.prototype.toString;
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    Function.prototype.toString = function() {
        var args = [];
        for (var _i = 0; _i < arguments.length; _i++) {
            args[_i] = arguments[_i];
        }
        var context = this.__sentry_original__ || this;
        return originalFunctionToString.apply(context, args);
    };
};
```

From https://docs.sentry.io/platforms/javascript/configuration/integrations/default/

> System integrations are enabled by default to integrate into the standard library or the interpreter itself. They are documented so you can be both aware of what they do and disable them if they cause issues.

From https://docs.sentry.io/platforms/react-native/configuration/options/

> integrations
> In some SDKs, the integrations are configured through this parameter on library initialization. For more information, please see our documentation for a specific integration.

> defaultIntegrations
> This can be used to disable integrations that are added by default. When set to false, no default integrations are added.

### Disable default integrations in sentry

```diff
diff --git a/app/util/setupSentry.js b/app/util/setupSentry.js
index 7b633ec0..a98c1b5a 100644
--- a/app/util/setupSentry.js
+++ b/app/util/setupSentry.js
@@ -18,6 +18,7 @@ export default function setupSentry() {
                dsn,
                debug: __DEV__,
                environment,
+               defaultIntegrations: false,
                integrations: [
                        new Dedupe(),
                        new ExtraErrorData(),
```

### New Error - Attempted to assign to readonly property.

```bash
TypeError: Attempted to assign to readonly property.

ERROR  Error: Requiring module "node_modules/ethjs/node_modules/ethjs-query/lib/index.js", which threw an exception: TypeError: Attempted to assign to readonly property.
ERROR  TypeError: undefined is not a constructor (evaluating 'new (_$$_REQUIRE(_dependencyMap[0], "ethjs-query"))(cprovider, self.options.query)'
```

## Where is this `readonly property` ?

* `node_modules/ethjs/lib/index.js`

```js
var query = new (_$$_REQUIRE(_dependencyMap[0], "ethjs-query"))(cprovider, self.options.query);
```

which calls

* `node_modules/ethjs/node_modules/ethjs-query/lib/index.js`

```js
var _regenerator2 = _interopRequireDefault(_$$_REQUIRE(_dependencyMap[0], "babel-runtime/regenerator"));
```

This is the first line of the file within the bundle. Let's call the `_$$REQUIRE()` function from the console

```js
_$$_REQUIRE(_dependencyMap[0], "babel-runtime/regenerator")
```

Throws

```bash
Error: Requiring module "node_modules/babel-runtime/regenerator/index.js", which threw an exception: TypeError: Attempted to assign to readonly property.
```

We have already removed the regenerator plugin. Investigating into the library, we found that it's referenced in its `package.json`. Further grepping show us more libraries with the same problem.

### Patching some `eth` libraries

To the following files...

* `node_modules/ethjs/node_modules/ethjs-query/package.json`
* `node_modules/ethjs-contract/package.json`
* `node_modules/ethjs/node_modules/ethjs-contract/package.json`

...Do the following change:

`"main": "lib/index.js",` - > `"main": "src/index.js",`

And invoke `npx patch-package --exclude "nothing" <LIB_NAME>`.
This flag `--exclude` is needed to be able to include the `package.json` file into the patch.

### New Error - Exception eventually appears

```bash
TypeError: Attempted to assign to readonly property.
```

There are also some other exceptions that will appear at the debugger

```bash
Exception with thrown value: TypeError: undefined is not an object (evaluating 'ethQuery[method]')

Exception with thrown value: TypeError: undefined is not an object (evaluating 'new this.web3.eth')
```

## The next `readonly property`

* At `node_modules/web3-core-methods/src/index.js`

```js
var Method = function Method(options) {

    if (!options.call || !options.name) {
        throw new Error('When creating a method you need to provide at least the "name" and "call" property.');
    }

    this.name = options.name;
    this.call = options.call;
    this.params = options.params || 0;
```

As `Function.protoype.call` is frozen, `this.call = options.call` throws

```bash
TypeError: Attempted to assign to readonly property.
```

### The `Object.defineProperty` workaround

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

Replace this

```js
this.call = options.call;
```

By this

```js
Object.defineProperty(this, 'call', {
  value: option.call
});
````

### It's working!

* Is it?
* Now we need to do extensive QA. I.e. Check every single functionality of the mobile app. But this is a good start!
