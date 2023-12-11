---
title: Migrating to SvelteKit v2
---

Upgrading from SvelteKit version 1 to version 2 should be mostly seamless. There are a few breaking changes to note, which are listed here. You can use `npx svelte-migrate sveltekit-2` to migrate some of these changes automatically.

We highly recommend upgrading to the most recent 1.x version before upgrading to 2.0, so that you can take advantage of targeted deprecation warnings. We also recommend [updating to Svelte 4](https://svelte.dev/docs/v4-migration-guide) first: Later versions of SvelteKit 1.x support it, and SvelteKit 2.0 requires it.

## `redirect` and `error` are no longer thrown by you

Previously, you had to `throw` the values returned from `error(...)` and `redirect(...)` yourself. In SvelteKit 2 this is no longer the case — calling the functions is sufficient.

```diff
import { error } from '@sveltejs/kit'

...
- throw error(500, 'something went wrong');
+ error(500, 'something went wrong');
```

`svelte-migrate` will do these changes automatically for you.

If the error or redirect is thrown inside a `try {...}` block (hint: don't do this!), you can distinguish them from unexpected errors using [`isHttpError`](/docs/modules#sveltejs-kit-ishttperror) and [`isRedirect`](/docs/modules#sveltejs-kit-isredirect) imported from `@sveltejs/kit`.

## path is required when setting cookies

When receiving a `Set-Cookie` header that doesn't specify a `path`, browsers will [set the cookie path](https://www.rfc-editor.org/rfc/rfc6265#section-5.1.4) to the parent of the resource in question. This behaviour isn't particularly helpful or intuitive, and frequently results in bugs because the developer expected the cookie to apply to the domain as a whole.

As of SvelteKit 2.0, you need to set a `path` when calling `cookies.set(...)`, `cookies.delete(...)` or `cookies.serialize(...)` so that there's no ambiguity. Most of the time, you probably want to use `path: '/'`, but you can set it to whatever you like, including relative paths — `''` means 'the current path', `'.'` means 'the current directory'.

```diff
export function load({ cookies }) {
-    cookies.set(name, value);
+    cookies.set(name, value, { path: '/' });
    return { response }
}
```

## Top-level promises are no longer awaited

In SvelteKit version 1, if the top-level properties of the object returned from a `load` function were promises, they were automatically awaited. With the introduction of [streaming](https://svelte.dev/blog/streaming-snapshots-sveltekit) this behavior became a bit awkward as it forces you to nest your streamed data one level deep.

As of version 2, SvelteKit no longer differentiates between top-level and non-top-level promises. To get back the blocking behavior, use `await` (with `Promise.all` to prevent waterfalls, where appropriate):

```diff
// If you have a single promise
export function load({ fetch }) {
-    const response = fetch(...).then(r => r.json());
+    const response = await fetch(...).then(r => r.json());
    return { response }
}
```

```diff
// If you have multiple promises
export function load({ fetch }) {
-    const a = fetch(...).then(r => r.json());
-    const b = fetch(...).then(r => r.json());
+    const [a, b] = Promise.all([
+      fetch(...).then(r => r.json()),
+      fetch(...).then(r => r.json()),
+    ]);
    return { a, b };
}
```

## `path` is now a required option for cookies

`cookies.set`, `cookies.delete` and `cookies.serialize` all have an options argument through which certain cookie serialization options are configurable. One of the is the `path` setting, which tells browser under which URLs a cookie is applicable. In SvelteKit 1.x, the `path` is optional and defaults to what the browser does, which is removing everything up to and including the last slash in the pathname of the URL. This means that if you're on `/foo/bar`, then the `path` is `/foo`, but if you're on `/foo/bar/`, the `path` is `/foo/bar`. This behavior is somewhat confusing, and most of the time you probably want to have cookies available more broadly (many people set `path` to `/` for that reason) instead of scratching their heads why a cookie they have set doesn't apply elsewhere. For this reason, `path` is a required option in SvelteKit 2.

```diff
// file: foo/bar/+page.svelte
export function load ({ cookies }) {
-    cookies.set('key', 'value');
+    cookies.set('key', 'value', { path: '/foo' });
}
```

`svelte-migrate` will add comments highlighting the locations that need to be adjusted.

## goto(...) no longer accepts external URLs

To navigate to an external URL, use `window.location = url`.

## paths are now relative by default

In SvelteKit 1, `%sveltekit.assets%` in your `app.html` was replaced with a relative path by default (i.e. `.` or `..` or `../..` etc, depending on the path being rendered) during server-side rendering unless the [`paths.relative`](/docs/configuration#paths) config option was explicitly set to `false`. The same was true for `base` and `assets` imported from `$app/paths`, but only if the `paths.relative` option was explicitly set to `true`.

This inconsistency is fixed in version 2. Paths are either always relative or always absolute, depending on the value of [`paths.relative`](/docs/configuration#paths). It defaults to `true` as this results in more portable apps: if the `base` is something other than the app expected (as is the case when viewed on the [Internet Archive](https://archive.org/), for example) or unknown at build time (as is the case when deploying to [IPFS](https://ipfs.tech/) and so on), fewer things are likely to break.

## Server fetches are not trackable anymore

Previously it was possible to track URLs from `fetch`es on the server in order to rerun load functions. This poses a possible security risk (private URLs leaking), and as such it was behind the `dangerZone.trackServerFetches` setting, which is now removed.

## Updated dependency requirements

SvelteKit requires Node `18.13` or higher, Vite `^5.0`, vite-plugin-svelte `^3.0`, TypeScript `^5.0` and Svelte version 4 or higher. `svelte-migrate` will do the `package.json` bumps for you.

As part of the TypeScript upgrade, the generated `tsconfig.json` (the one your `tsconfig.json` extends from) now uses `"moduleResolution": "bundler"` (which is recommended by the TypeScript team, as it properly resolves types from packages with an `exports` map in package.json) and `verbatimModuleSyntax` (which replaces the existing `importsNotUsedAsValues ` and `preserveValueImports` flags — if you have those in your `tsconfig.json`, remove them. `svelte-migrate` will do this for you).