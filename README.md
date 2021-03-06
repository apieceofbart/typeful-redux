# typeful-redux

A type-safe, low boilerplate wrapper for redux to be used in TypeScript projects.

## Elevator pitch

This is how you create a reducer and a store with typeful-redux. Note that all
calls are fully type-safe and will trigger type errors when used incorrectly.

```TypeScript
interface TodoItem {
    task: string;
    completed: boolean;
}

// Create a new reducer with initial state [], then add three actions
const TodoReducer = createReducer([] as TodoItem[])
    ('clear', s => [])
    ('add', (s: TodoItem[], newItem: TodoItem) => [...s, newItem])
    ('toggle', (s: TodoItem[], index: number) => [
        ...s.slice(0, index),
        { ...s[index], completed: !s[index].completed },
        ...s.slice(index + 1)
    ]);

// Create the store
const store = new StoreBuilder()
    .addReducer('todos', TodoReducer)
    .addMiddleware(reduxLogger) // as example
    .build();

// The result has type: { todos: TodoItem[] }
const state = store.getState();

// All dispatches are fully type checked

// Dispatches { type: 'todos/clear' }
store.dispatch.todos.clear();

// Dispatches
// { type: 'todos/add', payload: { completed: false, task: 'Provide a fully type-safe interface to redux' } }
store.dispatch.todos.add({
    task: 'Provide a fully type-safe interface to redux',
    completed: false
});

// Dispatches { type: 'todos/toggle', payload: 0 }
store.dispatch.todos.toggle(0);
```

A very simple, runnable example app can be found [here](./examples/simple-todo/). A TodoMVC implementation with
slightly more features is availabe [here](./examples/todomvc/).

## Motivation

[redux] is a fantastic approach to manage state in single page applications.
Unfortunately, vanilla redux requires some boilerplate and is hard to use
in a type-safe way.

[typeful-redux]'s primary goal is to provide a fully type-safe interface to redux. This means the redux `getState` and `dispatch` functions need to have the right types and these types should be maintained when using the [react-redux] `connect` function. All type-incorrect usages of `getState` or `dispatch` should trigger a type errror.

More specifically, [typeful-redux] seeks to address the following challenges when using redux:

  - **Full type safety:** redux makes it hard to fully type the `dispatch`
    method, to guarantee that only actions are dispatched which are handled by the store or that the dispatched actions are type correct (i.e. have the right payload).

    typeful-redux creates a store that gives a fully type-safe dispatch object, where every action is available as a function expecting the right payload. The `getState` method is also fully typed and returns a state with the right type.

  - **Low Boilerplate:** redux needs actions, possibly action creators and reducers.
    When trying to set this up in a type-safe way, many things need to be written down twice (or more). This introduces an opportunity for inconsistencies and errors.

    In typeful-redux, actions and their reducers are defined simultaneously,reducing the amount of code that needs to be written and maintained.

  - **Avoid inconsistencies:** When actions and reducers are defined
    seperately, there is the potential to forget handeling an action (or to misspell a type in a reducer's switch statement). typeful-redux makes this impossible by requiring the simultaneous definition of an action with its reducing code.

  - **Modularity:** In redux, each action type must be unique - so the action type
    namespace is 'global'. This is non-modular, e.g. the same actions and reducers can't be used for two parts of the store.

    typeful-redux namespaces reducers when combining them in a store. This means action types only need to be unique for any single reducer. Additionally, the same reducer can be used multiple times, for several parts of a store.

Besides these differences and different surface appearence, typeful-redux **is not an alternative redux implementation**, it is just a thin wrapper around reducer and store creation. The resulting runtime objects are plain redux reducers and stores equipped with the right type definitions (and sometimes some tiny convenience wrappers). All the existing redux ecosystem should be usable with this library. Please file an issue if you have trouble using a redux library with typeful-redux.

## Documentation

typeful-redux exports two functions and one class (and a few supporting type definitions). The purpose of the functions is described here. Also see the [examples](./examples/) for example usages. If you find the documentation insufficient please file an issue or complain to me via email (see profile).

### `createReducer`

This function allows creating a reducer by adding action names and the code
'reducing' the action simultaneously. While adding actions, the type of the
reducer is refined so that the right type of the `dispatch` object can be
inferred.

Actions and their handlers can be added by either calling the function with
the actions type name and a function handeling the reduction or by using the
`addSetter` (for creating an action without payload) or `addHandler` methods
(for creating an action with payload).

The initial example uses the call syntax to create three actions:

```TypeScript
const TodoReducer = createReducer([] as TodoItem[])
    ('clear', s => [])
    ('add', (s: TodoItem[], newItem: TodoItem) => [...s, newItem])
    ('toggle', (s: TodoItem[], index: number) => [
        ...s.slice(0, index),
        { ...s[index], completed: !s[index].completed },
        ...s.slice(index + 1)
    ]);
```

There is an alternative syntax to create a reducer with `addSetter` and `addHandler` methods to add new actions and reduction cases, this looks as follows:

```TypeScript
const TodoReducer = createReducer([] as TodoItem[])
    .addSetter('clear', s => [])
    .addHandler('add', (s: TodoItem[], newItem: TodoItem) => [...s, newItem])
    .addHandler('toggle', (s: TodoItem[], index: number) => [
        ...s.slice(0, index),
        { ...s[index], completed: !s[index].completed },
        ...s.slice(index + 1)
    ]);
```


### `StoreBuilder`

The `StoreBuilder` class is used to assemble a store using one or multiple
reducers and redux middlewares. It extracts the reducers from the objects
created by `createReducer` and returns a redux store, where the `dispatch`
function is extended by fully typed functions which dispatch the actions
created via `createReducer`. Otherwise the result of the `.build()` method
is a plain redux store where `getState()` has the right return type inferred.

```TypeScript
// Create the store
const store = new StoreBuilder()
    .addReducer('todos', TodoReducer)
    .addMiddleware(reduxLogger) // as example
    .build();
```

`store` is a plain redux store with `getState`, `subscribe` and `dispatch`
methods. The only difference is that `dispatch` now also holds objects with
methods to enable a type-safe dispatch and that `getState` has the _right_
return type.

```TypeScript
// This is fully typed
store.dispatch.todos.clear();
store.dispatch.todos.add({ task: 'Tell world about typeful-redux', completed: false });
store.dispatch.todos.toggle(0);
```

The type of `store.dispatch.todos` is

```TypeScript
{
    clear(): void;
    add(newItem: TodoItem): void;
    toggle(index: number): void;
}
```

Each method dispatches the right action on the store with the passed argument
as the payload. Actions are namespaced - so `store.dispatch.toggle(0)` dispatches
a `{ type: 'todos/toggle', payload: 0 }` action. This means that action types no
longer have to be globally unique - they just have to be unique for their reducer
and enables using the same reducer for multiple parts of the store.


### `connect`

This is a re-export of the redux connect function, with a more restricted type to ensure that the typing of the `dispatch` object is known in the `mapDispatchToProps` function. This makes it possible to *propagate type errors through `connect`*, which is not possible with the current type definition of [react-redux]'s `connect`.

To explain how `connect` can be used to its full benefit, we must understand the type
of the produced `store`. In general `store` will have the following type:

```TypeScript
type Store<STATE, DISPATCH> = {
    getState(): STATE;
    subscribe(): void;
    dispatch: DISPATCH;
};
```

where `STATE` is a map from the reducer names (here: `todos`) and the state types and
`DISPATCH` is a map from the reducer names (again `todos`) to functions which dispatch
the respective actions.

Now the `connect` function is set up so given a `mapStateToProps` which accepts a `STATE`
and a `mapDispatchToProps` which accepts a `DISPATCH`, it produces a container which needs
to have a `{ store: Store<STATE, DISPATCH>; }` as part of properties. This way the types
from the store can be propagated all the way to the components and changing the type
of an action-reducer triggers a type-error in all the right places.

```TypeScript
interface State {
    todos: TodoItem;
}

interface Dispatch {
    todos: {
        add(description: string): void;
        clear(): void;
        toggle(index: number): void;
    };
}
// Let's say we have a `TodoListComponent` which wants the following
// properties
interface TodoListProps {
    todos: TodoItem;
    add(description: string): void;
    clear(): void;
    toggle(index: number): void;
}

class TodoListComponent extends React.Component<TodoListProps> {
    // ...
}

const mapStateToProps = (state: State) => state;
const mapDispatchToProps = (dispatch: Dispatch) => dispatch.todos;


// TodoListContainer is infered to have a type which requires a property
// `{ store: Store<State, Dispatch> }`
//
const TodoListContainer = connect(mapStateToProps, mapDispatchToProps)(TodoListComponent);
```

`connect` can also be used with `mapStateToProps` and `mapDispatchToProps` with
a second argument, these second arguments become part of the required properties
of the connected container.

## How does it work?

A description of the tricks used to make typeful-redux do what it does can be found
[here](http://koerbitz.me).

## License

MIT


[redux]: http://redux.js.org
[react-redux]: https://github.com/reactjs/react-redux
[typeful-redux]: https://gitlab.com/paul.koerbitz/typeful-redux