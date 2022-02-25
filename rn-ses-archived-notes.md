# React Native + SES: Archived Notes

## The case of these prototypes

### Facts

* Issue already discovered here https://github.com/endojs/endo/issues/663

* `resolve` function is added to `Promise` here:
* https://github.com/then/promise/blob/91b7b4cb6ad0cacc1c70560677458fe0aac2fa67/src/es6-extensions.js#L24-L47

```js
// Source: node_modules/promise/lib/e6s-extensions.js
Promise.resolve = function (value) {

// At the Bundle
_$$_REQUIRE(_dependencyMap[0], "./core.js").resolve = function (value) {
```

* `prototype` is a non-configurable property, that's why we cannot delete it:

```js
// Console Query (at the bundle)
Object.getOwnPropertyDescriptor(_$$_REQUIRE(_dependencyMap[0], "./core.js").resolve, 'prototype')

// Response
{value: Object, writable: true, enumerable: false, configurable: false} = $5
```

* See https://262.ecma-international.org/6.0/#sec-function-instances-prototype

> Function instances that can be used as a constructor have a prototype property. Whenever such a function instance is created another ordinary object is also created and is the initial value of the function’s prototype property.

* Which is understood at SES in https://github.com/endojs/endo/blob/7b2b7206d8ed5f6cb0a0abe0026c5426f9cc8651/packages/ses/src/whitelist.js#L230-L238

### Solution

* Arrow functions don't have a `prototype`!
* https://blog.logrocket.com/anomalies-in-javascript-arrow-functions/

> Arrow functions can never be used as constructor functions. Hence, they can never be invoked with the new keyword. As such, a prototype property does not exist for an arrow function.

* https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#use_of_the_new_operator
* https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#use_of_prototype_property

### Applying two patches

We apply a patch at `metro-react-native-babel-preset` to avoid transpiling the arrow functions

```diff
diff --git a/node_modules/metro-react-native-babel-preset/src/configs/main.js b/node_modules/metro-react-native-babel-preset/src/configs/main.js
index caa45fe..4ab5085 100644
--- a/node_modules/metro-react-native-babel-preset/src/configs/main.js
+++ b/node_modules/metro-react-native-babel-preset/src/configs/main.js
@@ -79,7 +79,7 @@ const getPreset = (src, options) => {
   } // TODO(gaearon): put this back into '=>' indexOf bailout
   // and patch react-refresh to not depend on this transform.

-  extraPlugins.push([require("@babel/plugin-transform-arrow-functions")]);
+  // extraPlugins.push([require("@babel/plugin-transform-arrow-functions")]);

   if (!isHermes) {
     extraPlugins.push([require("@babel/plugin-transform-computed-properties")]);
```

And a patch to the `promise` library

```diff
diff --git a/node_modules/promise/setimmediate/es6-extensions.js b/node_modules/promise/setimmediate/es6-extensions.js
index 24665f5..cbb2992 100644
--- a/node_modules/promise/setimmediate/es6-extensions.js
+++ b/node_modules/promise/setimmediate/es6-extensions.js
@@ -21,7 +21,7 @@ function valuePromise(value) {
   p._W = value;
   return p;
 }
-Promise.resolve = function (value) {
+Promise.resolve = (value) => {
   if (value instanceof Promise) return value;

   if (value === null) return NULL;
@@ -58,7 +58,7 @@ var iterableToArray = function (iterable) {
   return Array.prototype.slice.call(iterable);
 }

-Promise.all = function (arr) {
+Promise.all = (arr) => {
   var args = iterableToArray(arr);

   return new Promise(function (resolve, reject) {
@@ -98,13 +98,13 @@ Promise.all = function (arr) {
   });
 };

-Promise.reject = function (value) {
+Promise.reject = (value) => {
   return new Promise(function (resolve, reject) {
     reject(value);
   });
 };

-Promise.race = function (values) {
+Promise.race = (values) => {
   return new Promise(function (resolve, reject) {
     iterableToArray(values).forEach(function(value){
       Promise.resolve(value).then(resolve, reject);
```

### New Error

```bash
 ERROR  failed to delete intrinsics.%PromisePrototype%.then.prototype [TypeError: Unable to delete property.]
 ERROR  TypeError: Unable to delete property.
```
