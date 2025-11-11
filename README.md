# boxedresponse

> Rust-inspired Result ergonomics for TypeScript services that require explicit, typed error propagation.

Boxed wraps every return value in a discriminated union so downstream callers cannot forget to handle failures. The API mirrors the ergonomics you get from Rust's `Result`, while staying entirely within plain TypeScript objects (`status`, `data`, `errorType`, `message`).

## Table of Contents

- [Motivation](#motivation)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Types](#core-types)
- [Constructors & Type Guards](#constructors--type-guards)
- [Working With `BoxedResponse`](#working-with-boxedresponse)
- [Consumer Helpers](#consumer-helpers)
- [Async Utilities](#async-utilities)
- [Tuple Consumption](#tuple-consumption)
- [API Cheat Sheet](#api-cheat-sheet)
- [Best Practices](#best-practices)

## Motivation

Every backend function that returns a `BoxedResponse` MUST have its error types propagated and handled by the caller. Because the error discriminant is a string/number literal, you get end-to-end exhaustiveness checking, friendly stack traces, and a single place to encode business error contracts.

## Installation

```bash
npm install tsboxed
# or
yarn add tsboxed
# or
pnpm add tsboxed
```

The published package name is `tsboxed` even though the repository is `boxedts`.

## Quick Start

```ts
import {
  BoxedResponse,
  Err,
  Ok,
  isErr,
  retryOnBoxedError,
  consumeOrThrow,
} from "tsboxed";

type UserError = "UserNotFound" | "UserInactive" | "DbOffline";

function fetchUser(
  id: string
): BoxedResponse<{ id: string; role: string }, UserError> {
  if (id === "root") {
    return Ok({ id, role: "admin" });
  }

  return Err("user was not found", "UserNotFound");
}

function ensureActive(user: {
  id: string;
  role: string;
}): BoxedResponse<{ id: string; role: string }, UserError> {
  return user.role === "suspended"
    ? Err("user is inactive", "UserInactive")
    : Ok(user);
}

const userResponse = fetchUser("root")
  .andThen(ensureActive)
  .map((user) => ({ ...user, canAccessBilling: user.role === "admin" }))
  .mapErr((err) => ({
    errorType: "UserServiceError",
    message: `failed because ${err.errorType}`,
  }));

if (isErr(userResponse)) {
  // Narrowed to IBoxedError<"UserServiceError">
  console.error(userResponse.errorType, userResponse.message);
} else {
  console.log("user is", userResponse.data);
}

// Lift a retry strategy around flaky providers.
const retryingFetch = retryOnBoxedError({ timeoutMs: 5000 });
const eventualUser = await retryingFetch(fetchUser, (attempt, err) => {
  console.warn(`[retry ${attempt}]`, err.errorType);
});

// Throw when you truly expect success (tests, CLI tools, etc.)
const user = consumeOrThrow(eventualUser);
```

## Core Types

### `BoxedResponse<T, E>`

A discriminated union (`status: true | false`) that is either an `IBoxedSuccess<T, E>` or an `IBoxedError<E>`. Use it as the return type for every operation that can fail.

### `IBoxedError<E>` / `BoxedError<E>`

Represents a failure:

- `status: false`, `errorType: E`, optional `message`.
- Rust-like helpers: `unwrap()` and `expect()` throw, `unwrapOr`/`unwrapOrElse` fall back, `mapErr` lets you rewrite the error type/message, `orElse` allows recovery.
- `toNullable()` always returns `null`.
- Constructors default to `message = "an error occurred"` and `errorType = "UnknownError"` unless you supply your own. If you use numeric codes, pass them explicitly (e.g. `Err("boom", -111 as const)`).

### `IBoxedSuccess<T, E>` / `BoxedSuccess<T, E>`

Represents a success:

- `status: true`, `data: T`.
- Helpers mirror the error shape: `unwrap()` returns the data, `map` transforms it, `andThen` chains more `BoxedResponse`s (and can widen the error type), `toNullable()` returns the raw value, and `isOk`/`isErr` narrow at the object level.

## Constructors & Type Guards

- `Ok(data)` – lightweight helper that returns a `BoxedSuccess`.
- `Err(message?, errorType?)` – helper that returns a `BoxedError` (`message` is first on purpose so stack traces read naturally).
- `isBoxedError(response)` / `isErr(response)` – type guard for narrowing to `IBoxedError`.
- `isOk(response)` – type guard for narrowing to `IBoxedSuccess`.

Because everything carries the `status` field, you can also discriminate manually (`if (response.status)`), but the guards convey intent and keep the compiler happy when inference gets tricky.

## Working With `BoxedResponse`

The shape borrows heavily from Rust's `Result`:

- **Transforming success**: `map(fn)` maps the success payload. `andThen(fn)` expects `fn` to return another `BoxedResponse`, keeping your error pipeline intact.
- **Transforming errors**: `mapErr(fn)` receives the current error and returns either a `{ errorType, message? }` object or a bare `errorType` literal. This is the idiomatic place to translate low-level errors into domain-specific ones.
- **Fallbacks**: `orElse(fn)` lets you recover from a failure by returning a new `BoxedResponse`.
- **Unwrapping**: `unwrap()` and `expect(msg)` should only be used when you are certain about success (tests, scripts). Prefer `unwrapOr(value)` or `unwrapOrElse(fn)` in user-facing paths.
- **Nullability**: `toNullable()` converts successes to the underlying value and errors to `null`, which is convenient for React Suspense/optional rendering hooks.
- **Runtime checks**: the instance methods `isOk()` / `isErr()` mirror the standalone guards when you already hold the concrete class.

## Consumer Helpers

In addition to the instance methods, several standalone helpers live in `index.ts`:

- `consumeOrThrow(response)` – throws if the response is an error, carrying `errorType`/`message` in the exception text.
- `consumeOrNull(response)` – returns `null` instead of throwing when the response is an error.
- `consumeOrCallback(response, callback)` – routes errors into your callback while still returning the success payload untouched.
- `consumeAll(tuple, consumeFn = consumeOrThrow)` – accepts a tuple/array of `BoxedResponse`s and returns a tuple of just the success payloads, short-circuiting through the provided consumer.

```ts
const [user, plan] = consumeAll([fetchUser("root"), fetchPlan("pro")]);

// Provide your own consumer to keep errors inside the tuple.
const [userOrNull, planOrNull] = consumeAll(
  [fetchUser("123"), fetchPlan("basic")],
  consumeOrNull
);
```

## Async Utilities

Boxed ships with a couple of utilities for dealing with async operations that produce `BoxedResponse`s:

### `consumeUntilSuccess(response, interval, maxAttempts = 10)`

Polls the provided `response` reference every `interval` milliseconds until it flips to a success state or `maxAttempts` is reached. Use this when you have long-lived objects that receive updates out of band.

### `retryOnBoxedError(timeOpts?)`

Returns an async function that repeatedly invokes your async producer until it resolves to a success or a permitted error:

```ts
const retryPayments = retryOnBoxedError({
  intervalMs: 750,
  timeoutMs: 15_000,
});

const payment = await retryPayments(
  () => chargeCard(request),
  (attempt, err, fnName) => logger.warn({ attempt, err, fnName }),
  ["ValidationError"] // stop retrying and return immediately for these error codes
);
```

- `intervalMs` defaults to `1000`.
- `timeoutMs` defaults to `10000`.
- `onRetry` receives `(attempt, err, fnName)` for logging/metrics.
- `returnErrors` is a whitelist of error codes that should be returned immediately instead of being retried.

The helper ultimately returns the final `BoxedResponse`.

### `retryOrThrow(timeOpts?)`

A thin wrapper around `retryOnBoxedError` that unwraps the final success (or throws if it never succeeded). Nice for scripts/CLIs where you want retries but still prefer exceptions on failure.

## Tuple Consumption

`consumeAll` deserves its own call-out because it preserves tuple indices and infers each success type individually. This makes it ideal for `Promise.all` style flows:

```ts
const resultTuple = await Promise.all([
  fetchUser("root"),
  fetchSubscription("root"),
] as const);

const [user, subscription] = consumeAll(resultTuple);
```

## API Cheat Sheet

| Export                                                     | Description                                                                               |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `IBoxedError<E>` / `BoxedError<E>`                         | Error shape with `status: false`, `errorType`, optional `message`, and Rust-like helpers. |
| `IBoxedSuccess<T, E>` / `BoxedSuccess<T, E>`               | Success shape with `status: true`, `data`, and matching helpers.                          |
| `BoxedResponse<T, E>`                                      | Union of the two shapes used as the public return type.                                   |
| `Ok(data)` / `Err(message?, errorType?)`                   | Convenience constructors for success/error values.                                        |
| `isBoxedError`, `isErr`, `isOk`                            | Type guards that refine a `BoxedResponse`.                                                |
| `consumeOrThrow`, `consumeOrNull`, `consumeOrCallback`     | Helpers for destructing a response according to your preferred error strategy.            |
| `consumeUntilSuccess`, `retryOnBoxedError`, `retryOrThrow` | Async utilities for polling or retrying until you get a success.                          |
| `consumeAll`                                               | Tuple-aware consumer that returns only the success payloads.                              |

## Best Practices

- Always bubble up the concrete error type string/number from the function you call—it's part of the contract.
- Use `mapErr` for translating infrastructure errors (`DbOffline`) into domain errors (`CheckoutUnavailable`) at the edges of your service.
- Prefer `andThen` when chaining multiple operations so each step can return its own `BoxedResponse` and failure short-circuits immediately.
- Reach for `retryOnBoxedError` instead of manual loops whenever you talk to flaky providers; the helper already handles timeouts, callback hooks, and stop conditions.
- Reserve `unwrap`/`expect` for places where throwing is intended (tests, scripts). Inside request handlers prefer `consumeOrThrow`, `consumeOrNull`, or explicit guards so you can tailor HTTP responses.
- Type-guard with `isErr`/`isOk` (or the instance methods) before touching `.data` or `.message` to keep the compiler honest.

---

If you need an example that is not covered here, check the exported members inside `index.ts`—everything is documented above so you can trace behavior directly from the source.
