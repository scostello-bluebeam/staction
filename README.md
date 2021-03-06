# Staction
A straightforward method of managing state, with support for Promises, Generators, Async/Await, and Async Generators (asyncIterable).

Because sometimes all you really need is state and actions.

[![Build Status](https://travis-ci.org/brochington/staction.svg?branch=master)](https://travis-ci.org/brochington/staction)

Basic usage:

```javascript
import Staction from 'staction'

let staction = new Staction()

let actions = {
  increment: ({ state, actions } , incrementAmount = 1) => {
    return {
      count: state().count + incrementAmount
    };
  }
}

let initialState = (actions) => {
  return {count: 0};
}

/*
   This method is called every time after state is updated.
   Useful for calling setState at the top of a React component tree.
*/

let onStateUpdate = (state, actions) => console.log(`count is ${state}`)

staction.init(
  actions,
  initialState,
  onStateUpdate
)

let incrementAmount = 5

/*
   all actions return a promise, that resolves with the updated state.
   This helps eliminate passing callbacks through, and also allows for a way
   to react to errors that occur in actions from outside the action.
*/
let result = staction.actions.increment(incrementAmount)

result
  .then(newState => console.log(newState))
  .catch(e => console.log(e))


console.log(staction.state) // state is {count: 5}
```

## Actions

Actions should always yield/return the full new state, or a Promise that resolves to the new state. They can be regular, async, or generator functions, and async call order will be maintained! The state argument is a function that will always return the current state. Additional arguments passed when the action is invoked are passed after state and actions.


All of the following are valid actions.

```javascript
const myActions = {
  action1: ({ state, actions }) => state() + 1,

  actions2: ({ state }) => {
    return Promise.resolve(state() + 1);
  },

  action3: function* ({ state, actions }) {
    yield state() + 1;
    // state() === 1

    yield state() + 1;
    // state() === 2
  },

  action4: function* ({ state, actions }) {
    yield state() + 1
    // state() === 1

    yield new Promise(resolve => resolve(state() + 1))
    // state() === 2

    // IIFE's come in really handy if you want to combine Generators and async functions.
    yield (async function(){
      return state() + 1
      // state() === 3
    }())
  },

  // ...Or just use Async Generators! Very handy for "isFetching" type state.
  action5: async function* ({ state }) {
    const nextState = await Promise.resolve(state() + 1);

    yield nextState;
  }
}
```

## Typescript

Staction aims to provide great Typescript support. This includes maintaining types througout actions! 

```typescript
import Staction, { ActionParams } from 'staction';

type State = {
  foo: string;
};

const initialState: State = {
  foo: 'bar',
};

type Params = ActionParams<State, Actions>;

const actions = {
  action1: (params: Params, val: string): State => {
    return params.state();
  },

  action2: async (params: Params): Promsie<State> => {
    return params.state();
  },
};

// You may explicitly type the actions above, this is just a bit of
// a shortcut.
type Actions = typeof actions;

const store = new Staction<State, Actions>();

store.init(
  actions,
  () => initialState,
  () => {}
);

// The arguments of this action should be correctly typed when calling it.
store.actions.actions1('hello');
```

## With React (and Typescript)

A "state down, actions up" style of configuration in a React component might look something like the following:

```typescript

/*  appState.ts  */

export type AppState = {
  foo: string;
}

export const initialState: AppState = {
  foo: 'bar',
}

/* appActions.ts */

import { ActionParams } from 'staction';

type Params = ActionParams<State, Actions>;

const appActions = {
  noopAction: (params: Params) => {return state}
}

export type AppActions = typeof appActions;

/* appStoreContext.ts */

import React from 'react';
import Staction from 'staction';
import { AppState } from './appState';
import { AppActions } from './appActions';

const defaultStaction = new Staction<AppState, AppActions>();

export type AppStore = Staction<AppState, AppActions>;

export const AppStoreContext = React.createContext<AppStore>(defaultStaction);

/* App.ts */

import React, { FC } from 'react';
import Staction, { ActionParams } from 'staction';
import { AppStore, AppStoreContext } from './appStoreContext';
import { initialState, AppState } from './appState';
import { appActions, AppActions } from './appActions';

const MyComponent: FC () => {
  const [appStore, setAppStore] = useState<AppStore>(new Staction<AppState, AppActions>());
  const [currentAppState, setCurrentAppState] = useState<AppState>(initialState);

  useEffect(() => {
    appStore.init(
      appActions,
      () => initialState,
      (nextState) => setCurrentAppState(nextState)
    )
  }, []);


  return appStore.initialized ? (
    <AppStoreContext.Provider value={appStore}>
      {/* App Components */}
    </AppStoreContext.Provider>
  ) : null;
}
}
```


## Staction instance methods

### Logging

- `staction.enableLogging()` - enable logging of each action call to console.
- `staction.disableLogging()` - disable logging of actions to console.
- `staction.disableStateWhenLogging()` - do not include current state when action logs.
- `staction.enableStateWhenLogging()` - include current state in action logs.


## Middleware

Staction supports basic middleware. They can be called pre or post action.

```javascript

const staction = new Staction();

staction.setMiddleware([
  {
    type: 'pre', // or 'post,
    method: ({ state, name, args, meta }) => { /* do stuff here. */ },
    meta: {} // An object of user configurable meta data. 
  }
])
```
