# ember-resources

An implementation of Resources in Ember.JS without decorators.
 - [More information on Resources](https://www.pzuraq.com/introducing-use/)
 - [Inspiration, ember-could-get-used-to-this](https://github.com/pzuraq/ember-could-get-used-to-this)

## Compatibility

* Ember.js v3.25+
* TypeScript v4.2+

_NOTE_: if you are also using ember-could-get-used-to-this, `@use` is not compatible with
this library's `LifecycleResource`, and `useResource` does not work with ember-could-get-used-to-this' `Resource`.
However, both libraries can still be used in the same project.

## Installation

```bash
npm install ember-resources
# or
yarn add ember-resources
# or
ember install ember-resources
```


## Usage

### `useResource`

`useResource` takes a `LifecycleResource` and an args thunk.

```ts
class MyClass {
  data = useResource(SomeResource, () => [arg list]);
}
```

When any tracked data in the args thunk, the `update` function on `SomeResource`
will be called.

the args thunk accepts the following data shapes:
```
() => [an, array]
() => ({ named: 'args', hello: 'there' })
() => ({ named: {...}, positional: [...] })
```
#### An array

when an array is passed, inside the Resource, `this.args.named` will be empty
and `this.args.positional` will contain the result of the thunk.

#### An object of named args

when an object is passed where the key `named` is not also an object (if it exists),
`this.args.named` will contain the result of the thunk and `this.args.positional`
will be empty.

#### An object containing both named args and positional args

when an object is passed containing the key `positional` or the value of `named` is an object:
 - `this.args.named` will be the value of the result of the thunk's `named` property
 - `this.args.positional` will be the value of the result of the thunk's `positional` property

This is the same shape of args used throughout Ember's Helpers, Modifiers, etc



### `useTask`

This is a utility wrapper like `useResource`, but can be passed an ember-concurrency task
so that the ember-concurrency task can reactively be re-called whenever args change.
This largely eliminates the need to start concurrency tasks from the constructor, modifiers,
getters, etc.

A concurrency task accessed via `useTask` is only "ran" when accessed, and automatically updates
when it needs to.

```ts
class MyClass {
  myData = useTask(this.myTask, () => [args, to, task])

  @task
  *myTask(args, to, task)  { /* ... */ }
}
```

### Making your own Resources with

#### `LifecycleResource`

This resource base class has 3 lifecycle hooks:
 - `setup` - called upon first access of the resource
 - `update` - called when any `tracked` used during `setup` changes
 - `teardown` - called when the containing context is torn down

An example of this might be an object that you want to have perform some
complex or async behavior

```ts
class MyResource extends LifecycleResource {
  @tracked isRunning;
  @tracked error;

  get status() {
    if (this.isRunning) return 'pending';
    if (this.error) return this.error;

    return 'idle';
  }

  setup() {
    this.doAsyncTask();
  }

  update() {
    this.doAsyncTask();
  }

  async doAsyncTask() {
    // need to consume potentially tracked data so that
    // update may be called when these args change
    let [ids] = this.args.positional;

    // defer to next (micro)task queue to not block UI
    // (and avoid double render bugs because we're about to set tracked data)
    await Promise.resolve();

    this.isRunning = true;
    this.error = undefined;

    try {
      // some long running stuff here
    } catch (e) {
      this.error = e
    }

    this.isRunning = false;
  }
}
```

Using your custom Resource would look like
```ts
class ContainingClass {
  data = useResource(MyResource, () => [this.ids])
}
```

#### `function` Resources

While functions can be "stateless", Resources don't provide much value unless
you can have state. `function` Resources solve this by passing the previous
invocation's return value as an argument to the next time the function is called.

Example:
```ts
class StarWarsInfo {
  info = useResource(async (state, ...args) => {
    if (state) {
      let { characters } = state;

      return { characters };
    }

    let [ids] = args;
    let response = await fetch(`/characters/${ids}`) ;
    let characters = await response.json();

    return { characters };
  }, [this.ids /* defined somewhere */])
}
```

While this example is a bit contrived, hopefully it demonstrates how the `state` arg
works. During the first invocation, `state` is falsey, allowing the rest of the
function to execute. The next time `this.ids` changes, the function will be called
again, except `state` will be the `{ characters }` value during the first invocation,
and the function will return the initial data.

This particular technique could be used to run any async function _safely_ (as long
as the function doesn't interact with `this`).

In this example, where the function is `async`, the "value" of `info` is `{}` until the
function completes.


## Contributing

See the [Contributing](CONTRIBUTING.md) guide for details.


## License

This project is licensed under the [MIT License](LICENSE.md).
