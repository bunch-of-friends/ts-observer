# @bunch-of-friends/observable

[![License](https://img.shields.io/github/license/mashape/apistatus.svg?style=flat-square)](https://github.com/bunch-of-friends/lerna-typescript-jest-boilerplate/blob/master/LICENSE)
[![npm version](https://badge.fury.io/js/%40bunch-of-friends%2Fobservable.svg)](https://badge.fury.io/js/%40bunch-of-friends%2Fobservable)
[![CircleCI](https://circleci.com/gh/bunch-of-friends/observable.svg?style=svg)](https://circleci.com/gh/bunch-of-friends/observable)

A simple implementation of the [observer pattern](https://en.wikipedia.org/wiki/Observer_pattern) written in TypeScript, usable in JavaScript as well.

## Key objects
There are three key objects: [Subject](https://github.com/bunch-of-friends/observable/blob/master/src/subject.ts), [Observable](https://github.com/bunch-of-friends/observable/blob/master/src/observable.ts) and [Observer](https://github.com/bunch-of-friends/observable/blob/master/src/observer.ts).

### Observer
A function, which gets called on a change. Observers can return a promise, see [Async observers](#async-observers) below.
```ts
export interface Observer<T> {
    (newState: T, previousState?: T): void | Promise<void>;
}
```

### Subject
Object responsible for maintaining a list of observers, keeping track of the state and notifying its changes.
Subject is intended to be used internally and shouldn't be exposed outside of the object using it.
```ts
export interface Subject<T> {
    registerObserver(observer: Observer<T>, owner: Object): Observer<T>;
    unregisterObserver(observer: Observer<T>): void;
    unregisterObserversOfOwner(owner: Object): void;
    unregisterAllObservers(): void;
    notifyObservers(newState?: T): Promise<void>;
    getCurrentState(): T;
}
```

### Observable
Observable is a subset of Subject and is intended to be exposed outside of the object using it. Users of the Observable can freely register and unregister, but cannot notify changes.
```ts
export interface Observable<T> {
    register: (observer: Observer<T>) => Observer<T>;
    unregister: (observer: Observer<T>) => void;
    unregisterAllObservers: () => void;
    getCurrentState(): T;
}
```

## Usage
The library is published to npm as commonjs ES5 module. It is intended to be used in browser, but requires a module bundler.
The library comes with embedded TypeScript types, so it is easy to use in TypeScript, but it can be used as well in JavaScript.

```bash
npm install @bunch-of-friends/observable --save
```

Use the `createSubject` function to create an instance of Subject:
```ts
import { createSubject } from '@bunch-of-friends/observable';

const subject = createSubject<T>(); //replace `T` with type of the object that the observers will be notified with

subject.registerObserver(currentState => /* ... */ );

subject.notifyChanges({ /* ... */ }); // the argument of notifyChanges is of type `T`
```

To create an observable that can be used for registering and unregistering observers only, use the `createObservable` function:
```ts
import { createObservable } from '@bunch-of-friends/observable';

const observable = createObservable(subject); // as created earlier

observable.register(currentState => /* ... */ );
```

You can also create an observable that only filters only changes that match a specific values, use the `createObservableForValue` function:
```ts
import { createSubject, createObservableForValue } from '@bunch-of-friends/observable';

enum State {
    Loading,
    Loaded
}

const subject = createSubject<State>();
const observable = createObservableForValue(subject, State.Loaded);
observable.registerObserver(() => console.log('state changed to loaded'))

subject.notifyChanges(State.Loading); // this will not call the observer
subject.notifyChanges(State.Loaded); // this will call the observer

```

## Example
A simple example demonstrating the intended use:
```ts
import { createSubject, createObservable, createObservableForValue } from '@bunch-of-friends/observable';

enum State {
    Loading,
    Loaded
}

class Component {
    private stateSubject = createSubject<State>();

    // use this to subscribe to any state change
    public onStateChanged = createObservable(stateSubject);

    // shorthand if you only want to subscribe to the state changing to Loaded
    public onLoaded = createObservableForValue(stateSubject, State.Loaded);

    constructor() {
        // notifyChanges would usually be called by some more reasonable code
        setTimeout(() => this.stateSubject.notifyChanges(State.Loading), 1000);
        setTimeout(() => this.stateSubject.notifyChanges(State.Loaded), 2000);
    }

    public dispose() {
        this.stateSubject.unregisterAllObservers();
    }
}

class ComponentConsumer {
    constructor(component: Component) {
        component.onStateChanged.register((state) => console.log('current state is: ' + state));
        component.onLoaded.register(() => console.log('component loaded'));
    }
}
```

## Async observers
The `Observer` function can return a promise. When `Subject` is notified of changes by the `notifyObservers` function, it will wait for all observers that return a promise and it will resolve, when all observers resolve or reject. `notifyObservers` will never reject, it catches all rejections. Errors are expected to be handled by the observers.

## Classes vs closures
Why are the `Subject` and `Observable` not ES6 classes?
The reason is that the classes would have private fields, for example `Subject` would have private field `registeredObservers`. The TypeScript complier doesn't allow other TypeScript code to acess it, but in plain JavaScript, the private fields will be accessible and that will violate the open/close principle.

In the example above, we use ES6 class, try running similar code. You will be able to access the private field `stateSubject`, which is not inteded, in TypeScript it is marked as private.

Therefore the `createSubject` and `createObservable` act as constructors and and we use closures instead of hide the private variables from the users.
