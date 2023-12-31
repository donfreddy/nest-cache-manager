# @dfreddy/cache-manager2 
[![codecov](https://codecov.io/gh/node-cache-manager/node-cache-manager/branch/master/graph/badge.svg?token=ZV3G5IFigq)](https://codecov.io/gh/nest-cache-manager/nest-cache-manager)
[![tests](https://github.com/node-cache-manager/node-cache-manager/actions/workflows/test.yml/badge.svg)](https://github.com/donfreddy/nest-cache-manager/actions/workflows/test.yml)
[![license](https://img.shields.io/github/license/@dfreddy/cache-manager2/@dfreddy/cache-manager2)](https://github.com/donfreddy/nest-cache-manager/blob/main/LICENSE)
[![npm](https://img.shields.io/npm/dm/nest-cache-manager)](https://npmjs.com/package/@dfreddy/cache-manager2)
![npm](https://img.shields.io/npm/v/@dfreddy/cache-manager2r)

# Flexible NestJS cache module

A cache module for nestjs that allows easy wrapping of functions in cache, tiered caches, and a consistent interface.

## Features

- Made with Typescript and compatible with [ESModules](https://nodejs.org/docs/latest-v14.x/api/esm.html)
- Easy way to wrap any function in cache.
- Tiered caches -- data gets stored in each cache and fetched from the highest.
  priority cache(s) first.
- Use any cache you want, as long as it has the same API.
- 100% test coverage via [vitest](https://github.com/vitest-dev/vitest).

## Installation

    pnpm install @dfreddy/cache-manager2

## Usage Examples

### Single Store

```typescript
import { caching } from '@dfreddy/cache-manager2';

const memoryCache = await caching('memory', {
  max: 100,
  ttl: 10 * 1000 /*milliseconds*/,
});

const ttl = 5 * 1000; /*milliseconds*/
await memoryCache.set('foo', 'bar', ttl);

console.log(await memoryCache.get('foo'));
// >> "bar"

await memoryCache.del('foo');

console.log(await memoryCache.get('foo'));
// >> undefined

const getUser = (id: string) => new Promise.resolve({ id: id, name: 'Bob' });

const userId = 123;
const key = 'user_' + userId;

console.log(await memoryCache.wrap(key, () => getUser(userId), ttl));
// >> { id: 123, name: 'Bob' }
```

See unit tests in [`test/caching.test.ts`](./test/caching.test.ts) for more information.

#### Example setting/getting several keys with mset() and mget()

```typescript
await memoryCache.store.mset(
  [
    ['foo', 'bar'],
    ['foo2', 'bar2'],
  ],
  ttl,
);

console.log(await memoryCache.store.mget('foo', 'foo2'));
// >> ['bar', 'bar2']

// Delete keys with mdel() passing arguments...
await memoryCache.store.mdel('foo', 'foo2');
```

#### [Example Express App Usage](./examples/express/src/index.mts)

#### Custom Stores

You can use your own custom store by creating one with the same API as the built-in memory stores.

- [Example Custom Store lru-cache](./src/stores/memory.ts)
- [Example Custom Store redis](https://github.com/node-cache-manager/node-cache-manager-redis-yet)
- [Example Custom Store ioredis](https://github.com/node-cache-manager/node-cache-manager-ioredis-yet)

### Multi-Store

```typescript
import { multiCaching } from '@dfreddy/cache-manager2';

const multiCache = multiCaching([memoryCache, someOtherCache]);
const userId2 = 456;
const key2 = 'user_' + userId;
const ttl = 5;

// Sets in all caches.
await multiCache.set('foo2', 'bar2', ttl);

// Fetches from highest priority cache that has the key.
console.log(await multiCache.get('foo2'));
// >> "bar2"

// Delete from all caches
await multiCache.del('foo2');

// Sets multiple keys in all caches.
// You can pass as many key, value tuples as you want
await multiCache.mset(
  [
    ['foo', 'bar'],
    ['foo2', 'bar2'],
  ],
  ttl
);

// mget() fetches from highest priority cache.
// If the first cache does not return all the keys,
// the next cache is fetched with the keys that were not found.
// This is done recursively until either:
// - all have been found
// - all caches has been fetched
console.log(await multiCache.mget('key', 'key2'));
// >> ['bar', 'bar2']

// Delete keys with mdel() passing arguments...
await multiCache.mdel('foo', 'foo2');
```

See unit tests in [`test/multi-caching.test.ts`](./test/multi-caching.test.ts) for more information.

### Refresh cache keys in background

Both the `caching` and `multicaching` modules support a mechanism to refresh expiring cache keys in background when using the `wrap` function.  
This is done by adding a `refreshThreshold` attribute while creating the caching store.

If `refreshThreshold` is set and after retrieving a value from cache the TTL will be checked.  
If the remaining TTL is less than `refreshThreshold`, the system will update the value asynchronously,  
following same rules as standard fetching. In the meantime, the system will return the old value until expiration.

NOTES:

* In case of multicaching, the store that will be checked for refresh is the one where the key will be found first (highest priority).
* If the threshold is low and the worker function is slow, the key may expire and you may encounter a racing condition with updating values.
* The background refresh mechanism currently does not support providing multiple keys to `wrap` function.
* If no `ttl` is set for the key, the refresh mechanism will not be triggered. For redis, the `ttl` is set to -1 by default.

For example, pass the refreshThreshold to `caching` like this:

```typescript
const memoryCache = await caching('memory', {
  max: 100,
  ttl: 10 * 1000 /*milliseconds*/,
  refreshThreshold: 3 * 1000 /*milliseconds*/,
});
```

When a value will be retrieved from Redis with a TTL minor than 3sec, the value will be updated in the background.

## Contribute

If you would like to contribute to the project, please fork it and send us a pull request. Please add tests
for any new features or bug fixes.

## License

node-cache-manager is licensed under the [MIT license](./LICENSE).
