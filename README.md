# Disposable AsyncContext.Variable

Champions:

- Chengzhong Wu ([@legendecas](https://github.com/legendecas))
- Luca Casonato ([@lucacasonato](https://github.com/lucacasonato))
- snek ([@devsnek](https://github.com/devsnek))

# Motivation

[AsyncContext][] enforces mutation with `AsyncContext.Variable` by a function scope API
`AsyncContext.Variable.prototype.run`. This requires any `Variable` value mutations to
be performed within a new function scope.

Modifications to `Variable` values are propagated to its subtasks. This `.run`
scope enforcement prevents any modifications to be visible to its caller
function scope, consequently been propagated to tasks created in sibling
function calls.

For instance, this prevents a mutation in an event listener be accidentally
leaked out to its dispatcher:

```js
const asyncVar = new AsyncContext.Variable();
eventTarget.addEventListener('click', function firstListener() {
  asyncVar.run('first', () => {
    //...
  });
  // 'first' is not visible outside of firstListener
});

eventTarget.addEventListener('click', function secondListener() {
  asyncVar.run('second', () => {
    //...
  });
});

eventTarget.dispatch(new Event('click'));
// 'first' is not visible to the secondListener.
```

## Usages of run

The `run` pattern can already handles many existing usage pattern well that
involves function calls, like:

- Event handlers,
- Middleware.

For example, an event handler can be easily refactored to use `.run(value, fn)`
by wrapping:

```js
function handler(event) {
  ...
}

button.addEventListener("click", handler);
// ... replace it with ...
button.addEventListener("click", event => {
  asyncVar.run(createSpan(), handler, event);
});
```

Or, on Node.js server applications, where middlewares are common to use:

```js
const middlewares = [];
function use(fn) {
  middlewares.push(fn)
}

async function runMiddlewares(req, res) {
  function next(i) {
    if (i === middlewares.length) {
      return;
    }
    return middlewares[i](req, res, next.bind(i++));
  }

  return next(0);
}
```

A tracing library like OpenTelemetry can instrument it with a middleware
wrapper like:

```js
async function otelMiddleware(req, res, next) {
  const w3cTraceContext = extractW3CHeaders(req);
  const span = createSpan(w3cTraceContext);
  try {
    await asyncVar.run(span, next);
  } catch (e) {
    span.setError(e);
  } finally {
    span.end();
  }
}
```

### Limitation of run

The enforcement of mutation scopes can reduce the chance that the mutation is
exposed to the parent scope in unexpected way, but it also increases the bar to
use the feature or migrate existing code to adopt the feature.

For example, given a snippet of code:

```js
function *gen() {
  yield computeResult();
  yield computeResult2();
}
```

If we want to scope the `computeResult` and `computeResult2` calls with a new
AsyncContext value, it needs non-trivial refactor:

```js
const asyncVar = new AsyncContext.Context();

function *gen() {
  const span = createSpan();
  yield asyncVar.run(span, () => computeResult());
  yield asyncVar.run(span, () => computeResult2());
  // ...or
  yield* asyncVar.run(span, function *() {
    yield computeResult();
    yield computeResult2();
  });
}
```

`.run(val, fn)` creates a new function body. The new function environment
is not equivalent to the outer environment and can not trivially share code
fragments between them. Additionally, `break`/`continue`/`return` can not be
refactored naively.

It will be more intuitive to be able to insert a new line and without refactor
existing code snippet.

```diff
 const asyncVar = new AsyncContext.Variable();

 function *gen() {
+  using _ = asyncVar.withValue(createSpan(i));
   yield computeResult(i);
   yield computeResult2(i);
 }
```

# The proposal

To address the limitation of `.run`, but still with the advantage of enforced
scope of mutation, `AsyncContext.Variable` can implement the well-known symbol
interface [`@@dispose`][] by the `using` declaration (and potentially enforcing
the `using` declaration with [`@@enter`][]).

> `AsyncContext.Snapshot` is intentionally excluded from this feature, as it
> affects the AsyncContext mappings including other `AsyncContext.Variable` instances.

```js
const asyncVar = new AsyncContext.Variable();

{
  using _ = asyncVar.withValue("main");
  new AsyncContext.Snapshot() // snapshot 0
  console.log(asyncVar.get()); // => "main"
}

{
  using _ = asyncVar.withValue("value-1");
  new AsyncContext.Snapshot() // snapshot 1
  Promise.resolve()
    .then(() => { // continuation 1
      console.log(asyncVar.get()); // => 'value-1'
    })
}

{
  using _ = asyncVar.withValue("value-2");
  new AsyncContext.Snapshot() // snapshot 2
  Promise.resolve()
    .then(() => { // continuation 2
      console.log(asyncVar.get()); // => 'value-2'
    })
}
```

Notably, the `withValue` does not mutate the AsyncContext mapping in place, just
like `.run` does. It creates a new mapping for the subsequent scope. The value
mapping is equivalent to:

```
⌌-----------⌍ snapshot 0
|   'main'  |
⌎-----------⌏
      |
⌌-----------⌍ snapshot 1
| 'value-1' |  <---- the continuation 1
⌎-----------⌏
      |
⌌-----------⌍ snapshot 2
| 'value-2' |  <---- the continuation 2
⌎-----------⌏
```

Each `@@enter` operation creates an AsyncContext mapping with the new value, avoids
any mutation to existing AsyncContext mapping where the current
`AsyncContext.Variable` value was captured.

This trait is important with both `run` and `withValue` because mutations to an
`AsyncContext.Variable` must not mutate prior `AsyncContext.Snapshot`s.

However, the well-known symbol `@@dispose` and `@@enter` is not bound to the
`using` declaration syntax, and they can be invoked manually. This is a
by-design feature allowing advanced user-land extension, like OpenTelemetry's
example in the next section.

This is an extension to the proposed `run` semantics.

## Use cases

### Web frameworks

> TODO: Let's refer to web frameworks for examples.

### Tracing

The set semantic allows instrumenting existing codes without nesting them in a
new function scope and reducing the refactoring work:

```js
async function doAnotherWork() {
  // defer work to next promise tick.
  await 0;
  using span = tracer.startActiveSpan("anotherWork");
  console.log("doing another work");
  // the span is closed when it's out of scope
}

async function doWork() {
  using parent = tracer.startActiveSpan("parent");
  // do some work that 'parent' tracks
  console.log("doing some work...");
  await doSomeWork();
  // Create a nested span to track nested work
  {
    using child = tracer.startActiveSpan("child");
    // do some work that 'child' tracks
    console.log("doing some nested child work...")
    // the nested span is closed when it's out of scope
  }
  await doAnotherWork();
  // This parent span is also closed when it goes out of scope
}
```

> This example is adapted from the OpenTelemetry Python example.
> https://opentelemetry.io/docs/languages/python/instrumentation/#creating-spans

Each `tracer.startActiveSpan` invocation retrieves the parent span from its
own `AsyncContext.Variable` instance and create span as a child, and set the
child span as the current value of the `AsyncContext.Variable` instance:

```js
class Tracer {
  #var = new AsyncContext.Variable();

  startActiveSpan(name) {
    let scope;
    const span = {
      name,
      parent: this.#var.get(),
      [Symbol.enter]: () => {
        scope = this.#var.withValue(span)[Symbol.enter]();
        return span;
      },
      [Symbol.dispose]: () => {
        scope[Symbol.dispose]();
      },
    };
    return span;
  }
}
```

The semantic that doesn't mutate existing AsyncContext mapping is crucial to the
`startAsCurrentSpan` example here, as it allows `doAnotherWork` to be a child
span of the `"parent"` instead of `"child"`, shown as graph below:

```
⌌----------⌍
| 'parent' |
⌎----------⌏
  |   ⌌-----------------⌍
  |---| 'doSomeWork'    |
  |   ⌎-----------------⌏
  |   ⌌---------⌍
  |---| 'child' |
  |   ⌎---------⌏
  |   ⌌-----------------⌍
  |---| 'doAnotherWork' |
  |   ⌎-----------------⌏
```

[`@@dispose`]: https://github.com/tc39/proposal-explicit-resource-management?tab=readme-ov-file#using-declarations
[`@@enter`]: https://github.com/tc39/proposal-using-enforcement?tab=readme-ov-file#proposed-solution
[`ContinuationVariable`]: ./CONTINUATION.md
[AsyncContext]: https://github.com/tc39/proposal-async-context
