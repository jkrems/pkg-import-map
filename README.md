# Bare Module Specifier Resolution in node.js

**Contributors:** Guy Bedford, Geoffrey Booth, Jan Krems, Saleh Abdel Motaal

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server` for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.
* A package wants to expose multiple entrypoints as its public API without leaking internal directory structure.

## High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `/x` will only ever import exactly the sibling file "x" without appending paths or extensions.
  `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.
* Resolution should not depend on file extensions, allowing ESM syntax in `.js` files.
* The directory structure of a module should be treated as private implementation detail.

## `package.json` Interface

We propose a field in `package.json` to specify one or more entrypoint locations when importing bare specifiers.

> **The key is TBD, the examples use `"exports"` as a placeholder.**
> **Neither the name nor the fact that it exists top-level is final.**

The `package.json` `"exports"` interface will only be respected for bare specifiers, e.g. `import _ from 'lodash'` where the specifier `'lodash'` doesn’t start with a `.` or `/`.

`"exports"` works in concert with the `package.json` `"type": "module"` signifier that a package can be imported as ESM by Node.
`"exports"` by itself does not signify that a package should be treated as ESM, but `"exports"` is currently ignored for packages `require`d as CommonJS.
Extending this feature to CommonJS may occur in the future, but issues of backward compatibility would need to be addressed.
At the moment `"exports"` is limited to ESM packages or dual ESM/CommonJS packages that are imported as ESM.

For `"type": "module"` packages with both `"main"` and `"exports"`, a main entrypoint defined by `"exports"` takes precedence over one defined by `"main"`.
This allows a package to be importable as either ESM or CommonJS.
If a `package.json` lacks `"exports"` but includes `"type": "module"`, `"main"` defines the package’s ESM entrypoint.

### Example

Here’s a complete `package.json` example, for a hypothetical module named `@momentjs/moment`:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    "/": "./src/util/",
    "/timezones/": "./data/timezones/",
    "/timezones/utc": "./data/timezones/utc/index.mjs"
  }
}
```

Within the `"exports"` object, the key string from the `'/'` is concatenated on the end of the name field, e.g. `import utc from '@momentjs/moment/timezones/utc'` is formed from `'@momentjs/moment'` + `'/timezones/utc'`. Note that this is string manipulation, not a file path: `"./timezones/utc"` is allowed, but just `"timezones/utc"` is not. The `.` is a placeholder representing the package name. The main entrypoint is therefore the dot string, `".": "./src/moment.mjs"`.

Keys that end in slashes can map to folder roots, following the [pattern in the browser import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes): `"/timezones/": "./data/timezones/"` would allow `import pdt from "@momentjs/moment/timezones/pdt.mjs"` to import `./data/timezones/pdt.mjs`.

- Using `"/"` maps the root, so `"/": "./src/util/"` would allow `import tick from "@momentjs/moment/tick.mjs"` to import `./src/util/tick.mjs`.

- Mapping a key of `"/"` to a value of `"/"` exposes all files in the package, where `"/": "/"` would allow `import privateHelpers from "@momentjs/moment/private-helpers.mjs"` to import `./private-helpers.mjs`.

- When mapping to a folder root, both the left and right sides must end in slashes: `"/": "/dist/"`, not `"/": "./dist"`.

- Unlike in CommonJS, there is no automatic searching for `index.js` or `index.mjs` or for file extensions. This matches the [behavior of the import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes).

The value of an export, e.g. `"/src/moment.mjs"`, must begin with `/` to signify a path within the package (e.g. "/src" is okay, but `"./src"` or `"src"` are not). This is to reserve potential future use for `"exports"` to also remap URLs or perform internal remappings of imports within the package boundary.

There is the potential for collisions in the exports, such as `"/timezones/"` and `"/timezones/utc"` in the example above (e.g. if there’s a file named `utc` in the `./data/timezones` folder).
Rough outline of a possible resolution algorithm:

1. Find the package matching the base specifier, e.g. `@momentjs/moment` or `request`.
1. Load its exports map.
1. If there is an exact match for the requested specifier, return the resolution.
1. Otherwise, find the longest matching path prefix. If there is a path prefix, return the resolution by applying the prefix.
1. Return an error - no mapping found.

In the future, the algorithm might be adjusted to align with work done in the [import maps proposal](https://github.com/domenic/import-maps).

### Usage

For a consumer, the above `@momentjs/moment` and `request` packages can be used as follows, assuming the user’s project is in `/app` with `/app/package.json` and `/app/node_modules`:

```js
import request from 'request';
// Loads file:///app/node_modules/request/request.mjs

import request from './node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import request from 'file:///app/node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import utc from '@momentjs/moment/timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import request from 'request/';
// Error: trailing slash not mapped

import request from 'request/request.mjs';
// Error: no such mapping

import moment from '@momentjs/moment/';
// Error: trailing slash not allowed (cannot import folders, only files)

import request from 'file:///app/node_modules/request';
// Error: folders cannot be imported, package.json only considered for bare imports

import request from 'file:///app/node_modules/request/';
// Error: folders cannot be imported, package.json only considered for bare imports

import utc from '@momentjs/moment/timezones/utc/'; // Note trailing slash
// Error: folders cannot be imported (there is no index.* magic)
```

### Prior Art

* [`package.json#browser`](https://github.com/defunctzombie/package-browser-field-spec)
* [Import Maps](https://github.com/domenic/import-maps)
* [`package.json#mimes`](https://github.com/nodejs/modules/pull/160)
* [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)
