# Porting to the Buffer.from/Buffer.alloc API

## Variant 1

Drop support for Node.js ≤ 4.4.x and 5.0.0 — 5.9.x.

This is the recommended solution nowdays that would imply only minimal churn.

Note that `Buffer.alloc()` is also _faster_ on the current Node.js versions than
`new Buffer(size).fill(0)` which you need to ensure zero-filling otherwise.

Enabling eslint rule [no-buffer-constructor](https://eslint.org/docs/rules/no-buffer-constructor)
or
[node/no-deprecated-api](https://github.com/mysticatea/eslint-plugin-node/blob/master/docs/rules/no-deprecated-api.md)
is recommended to avoid accidential unsafe Buffer API usage.

_If you currently support those Node.js versions and dropping them would be a semver-major change
for you, or if you support older branches of your packages, consider using [Variant 2](#variant-2)
or [Variant 3](#variant-3) on older branches, so people using those older branches will also receive
the fix. That way, you will eradicate potential issues caused by unguarded Buffer API usage and
your users will not observe a runtime deprecation warning when running your code on Node.js 10._

## Variant 2

Utilize [safer-buffer](https://www.npmjs.com/package/safer-buffer) to polyfill to support older
Node.js versions.

Exacly the same as [Variant 1](#variant-1), but with a polyfill
`const Buffer = require('safer-buffer').Buffer` in all files where you use the new `Buffer` api.

Make sure that you do not use old `new Buffer` API — in any files where the line above is added,
using old `new Buffer()` API will _throw_. It will be easy to notice that in CI, though.

_Alternatively, you could use [safe-buffer](https://www.npmjs.com/package/safe-buffer) — it also
provides a polyfill, but takes a different approach which has
[it's drawbacks](https://github.com/chalker/safer-buffer#why-not-safe-buffer). It will allow you
to also use the older `new Buffer()` API in your code, though — but that's arguably a benefit, as
it is problematic, can cause issues in your code, and will start emitting runtime deprecation
warnings starting with Node.js 10._

Note that in either of those two it is important that you also remove all calls to the old Buffer
API manually — just throwing in `safe-buffer` doesn't fix the problem by itself, it just provides
a polyfill for the new API. I have seen people doing that mistake.

Enabling eslint rule [no-buffer-constructor](https://eslint.org/docs/rules/no-buffer-constructor)
or
[node/no-deprecated-api](https://github.com/mysticatea/eslint-plugin-node/blob/master/docs/rules/no-deprecated-api.md)
is recommended.

## Variant 3 — manual detection, with safeguards

Useful if you create Buffer instances in a minimal number of places (e.g. one) or have your own
wrapper around them.

### Buffer(0)

This special case for creating empty buffers can be safely replaced with `Buffer.concat([])`, which
returns the same result all the way down to Node.js 0.8.x.

### Buffer(notNumber)

```js
var buf;
if (Buffer.from && Buffer.from !== Uint8Array.from) {
  buf = Buffer.from(notNumber, encoding);
} else {
  if (typeof notNumber === 'number')
    throw new Error('The "size" argument must be of type number.');
  buf = new Buffer(notNumber, encoding);
}
```

`encoding` is optional.

The `typeof notNumber` check is not required in cases if `notNumber` argument is hardcoded (e.g.
literal `"abc"` or `[0,1,2]`), but is required if it's passed from somewhere else.

Note that the type-check before `new Buffer` is required (for cases when `notNumber` argument is not
hard-coded) and _is not caused by the deprecation of Buffer constructor_ — it's exactly _why_ the
Buffer constructor is deprecated. Ecosystem packages lacking this type-check caused numereous
security issues — situations when unsanitized user input could end up in the `Buffer(arg)` create
problems ranging from DoS to leaking sensitive information to the attacker from the process memory.

Also note that using TypeScript does not fix this problem for you — when libs written in
`TypeScript` are used from JS, or when user input ends up there — it behaves exactly as pure JS, as
all type checks are translation-time only and are not present in the actual JS code which TS
compiles to.

### Buffer(number)

For Node.js 0.10.x (and below) support:

```js
var buf;
if (Buffer.alloc) {
  buf = Buffer.alloc(number);
} else {
  buf = new Buffer(number);
  buf.fill(0);
}
```

Otherwise (Node.js ≥ 0.12.x):

```js
const buf = Buffer.alloc ? Buffer.alloc(number) : new Buffer(number).fill(0);
```

## Regarding Buffer.allocUnsafe

Be extra cautious when using `Buffer.allocUnsafe`:
 * Don't use it if you don't have a good reason to
   * e.g. you probably won't ever see a performance difference for small buffers, in fact, those
     might be even faster with `Buffer.alloc()`,
   * if your code is not in the hot code path — you also probably won't notice a difference,
   * keep in mind that zero-filling minimizes the potential risks.
 * If you use it, make sure that you never return the buffer in a partially-filled state,
   * if you are writing to it sequentially — always truncate it to the actuall written length

Errors in handling buffers allocated with `Buffer.allocUnsafe` could result in various issues,
ranged from undefined behaviour of your code to sensitive data (user input, passwords, certs)
leaking to the remote attacker.

_Note that the same applies to `new Buffer` usage without zero-filling, depending on the Node.js
version (and lacking type checks also adds DoS to the list of potential problems)._
