---
name: building-reactables
description: Use when building or reviewing state management with the Reactables library (@reactables/core, @reactables/react, @reactables/forms) — creating a reactable, handling async/API calls, side effects, debounce/cancellation, composing or sharing state across reactables, or wiring one into a React component.
---

# Building Reactables

## Overview

Reactables is a framework-agnostic, RxJS-based state manager. The unit is a **Reactable**: a tuple `[state$, actions, actions$]` produced by `RxBuilder`.

- `state$` — `Observable` emitting current state (ReplaySubject, seeded).
- `actions` — dictionary of methods that dispatch updates, plus `destroy()`.
- `actions$` — `Observable` of every action; carries `.types` (action-type constants) and `.ofTypes([...])` (filtered stream).

Core principle: **reducers are pure and synchronous; everything async or side-effecting is an `effect`.** Actions flow one direction and reactables compose into a DAG.

## Quick Reference

| Need | Use |
|------|-----|
| Create a reactable | `RxBuilder({ initialState, reducers, sources?, debug?, name? })` |
| Sync state change | reducer fn `(state, action) => newState` (pure, immutable) |
| Async / API / side effect | reducer **config** `{ reducer, effects: [ (action$) => Observable<Action> ] }` |
| Cancel stale in-flight request | `switchMap` inside the effect |
| Independent per-item async (per row/id) | scoped effect: `effects: (payload) => ({ key: payload.id, effects: [...] })` |
| React to another reactable | pass its `actions$.ofTypes([...])` (mapped) into a consumer's `sources` |
| Fire an action on creation/mount | `sources: [of({ type: 'fetch' })]` |
| Merge reactables into one | `combine({ auth: rxAuth, profile: rxProfile })` |
| Bind to a React component | `useReactable(factory, ...args)` → `[state, actions, state$, actions$]` |
| Log actions + state diffs | `debug: true` |

## Core Pattern

Wrap `RxBuilder` in a named factory. Reducers return **new** state — never mutate.

```typescript
import { RxBuilder, Action } from '@reactables/core';

export const RxCounter = () =>
  RxBuilder({
    name: 'rxCounter', // shows in debug logs
    initialState: { count: 0 },
    reducers: {
      increment: (state) => ({ count: state.count + 1 }), // pure, no mutation
      reset: () => ({ count: 0 }),
      add: (state, { payload }: Action<number>) => ({ count: state.count + payload }),
    },
  });
```

Consume: `const [state$, actions, actions$] = RxCounter(); actions.add(5);`

## Async & Side Effects

Async work goes in `effects`, which replay the triggering action through an RxJS pipe and map the result to **new actions** dispatched back to the store. Never call an API or mutate outside state in a reducer.

```typescript
import { RxBuilder, Action } from '@reactables/core';
import { Observable, from, of } from 'rxjs';
import { switchMap, map, catchError } from 'rxjs/operators';

export const RxFetchUser = (api: (id: string) => Promise<User>) =>
  RxBuilder({
    initialState: { loading: false, user: null as User | null, error: null as string | null },
    reducers: {
      fetch: {
        reducer: (state) => ({ ...state, loading: true, error: null }),
        effects: [
          (fetch$: Observable<Action<string>>) =>
            fetch$.pipe(
              switchMap(({ payload: id }) =>        // switchMap cancels a stale request
                from(api(id)).pipe(
                  map((user) => ({ type: 'fetchSuccess', payload: user })),
                  catchError(() => of({ type: 'fetchError', payload: 'Failed' })), // INSIDE switchMap
                ),
              ),
            ),
        ],
      },
      fetchSuccess: (s, { payload }: Action<User>) => ({ ...s, loading: false, user: payload }),
      fetchError: (s, { payload }: Action<string>) => ({ ...s, loading: false, error: payload }),
    },
  });
```

Rules:
- **`switchMap`** cancels the previous pending call — right default for search/autocomplete/latest-wins. Use `concatMap`/`exhaustMap`/`mergeMap` deliberately when you need ordering/ignore-while-busy/parallel.
- Put **`catchError` inside** the inner (per-request) observable so one failure doesn't tear down the long-lived effect stream.
- **Scoped `key`**: make `effects` a function returning `{ key, effects }` when each item runs independently (e.g. updating rows by id) so each key gets its own effect stream and its own `switchMap`. Omit the key for a single shared stream (e.g. one global debounced search).

## Composition Over Nested Subscribes

To make one reactable react to another, feed the source's actions into the consumer's `sources` — do **not** `subscribe` inside a reducer/effect and dispatch manually. Then `combine` for a single state tree.

```typescript
import { combine } from '@reactables/core';
import { map } from 'rxjs/operators';

export const RxApp = () => {
  const rxAuth = RxAuth();
  const [, , authActions$] = rxAuth;

  const fetchOnLogin$ = authActions$
    .ofTypes([authActions$.types.loginSuccess])
    .pipe(map(({ payload }) => ({ type: 'fetchProfile', payload })));

  const rxProfile = RxProfile({ sources: [fetchOnLogin$] });

  return combine({ auth: rxAuth, profile: rxProfile }); // state: { auth, profile }
};
```

## React

Use `useReactable` — it owns creation, subscription, and teardown (`destroy()`). Don't recreate the reactable inline or subscribe to `state$` yourself just to render.

```tsx
const Search = ({ api }: { api: SearchApi }) => {
  const [state, actions] = useReactable(RxSearch, api);
  if (!state) return null; // guard: state can be undefined on first render
  return <input value={state.term} onChange={(e) => actions.search(e.target.value)} />;
};
```

Subscribe to the returned `state$` / `actions$` only for **side effects**, inside `useEffect`, and unsubscribe on cleanup. For shared/global state, create the reactable once and pass it via Context.

## Testing

Reactables test independently of any UI via RxJS marble testing: dispatch actions on a cold stream, assert the emitted state sequence.

```typescript
testScheduler.run(({ expectObservable, cold }) => {
  const [state$, { increment, reset }] = RxCounter();
  cold('--b-c', { b: increment, c: reset }).subscribe((a) => a());
  expectObservable(state$).toBe('a-b-c', { a: { count: 0 }, b: { count: 1 }, c: { count: 0 } });
});
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| API call / `await` / side effect inside a reducer | Move it to `effects`; reducer stays pure & sync |
| Mutating state (`state.count++`, `push`) | Return a new object/array (`{ ...state }`, `concat`) |
| `mergeMap` for search/latest-wins | `switchMap` to cancel stale requests |
| `catchError` on the outer effect stream | Nest it inside the per-request observable so the stream survives |
| Nested `subscribe` + manual dispatch to link reactables | Wire via `sources` + `ofTypes`, then `combine` |
| Recreating the reactable each render / manual `state$` subscribe to render | `useReactable`; subscribe only for side effects in `useEffect` |
| Forgetting teardown (leaked subscriptions) | `useReactable` calls `destroy()`; standalone, call `actions.destroy()` |
| Extending behavior by forking a factory | Spread extra `reducers` into the base factory's config |
| Can't tell what changed | Set `debug: true` and a `name` |
```
