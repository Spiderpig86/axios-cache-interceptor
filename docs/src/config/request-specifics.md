# Request specifics

Each request can have its own cache customization, by using the `cache` property. This
way, you can have requests behaving differently from each other without much effort.

The inline documentation is self explanatory, but here is a shortly brief of what each
property does:

## id

<Badge text="optional" type="warning"/>

- Type: `string`
- default: _(auto generated by the current
  [key generator](../guide/request-id.md#custom-generator))_

You can override the request id used by this property.
[See more about ids](../guide/request-id.md).

## cache

<Badge text="optional" type="warning"/>

- Type: `false` or `Partial<CacheProperties<R, D>>`.
- Default: `{}` _(Inherits from global options)_

::: tip

This property is optional, and if not provided, the default cache properties will be used.

:::

The cache option available through the request config is where all the cache customization
happens.

Setting the `cache` property to `false` will disable the cache for this request.

This does not mean that the cache will be excluded from the storage, in which case, you
can do that by deleting the storage entry:

```ts
// Make a request with cache disabled.
const { id: requestId } = await axios.get('url', { cache: false });

// Delete the cache entry for this request.
await axios.storage.remove(requestId);
```

## cache.ttl

<Badge text="optional" type="warning"/>

- Type: `number`
- Default: `1000 * 60 * 5` _(5 Minutes)_

::: warning

When using [**interpretHeader**](#cache-interpretheader), this value will only be used if
the interpreter can't determine their TTL value to override this one.

:::

The time until the cached value is expired in milliseconds.

If a function is used, it will receive the complete response and waits to return a TTL
value

## cache.interpretHeader

<Badge text="optional" type="warning"/>

- Type: `boolean`
- Default: `true`

If activated, when the response is received, the `ttl` property will be inferred from the
requests headers. As described in the MDN docs and HTML specification.

::: tip

You can override the default behavior by setting the
**[headerInterpreter](../config.md#headerinterpreter)** when creating the cached axios
client.

:::

See the actual implementation of the
[`interpretHeader`](https://github.com/arthurfiorette/axios-cache-interceptor/blob/main/src/header/interpreter.ts)
method for more information.

## cache.methods

<Badge text="optional" type="warning"/>

- Type: `Method[]`
- Default: `["get"]`

Specify what request methods should be cached.

Defaults to only cache `GET` methods.

## cache.cachePredicate

<Badge text="optional" type="warning"/>

- Type: `CachePredicate<R, D>`
- Default: `{}`

An object or function that will be tested against the response to test if it can be
cached.

```ts{5,8,13}
axios.get<{ auth: { status: string } }>('url', {
  cache: {
    cachePredicate: {
      // Only cache if the response comes with a "good" status code
      statusCheck: (status) => /* some calculation */ true,

      // Tests against any header present in the response.
      containsHeaders: {
        'x-custom-header-3': (value) => /* some calculation */ true
      },

      // Check custom response body
      responseMatch: ({ data }) => {
        // Sample that only caches if the response is authenticated
        return data.auth.status === 'authenticated';
      }
    }
  }
});
```

## cache.update

<Badge text="optional" type="warning"/>

- Type: `CacheUpdater<R, D>`
- Default: `{}`

Once the request is resolved, this specifies what other responses should change their
cache. Can be used to update the request or delete other caches. It is a simple `Record`
with the request id.

Here's an example with some basic login:

Using a function instead of an object is supported but not recommended, as it's better to
just consume the response normally and write your own code after it. But it`s here in case
you need it.

```ts
// Some requests id's
let profileInfoId;
let userInfoId;

axios.post<{ auth: { user: User } }>(
  'login',
  { username, password },
  {
    cache: {
      update: {
        // Evicts the profile info cache, because now he is authenticated and the response needs to be re-fetched
        [profileInfoId]: 'delete',

        // An example that update the "user info response cache" when doing a login.
        // Imagine this request is a login one.
        [userInfoResponseId]: (cachedValue, response) => {
          if (cachedValue.state !== 'cached') {
            // Only needs to update if the response is cached
            return 'ignore';
          }

          cachedValue.data = data;

          // This returned value will be returned in next calls to the cache.
          return cachedValue;
        }
      }
    }
  }
);
```

## cache.etag

<Badge text="optional" type="warning"/>

- Type: `boolean`
- Default: `true`

If the request should handle
[`ETag`](https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Headers/ETag) and
[`If-None-Match support`](https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Headers/If-None-Match).
Use a string to force a custom static value or true to use the previous response ETag.

To use `true` (automatic ETag handling), `interpretHeader` option must be set to `true`.

## cache.modifiedSince

<Badge text="optional" type="warning"/>

- Type: `boolean`
- Default: `true`

Use `If-Modified-Since` header in this request. Use a date to force a custom static value
or true to use the last cached timestamp.

If never cached before, the header is not set.

If `interpretHeader` is set and a `Last-Modified` header is sent to us, then value from
that header is used, otherwise cache creation timestamp will be sent in
`If-Modified-Since`.

## cache.staleIfError

- Type: `number` or `boolean` or `StaleIfErrorPredicate<R, D>`
- Default: `true`

Enables cache to be returned if the response comes with an error, either by invalid status
code, network errors and etc. You can filter the type of error that should be stale by
using a predicate function.

::: warning

If the response is treated as error because of invalid status code _(like when using
[statusCheck](#cache-cachepredicate))_, and this ends up `true`, the cache will be
preserved over the "invalid" request.

So, if you want to preserve the response, you can use the below predicate:

:::

```ts
const customPredicate = (response, cache, error) => {
  // Blocks staleIfError if has a response
  return !response;

  // Note that, this still respects axios default implementation
  // and throws an error, (but it keeps the response)
};
```

Types Explanations

- `number` -> the max time (in seconds) that the cache can be reused.
- `boolean` -> `false` disables and `true` enables with infinite time.
- `function` -> a predicate that can return `number` or `boolean` as described above.

## cache.override

<Badge text="optional" type="warning"/>

- Type: `boolean`
- Default: `false`

This option bypasses the current cache and always make a new http request. This will not
delete the current cache, it will just replace the cache when the response arrives.

Unlike as `cache: false`, this will not disable the cache, it will just ignore the
pre-request cache checks before making the request. This way, all post-request options are
still available and will work as expected.
