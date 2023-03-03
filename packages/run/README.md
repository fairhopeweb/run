> **Warning**
> This project is in BETA - use at your own peril, but please do provide helpful feedback.

<div align="center">
  <!-- Logo -->
  <h1>
    <img alt="" src="https://user-images.githubusercontent.com/4985201/115444712-ca550500-a1c9-11eb-9897-238ece59129c.png" height="118"/>
    <br/>
    @marko/run
  </h1>

  <!-- Language -->
  <a href="https://www.typescriptlang.org">
    <img src="https://img.shields.io/badge/%3C%2F%3E-typescript-blue.svg" alt="TypeScript"/>
  </a>
</div>

`@marko/run` will help you get up and *running* with [Marko](https://markojs.com)

- Vite plugin that encapsulates [`@marko/vite`](https://github.com/marko-js/vite)
- CLI to simplify build modes
- File-based routing with layouts and middleware
- Efficient routing using a compiled static trie
- [Designed with web standards](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/URLPattern) to run anywhere
- TypeScript support

## Installation

```sh
npm install @marko/run
```

## CLI

The package provides a command line tool `marko-run` which can be run using scripts in your package.json or with npx.

### Getting Started / Zero Config



`marko-run` makes it easy to get started without little to no config. The package ships with a default Vite config and node-based adapter that means a minimal project start can be:
1. Install `@marko/run`
2. Create file `src/routes/+page.marko`
3. Run `npx marko-run`
4. Open browser to `http://localhost:3000`

### Commands

**`dev`** - Start development server in watch mode
```bash
> npx marko-run dev
```
or (default command)
```bash
> npx marko-run
```


**`build`** - Create a production build
```bash
> npx marko-run build
```

**`serve`** - Create a production build and serve
```bash
> npx marko-run serve
```
## Vite Plugin

This package’s Vite plugin discovers your route files, generates the routing code, and registers the `@marko/vite` plugin to compile your `.marko` files.

```ts
// vite.config.ts
import { defineConfig } from "vite";
import marko from "@marko/run/vite"; // Import the Vite plugin

export default defineConfig({
  plugins: [marko()] // Register the Vite plugin
})
```

## Adapters

Adapters provide the means to change the development, build and preview process to fit different deployment platforms and runtimes while allowing authors to write idiomatic code.

### Configure

Specify your adapter in the Vite config when registering the `@marko/run` plugin

```ts
// vite.config.ts
import { defineConfig } from "vite";
import marko from "@marko/run/vite";
import netlify from "@makor/run-adapter-netlify" // Import the adapter

export default defineConfig({
  plugins: [marko({
    adapter: netlify({ edge: true }) // Configure and apply the adapter
  })]
})
```

### Adapter List

- ### [@marko/run-adapter-node](./packages/adapters/node/README.md)
- ### [@marko/run-adapter-netlify](./packages/adapters/netlify/README.md)
- ### [@marko/run-adapter-static](./packages/adapters/static/README.md)

## Runtime

Generally, when using an adapter, this runtime will be abstracted away.

<!-- TODO: Add examples -->
<!-- TODO: Split fetch and match + invoke in two sections and explain why you might use one or the other  -->

```ts
import * as Run from '@marko/run/router`;
```

### Emdedding in Existing Server



### `Run.fetch`

```ts
async function fetch<T>(request: Request, platform: T) => Promise<Response | void>;
```



This asynchronous function takes a [WHATWG `Request` object](https://fetch.spec.whatwg.org/#request-class) object and an object containing any platform specific data you may want access to and returns the [WHATWG `Response` object](https://fetch.spec.whatwg.org/#response-class) from executing any matched route files or undefined if the request was explicitly not handled. If no route matches the requested path, a `404` status code response will be returned. If an error occurs a `500` status code response will be returned.

Express example:
```ts
import express from "express";
import * as Run from "@marko/run/router";

express()
  .use(async (req, res, next) => {
    const request = // ...code to create a WHATWG Request from `req`

    const response = await Run.fetch(request, {
      req,
      res
    });

    if (response) {
      // ...code to apply response to `res`
    } else {
      next();
    }
  })
  .listen(3000);
```


### Other APIs

In some cases you might want more control over when route matching and invokation (creating a response) occur. For instance you may have middleware in your server which need to know if there is a matched route. The runtime provides these additional methods

### `Run.match`

```ts
interface interface Route {
  params: Record<string, string>;
  meta: unknown;
}

function match(method: string, pathname: string) => Route | null;
```

This synchronous function takes an HTTP method and path name, then returns an object representing the best match — or `null` if no match is found.

- `params` - a `{ key: value }` collection of any path parameters for the route
- `meta` - metadata for the route

### `Run.invoke`

```ts
async function invoke<T>(route: Route, request: Request, platform: T) => Promise<Response | void>;
```
This asynchronous function takes a route object returned by [Run.match](#Run.match) the request and platform data and returns a response in the same way the [Run.fetch](#Run.fetch) does.

Express example:
```ts
import express from "express";
import * as Run from "@marko/run/router";

express()
  .use((req, res) => {
    const matchedRoute = Run.match(req.method, req.path);
    if (matchedRoute) {
      req.match = matchedRoute;
    }
  })

  // ...other middleware

  .use(async (req, res, next) => {
    // Check if a route was previously matched
    if (!req.match) {
      next();
      return;
    }

    const request = // ...code to create a WHATWG Request from `req`
    const response = await Run.invoke(req.match, request, {
      req,
      res
    });

    if (response) {
      // ...code to apply response to `res`
    } else {
      next();
    }
  })
  .listen(3000);
```



## File-based Routing

<!-- ### Nested Routing

*🎗 TODO: provide a quick overview* -->

### Routes Directory

The plugin looks for route files in the configured **routes directory**. By default, that’s `./src/routes`, relative to the Vite config file.

To change what directory routes are found in:

```ts
// vite.config.ts
import { defineConfig } from "vite";
import marko from "@marko/run/vite";

export default defineConfig({
  plugins: [marko({
    routesDir: 'src/pages' // Use `./src/pages` (relative to this file) as the routes directory
  })]
})
```

### Routeable Files

To allow for colocation of files that shouldn’t be served (like tests, assets, etc.), the router only recognizes certain filenames.

The following filenames will be discovered in any directory inside your application’s [routes directory](#routes-directory).

#### `+page.marko`

These files establish a route at the current directory path which will be served for `GET` requests with the HTML content of the page. Only one page may exists for any served path.

#### `+layout.marko`

These files provide a **layout component**, which will wrap all nested layouts and pages.

Layouts are like any other Marko component with no extra constraints. Each layout receives the request, path params, URL, and route metadata as input, as well as a `renderBody` which will be the next layout or page to project. 

```marko
<main>
  <h1>My Products</h1>

  ${input.renderBody} // render the page or layout here
</main>
```

#### `+handler.*`

These files establish a route at the current directory path which can handle requests for `GET`, `POST`, `PUT`, and `DELETE` HTTP methods. <!-- TODO: what about HEAD? -->

Typically, these will be `.js` or `.ts` files depending on your project. Like pages, only one handler may exist for any served path. A handler should export functions

<details>
  <summary>More Info</summary>
  
  - Valid exports are functions named `GET`, `POST`, `PUT`, or `DELETE`.
  - Exports can be one of the following
    - Handler function (see below)
    - Array of handler functions - will be composed by calling them in order
    - Promise that resolves to a handler function or array of handler functions 
  - Handler functions are synchronous or asynchronous functions that
    - Receives a `context` and `next` argument,
      - The `context` argument contains the WHATWG request object, path parameters, URL, and route metadata.
      - The `next` argument will call the page for get requests where applicable or return a `204` response.
    - Return a WHATWG response, throw a WHATWG response, return undefined. If the function return's undefined the `next` argument with be automatically called and used as the response.

  ```js
  export function POST(context, next) {
    const { request, params, url, meta } = context;
    return new Response('Successfully updated', { status: 200 });
  }

  export function PUT(context, next) {
    // `next` will be called for you by the runtime
  }

  export async function GET(context, next) {
    // do something before calling `next`
    const response = await next();
    // do something with the response from `next`
    return response;
  }

  export function DELETE(context, next) {
    return new Response('Successfully removed', { status: 204 });
  }
  ```
</details>


#### `+middleware.*`

These files are like layouts, but for handlers. Middleware get called before handlers and let you perform arbitrary work before and after.

> **Note**: Unlike handlers, middleware run for all HTTP methods.

<details>
  <summary>More Info</summary>
  
  - Expects a `default` export that can be one of the following
    - Handler function (see below)
    - Array of handler functions - will be composed by calling them in order
    - Promise that resolves to a handler function or array of handler functions 
  - Handler functions are synchronous or asynchronous functions that
    - Receives a `context` and `next` argument,
      - The `context` argument contains the WHATWG request object, path parameters, URL, and route metadata.
      - The `next` argument will call the page for get requests where applicable or return a `204` response.
    - Return a WHATWG response, throw a WHATWG response, return undefined. If the function return's undefined the `next` argument with be automatically called and used as the response.

  ```ts
  export default async function(context, next) {
    const requestName = `${context.request.method} ${context.url.href}`;
    let success = true;
    console.log(`${requestName} request started`)
    try {
      return await next(); // Wait for subsequent middleware, handler and page
    } catch (err) {
      success = false;
      throw err;
    } finally {
      console.log(`${requestName} completed ${success ? 'successfully' : 'with errors'}`);
    }
  }
  ```
</details>

#### `+meta.*`

These files represent static metadata to attach to the route. This metadata will be automatically provided on the the route `context` when invoking a route.



### Special Files

In addition to the files above which can be defined in any directory under the _routes directory_, there are some special files which can only be defined at the top-level of the _routes directory_. <!-- TODO: do we want to keep this restriction? Having nested 404s would be handy for disambiguating things like “there’s no user with that name” or “that promotion wasn’t found, it may have expired” -->

These special pages are subject to a root layout file (`pages/+layout.marko` in the default configuration).

#### `+404.marko`

This special page responds to any request where:

- The `Accept` request header includes `text/html`
- *And* no other handler or page rendered the request

Responses with this page will have a `404` status code.

#### `+500.marko`

This special page responds to any request where:

- The `Accept` request header includes `text/html`
- *And* an uncaught error occurs while serving the request

Responses with this page will have a `500` status code.

### Execution Order

Given the following routes directory structure

<pre>
routes/
  about/
    +handler.js
    +layout.marko
    +middleware.js
    +page.marko
  +layout.marko
  +middleware.js
  +page.marko
</pre>

When the path `"/about"` is requested, the routable files execute in the following order:

1. Middlewares from root-most to leaf-most
2. Handler
3. Layouts from root-most to leaf-most
4. Page

```mermaid
sequenceDiagram
    participant MW1 as routes/+middleware.js
    participant MW2 as routes/about/+middleware.js
    participant H as routes/about/+handler.js
    participant L1 as routes/+layout.marko
    participant L2 as routes/about/+layout.marko
    participant P as routes/about/+page.marko
    Note over L1,P: Combined at build-time as a single component
    MW1->>MW2: next()
    MW2->>H: next()
    H->>L1: next()
    L1->L2: ${input.renderBody}
    L2->P: ${input.renderBody}
    L1-->>H: Stream Response
    H-->>MW2: Response
    MW2-->>MW1: Response
```

### Path Structure

Within the _routes directory_, the directory structure will determine the path the route will be served. There are four types of directory names: static, pathless, dynamic, and catch-all.

1. **Static directories** - The most common type. Each static directory contributes its name as a segment in the route's served path, like a traditional fileserver. Unless a directory name matches the requirements for one of the below types, it defaults to a static directory.

  Examples:
  ```
  /foo
  /users
  /projects
  ```

2. **Pathless directories** - These directories do **not** contribute their name to the route's served path. Directory names that start with an underscore (`_`) will be a pathless directory.

  Examples:
  ```
  /_users
  /_public
  ```

3. **Dynamic directories** - These directories introduce a dynamic parameter to the route's served path and will match any value at that segment. Any directory name that starts with a single dollar sign (`$`) will be a dynamic directory, and the remaining directory name will be the parameter at runtime. If the directory name is exactly `$`, the parameter will not be captured but it will be matched.

  Examples:
  ```
  /$id
  /$name
  /$
  ```

4. **Catch-all directories** - These directories are similar to dynamic directories and introduce a dynamic parameter, but instead of matching a single path segment, they match to the end of the path. Any directory that starts with two dollar signs (`$$`) will be a catch-all directory, and the remaining directory name will be the parameter at runtime. In the case of a directory named `$$`, the parameter name will not be captured but it will match. Catch-all directories can be used to make `404` Not Found routes at any level, including the root.

  Because catch-all directories match any path segment and consume the rest of the path, you cannot nest route files in them and no further directories will be traversed.

  Examples:
  ```
  /$$all
  /$$rest
  /$$
  ```

<!-- ### Match Ranking

*TODO: Write some things* -->


## TypeScript


### Global Namespace
marko/run provides a global namespace `MarkoRun` with the folling types:

**`MarkoRun.Handler`** - Type that represents a handler function to be exported by a +handler or +middleware file

**`MarkoRun.CurrentRoute`** - Type of the route's params and meta data

**`MarkoRun.CurrentContext`** - Type of the request context object in a handler and `out.global` in your Marko files


### Generated Types
If a [TSConfig](https://www.typescriptlang.org/tsconfig) file is discovered in the project root, the Vite plugin will automatically generate a .d.ts file which provides more specific types for each of your middleware, handlers, layouts and pages. This file will be generated at `.marko-run/routes.d.ts` whenever the project is built - including dev.
> **Note** TypeScript will not include this file by default. If you are not using the [Marko VSCode plugin](https://marketplace.visualstudio.com/items?itemName=Marko-JS.marko-vscode) and you will need to [add it in your tsconfig](https://www.typescriptlang.org/tsconfig#include).

These types are replaced with more specific versions per routeable file:

**`MarkoRun.Handler`**
- Overrides context with specific MarkoRun.CurrentContext

**`MarkoRun.CurrentRoute`**
- Adds specific parameters and meta types 
- In middleware and layouts which are used in many routes, this type will be a union of all possible routes that file will see

**`MarkoRun.CurrentContext`**
- In middleware and layouts which are used in many routes, this type will be a union of all possible routes that file will see.
- When an adapter is used, it can provide types for the platform

## Beta Roadmap

- Error handling
- Error component
- Redirect component