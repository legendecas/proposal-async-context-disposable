# Disposable AsyncContext.Variable

Champions:

- Chengzhong Wu ([@legendecas](https://github.com/legendecas))
- Luca Casonato ([@lucacasonato](https://github.com/lucacasonato))
- snek ([@devsnek](https://github.com/devsnek))

# Motivation

[AsyncContext][AsyncContext] enforces mutation with `AsyncContext.Variable` by a
function scope API `AsyncContext.Variable.prototype.run`. This requires any
`Variable` value mutations to be performed within a new function scope.

Modifications to `Variable` values are propagated to its subtasks. This `.run`
scope enforcement prevents any modifications to be visible to its caller
function scope, consequently been propagated to tasks created in sibling
function calls.

For instance, this prevents a mutation in an event listener be accidentally
leaked out to its dispatcher:

```js
const asyncVar = new AsyncContext.Variable();
eventTarget.addEventListener("click", function firstListener() {
  asyncVar.run("first", () => {
    //...
  });
  // 'first' is not visible outside of firstListener
});

eventTarget.addEventListener("click", function secondListener() {
  asyncVar.run("second", () => {
    //...
  });
});

eventTarget.dispatch(new Event("click"));
// 'first' is not visible to the secondListener.
```

## Usages of `run`

The `run` pattern can already handle many existing usage patterns well that
involve function calls, like:

- Event handlers,
- or Middleware.

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
  middlewares.push(fn);
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

A tracing library like OpenTelemetry can instrument it with a middleware wrapper
like:

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

### Limitations of `run`

The enforcement of mutation scopes can reduce the chance that the mutation is
exposed to the parent scope in unexpected way, but it also increases the bar to
use the feature or migrate existing code to adopt the feature.

For example, given a snippet of code:

```js
function* gen() {
  yield computeResult();
  yield computeResult2();
}
```

If we want to scope the `computeResult` and `computeResult2` calls with a new
AsyncContext value, it needs non-trivial refactor:

```js
const asyncVar = new AsyncContext.Context();

function* gen() {
  const span = createSpan();
  yield asyncVar.run(span, () => computeResult());
  yield asyncVar.run(span, () => computeResult2());
  // ...or
  yield* asyncVar.run(span, function* () {
    yield computeResult();
    yield computeResult2();
  });
}
```

`.run(val, fn)` creates a new function body. The new function environment is not
equivalent to the outer environment and can not trivially share code fragments
between them. Additionally, `break`/`continue`/`return` can not be refactored
naively.

It would be more intuitive to be able to insert a single line of code to scope
the `computeResult` and `computeResult2` calls with a new AsyncContext value
without needing to refactor the existing code.

```diff
 const asyncVar = new AsyncContext.Variable();

 function *gen() {
+  using _ = asyncVar.withValue(createSpan(i));
   yield computeResult(i);
   yield computeResult2(i);
 }
```

# Goal

We are looking for a way to bind a `Variable` value to a lexical scope, without
having to create a new lexical scope in the form of a function.

We are also looking to enable non-`Variable` objects that internally contain
`AsyncContext.Variable` instances to still be able to use the same lexical
binding for their internal `AsyncContext.Variable` instances, without exposing
the `AsyncContext.Variable` instance to the user.

We are not necessarily looking to create a general purpose `enter`/`exit` API
for async context that could arbitrarily interleave variable scopes. We have
heard from implementers that doing so would be very challenging to implement
performantly (see [#3][issue-3]). After a review of many current ecosystem uses
of `AsyncLocalStorage` in Node, we are relatively confident that the majority of
use cases that have used `als.enterWith()` in Node can either switch to a
`using` based API as proposed here, or the `AsyncContext.Variable#run()` API.

> If you think you have use-cases that require an "unsafe" general purpose
> `enter`/`exit` API, please file an issue to discuss. We are interested to
> learn about them and see how we can accommodate these use-cases.

# Proposals

We have not settled on any solution, but are currently exploring the following
three options:

- Using `using` for lexical binding, by adding `@@dispose` and `@@enter` to
  `AsyncContext.Variable`
- Using `using` for lexical binding, enforcing the mutation to only exist in the
  lexical scope that `using` is in, by automatically cleaning up after manually
  entering in `Symbol.enter`
- Using `using` for lexical binding, with a special sub-classable class that
  integrates with `using` directly to automatically clean up

Each of these options has its own trade-offs, but they all share the same
semantics of mutating the `AsyncContext.Variable` value in a lexical scope. Some
of these have side-effects that would allow manually entering and exiting a
scope out of sync with lexical scoping. The options are discussed in more detail
below.

## Proposal A: manually callable `@@dispose` and `@@enter`

`AsyncContext.Variable` would get a `withValue` method that returns an object
that implements the well-known symbol interface [`@@dispose`][`@@dispose`] and
[`@@enter`][`@@enter`]. This would allow the `using` declaration to be used to
bind the value of the `AsyncContext.Variable` to a lexical scope. The
`@@dispose` method would be called when the `using` declaration goes out of
scope.

The `@@enter` method would enter a new async context scope with the value of the
`AsyncContext.Variable` set to the value passed to `withValue`. The `@@dispose`
method would exit the async context scope and restore the previous value of the
`AsyncContext.Variable`.

> `AsyncContext.Snapshot` is intentionally excluded from this feature, as it
> affects the AsyncContext mappings including other `AsyncContext.Variable`
> instances.

```js
const asyncVar = new AsyncContext.Variable();

{
  using _ = asyncVar.withValue("main");
  new AsyncContext.Snapshot(); // snapshot 0
  console.log(asyncVar.get()); // => "main"
}

{
  using _ = asyncVar.withValue("value-1");
  new AsyncContext.Snapshot(); // snapshot 1
  Promise.resolve()
    .then(() => { // continuation 1
      console.log(asyncVar.get()); // => 'value-1'
    });
}

{
  using _ = asyncVar.withValue("value-2");
  new AsyncContext.Snapshot(); // snapshot 2
  Promise.resolve()
    .then(() => { // continuation 2
      console.log(asyncVar.get()); // => 'value-2'
    });
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

Each `@@enter` operation creates an AsyncContext mapping with the new value,
avoids any mutation to existing AsyncContext mapping where the current
`AsyncContext.Variable` value was captured.

This trait is important with both `run` and `withValue` because mutations to an
`AsyncContext.Variable` must not mutate prior `AsyncContext.Snapshot`s.

This does have a downside though: the well-known symbol `@@dispose` and
`@@enter` are not bound to the `using` declaration syntax, so they can be
invoked manually. This can lead to a situation where a user could manually enter
and exit a scope out of sync with the lexical scoping (see [#2][issue-2]).
Unless mitigations for this are added, this could result in a feature that would
not be implementable without performance overhead (see [#3][issue-3]).

Additional mitigations to ensure that the `@@dispose` and `@@enter` methods are
not used with `DisposableStack` would have to be added, because
`DisposableStack` does not enforce binding to a lexical scope.

## Proposal B: enforced disposal through automatic enter tracking

As a mitigation to the above issue of manually entering and exiting a scope out
of sync with the lexical scoping, an alternative proposal is to specialize the
handling of async context variables in `using` declarations as follows:

- In the `using` machinery, before calling the `@@enter` method, a global flag
  would be set that indicates that the `@@enter` method is being called from a
  `using` declaration.
- The `@@enter` method implementation would check if the global flag is set. If
  it is not, an error would be thrown. If it is set, the `@@enter` method would
  capture the async context snapshot and set it into a global variable that is
  later read from the `using` machinery. The `@@enter` method would then create
  a new async context scope with the new value, and enter it.
- In the `using` machinery, after the `@@enter` method returns, the global flag
  would be cleared, and the value of the value of the global snapshot variable
  would be captured.

- In the `using` machinery, once the `using` declaration lexical scope closes,
  as usual the `@@dispose` method is called. Once the `@@dispose` method
  returns, the snapshot captured during enter is restored inside the `using`
  machinery.

This proposal does not enable manual entering and exiting of the async context
scope, and it requires the `@@enter` method to be called from a `using`
declaration. Binding to a lexical scope is enforced, just like with `run`.

The proposal does add some additional complexity to the `using` machinery.

## Proposal C: enforced disposal through a specialized class

As an alternative mitigation to the above issue of manually entering and exiting
a scope out of sync with the lexical scoping, an alternative proposal is to
specialize the handling of async context variables in `using` declarations as
follows:

// TODO: snek

# Use cases

## Tracing

In tracing systems like OpenTelemetry, an `AsyncContext.Variable` is used to
keep track of the currently active span. This is used to create child spans, and
to manage context propagation without having to manually pass the span
information around through the entire application and its dependencies.

Because the `AsyncContext.Variable` is used to keep track of the currently
active span, `.run` must be used to create a new async context scope for the
active span. This is inconvenient as every time a new span is created, a new
function scope must be created. This is especially inconvenient for spans inside
of loops or generators, because `break`/`continue`/`return`/`yield` statements
do not work anymore when wrapped in a new function scope.

Currently, this means a lot of boilerplate code is needed to create spans and to
manage the async context:

<table>
<thead>
<tr>
  <th>Without instrumentation</th>
  <th>With instrumentation</th>
</tr>
</thead>
<tbody>
<tr>
  <td>

```js
async function doAnotherWork() {
  // defer work to next promise tick.
  await 0;
  console.log("doing another work");
}

async function* doGeneratedWork() {
  console.log("doing some work...");
  yield 1;
  yield 2;
  yield 3;
}

async function doWork() {
  // do some work that 'parent' tracks
  console.log("doing some work...");
  await doAnotherWork();
  console.log("doing some nested child work...");
  // Call a generator function
  for await (const work of doGeneratedWork()) {
    console.log("did work ", work);
  }
}
```

</td>
  <td>

```js
async function doAnotherWork() {
  // defer work to next promise tick.
  await 0;
  const span = tracer.startSpan("anotherWork");
  return span.run(async () => {
    console.log("doing another work");
    // the span is closed when it's out of scope
  });
}

async function* doGeneratedWork() {
  const span = tracer.startSpan("generatedWork");
  yield* span.run(async function* () {
    console.log("doing some work...");
    yield 1;
    yield 2;
    yield 3;
  });
}

async function doWork() {
  const parent = tracer.startSpan("doWork");
  return parent.run(async () => {
    // do some work that 'parent' tracks
    console.log("doing some work...");
    await doAnotherWork();
    // Create a nested span to track nested work
    const child = tracer.startSpan("child");
    await child.run(async () => {
      // do some work that 'child' tracks
      console.log("doing some nested child work...");
    });
    // Call a generator function
    for await (const work of doGeneratedWork()) {
      console.log("did work ", work);
    }
  });
}
```

</td>
</tr>
</tbody>
</table>

If integrated with `using`, adding tracing to existing code would involve
significantly less refactoring, and would look much more intuitive:

```js
async function doAnotherWork() {
  // defer work to next promise tick.
  await 0;
  using span = tracer.startActiveSpan("anotherWork");
  console.log("doing another work");
  // the span is closed when it's out of scope
}

async function* doGeneratedWork() {
  using span = tracer.startActiveSpan("generatedWork");
  console.log("doing some work...");
  yield 1;
  yield 2;
  yield 3;
  // the span is closed when it's out of scope
}

async function doWork() {
  using parent = tracer.startActiveSpan("doWork");
  // do some work that 'parent' tracks
  console.log("doing some work...");
  await doAnotherWork();
  // Create a nested span to track nested work
  {
    using child = tracer.startActiveSpan("child");
    // do some work that 'child' tracks
    console.log("doing some nested child work...");
  }
  // Call a generator function
  for await (const work of doGeneratedWork()) {
    console.log("did work ", work);
  }
  // This parent span is also closed when it goes out of scope
}
```

> This example is adapted from the OpenTelemetry Python example.
> https://opentelemetry.io/docs/languages/python/instrumentation/#creating-spans

With proposal A, an implementation of `tracer.startActiveSpan` could look like
below. It retrieves the parent span from its own `AsyncContext.Variable`
instance and create span as a child, and set the child span as the current value
of the `AsyncContext.Variable` instance:

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
[issue-2]: https://github.com/legendecas/proposal-async-context-disposable/issues/2
[issue-3]: https://github.com/legendecas/proposal-async-context-disposable/issues/3
