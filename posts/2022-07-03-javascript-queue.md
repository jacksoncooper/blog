---
layout: layouts/post.njk
title: Queues in JavaScript
date: git Last Modified
authored: 2022-07-03
---

JavaScript is adorable. It's got abusable C-style syntax like [the comma operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Comma_Operator) and you can fit the entirety of its inheritance model in your head. JavaScript is also adorable because its standard library is a bit threadbare. It doesn't have queues.

That's okay, let's write our own.

## Attempt 1: Just use objects

To keep the weird syntax from TypeScript simple, our queue will hold strings.

```ts
type Queue = { [property: symbol]: Item };

type Item = string;

const queue = {
    enqueue: function(this: Queue, item: Item): void {
        this[Symbol()] = item;
    },

    dequeue: function(this: Queue): Item | null {
        const item = Object.getOwnPropertySymbols(this);
        if (item.length > 0) {
            const property = item[0], oldest = this[property];
            delete this[property];
            return oldest;
        }
        return null;
    }
 }

const pocket = Object.create(queue);

pocket.enqueue('string');
pocket.enqueue('button');
pocket.enqueue('shell');

console.log(pocket.dequeue()); // 'string'
```

Here's we're making use of the order in which properties are enumerated. In the documentation for the `for...in` loop, [MDN tells us](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in#description)

> The traversal order, as of modern ECMAScript specification, is well-defined and consistent across implementations. Within each component of the prototype chain, all non-negative integer keys (those that can be array indices) will be traversed first in ascending order by value, then other string keys _in ascending chronological_ order of property creation.

The emphasis is mine. Unfortunately, this implementation has a critical performance flaw in addition to being hacky.

```ts
const values = Object.getOwnPropertySymbols(this);
```

This call does not take time or space proportional to a constant. It must enumerate all of the items in the queue and collect them into an `Array`, from which we pluck the first, because the items are arranged in ascending chronological order, and so the oldest is at the front. This means `enqueue` is $$ \Theta(1) $$ and `dequeue` is $$ \Theta(n) $$.

As far as I know, there's no way to obtain a [generator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator). `getOwnPropertySymbols` eagerly marshals all the items in our queue.

## Attempt 2: Be functional

We take this type from Haskell

```haskell
data List = Node Item List | Null
type Item = String
```

used like

```haskell
things = Node "string" (Node "button" (Node "shell" Null))
```

and translate it to TypeScript. Type definitions cannot be recursive, but they can be mutually recursive apparently.

```ts
type List = Node | null;
type Node = { item: Item, next: List };

type Item = string;
```

We rely on the invariant that if one of the node references `oldest` or `newest` is `null`, then they are both `null`. We have to be kind of careful with our type narrowing of these references to avoid asserting to the compiler that they're not null (`!`) when accessing their `next` property.

I always struggle with remembering which end of the queue should be the `oldest` reference and which should be the `newest`. The key is that the `oldest` reference must be able to locate the next oldest item after a `dequeue` operation, and if it points to the end of the list it will be unable to swim upstream against the links.

This one does perform operations in constant time like we expect from a queue, and is probably the easiest to deploy in an interview if you're asked to implement your queue API or if you're doing an online assessment. Although you probably shouldn't be interviewing in JavaScript when you can just manifest a `dequeue` from the Python Standard Library.

Your CPU data cache will murmur mean things about you but it will be worth it.

```ts
class Queue
{
    oldest: List = null;
    newest: List = null;

    enqueue(item: Item): void {
        const newest = { item, next: null };

        // The list is empty.
        if (this.newest === null) {
            this.oldest = this.newest = newest;
            return;
        }

        this.newest.next = newest;
        this.newest = newest;
    }

    dequeue(): Item | null {
        // The list is empty.
        if (this.oldest === null) {
            return null;
        }

        const oldest = this.oldest.item;

        // The list has a single item.
        if (this.oldest === this.newest) {
            this.oldest = this.newest = null;
            return oldest;
        }

        this.oldest = this.oldest.next;
        return oldest;
    }
}
```

Although we have types, for crying out loud. We know that `oldest` and `newest` can't be `null` at the same time, but the types are decoupled. That's error-prone. We can walk through our old implementation and let the compiler guide us.

```ts
type State = 'empty' | { oldest: Node, newest: Node };

class Queue
{
    state: State = 'empty';

    enqueue(item: Item): void {
        const newest = { item, next: null };

        if (this.state === 'empty') {
            this.state = { oldest: newest, newest };
            return;
        }

        this.state.newest.next = newest;
        this.state.newest = newest;
    }

    dequeue(): Item | null {
        if (this.state === 'empty') {
            return null;
        }

        const oldest = this.state.oldest.item;

        // The list has a single item.
        if (this.state.oldest.next === null) {
            this.state = 'empty';
            return oldest;
        }

        this.state.oldest = this.state.oldest.next;
        return oldest;
    }
}
```

TypeScript often looks very removed from JavaScript, but the compiled code is identical. I've
compiled to CommonJS (using `export default class` in the module).

```ts
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
class Queue {
    state = 'empty';
    enqueue(item) {
        const newest = { item, next: null };
        if (this.state === 'empty') {
            this.state = { oldest: newest, newest };
            return;
        }
        this.state.newest.next = newest;
        this.state.newest = newest;
    }
    dequeue() {
        if (this.state === 'empty') {
            return null;
        }
        const oldest = this.state.oldest.item;
        // The list has a single item.
        if (this.state.oldest.next === null) {
            this.state = 'empty';
            return oldest;
        }
        this.state.oldest = this.state.oldest.next;
        return oldest;
    }
}
exports.default = Queue;
```

If you're interested in the generic version, it looks like this.

```ts
type List<T> = Node<T> | null;
type Node<T> = { item: T, next: List<T> };

type State<T> = 'empty' | { oldest: Node<T>, newest: Node<T> };

class Queue<T>
{
    state: State<T> = 'empty';

    enqueue(item: T): void {
        const newest = { item, next: null };

        if (this.state === 'empty') {
            this.state = { oldest: newest, newest };
            return;
        }

        this.state.newest.next = newest;
        this.state.newest = newest;
    }

    dequeue(): T | null {
        if (this.state === 'empty') {
            return null;
        }

        const oldest = this.state.oldest.item;

        // The list has a single item.
        if (this.state.oldest.next === null) {
            this.state = 'empty';
            return oldest;
        }

        this.state.oldest = this.state.oldest.next;
        return oldest;
    }
}
```

## Attempt 3: Be random-access

Below is a queue implementation that uses a circular array, which necessarily has a fixed capacity because it's circular (imagine taking a list and connecting its ends). The `oldest` and `newest` references advance in the same direction around the circle, just like our linked implementation. We just have to be mindful of what happens when the queue gets full.

Below, we ignore `enqueue` operations when we don't have room for the new element. We can also perform a sequential `dequeue` and `enqueue` operation to maintain the `this.capacity` most recent elements, useful for problems like a running average. This is all wonderfully hypothetical though, and I didn't bomb such an interview question last fall.

Besides this small wrinkle, the implementation looks identical to the linked queue.

There's no reason we have to clean up after ourselves by calling `delete`. Because both `oldest` and `newest` advance in the same direction, `newest` will write over the stale value before `oldest` gets to it. I just like being tidy.

Resizing can be done in amortized constant time just like a vector.

```ts
type State = 'empty' | { oldest: number, newest: number };

type Item = string;

class Queue
{
    capacity: number;
    state: State = 'empty';
    items: Item[] = [];

    constructor(capacity: number) {
        this.capacity = capacity;
    }

    enqueue(item: Item): void {
        if (this.state === 'empty') {
            this.state = { oldest: 0, newest: 0 };
            this.items[0] = item;
            return;
        }

        // The list is full.
        if (this.advance(this.state.newest) === this.state.oldest) {
            return;
        }

        this.state.newest = this.advance(this.state.newest);
        this.items[this.state.newest] = item;
    }

    dequeue(): Item | null {
        if (this.state === 'empty') {
            return null;
        }

        const oldest = this.items[this.state.oldest];
        delete this.items[this.state.oldest];

        // The list has a single item.
        if (this.state.oldest === this.state.newest) {
            this.state = 'empty';
            return oldest;
        }

        this.state.oldest = this.advance(this.state.oldest);
        return oldest;

    }

    advance(index: number): number {
        return (index + 1) % this.capacity;
    }
}
```

## Attempt 4: Oh no what about priority queues

Short answer: 😭

Long answer: Here is a terrible, horrible, no good, very bad binary heap. Don't use this in production, use
[heapq](https://docs.python.org/3/library/heapq.html).

```ts
class Heap<T>
{
    keys: T[] = new Array(1);
    lighter: (key: T, other: T) => boolean;

    constructor(lighter: (key: T, other: T) => boolean = (key, other) => key < other) {
        this.lighter = lighter;
    }

    enqueue(key: T): void {
        this.keys.push(key);
        this.swim(this.keys.length - 1);
    }

    dequeue(): T | null {
        if (this.keys.length === 1) {
            return null;
        }

        const top = this.keys[1];
        this.exchange(1, this.keys.length - 1); // What happens when the queue has length 1?
        this.keys.pop();
        this.sink(1);
        return top;
    }

    swim(child: number): void {
        let parent = this.parent(child);
        while (parent && this.lighter(this.keys[child], this.keys[parent])) {
            this.exchange(child, parent);
            child = parent, parent = this.parent(parent); // Comma operator abuse.
        }
    }

    sink(parent: number): void {
        let lighterChild = this.lighterChild(parent);
        while(lighterChild && this.lighter(this.keys[lighterChild], this.keys[parent])) {
            this.exchange(parent, lighterChild);
            parent = lighterChild, lighterChild = this.lighterChild(parent);
        }
    }

    lighterChild(parent: number): number | null
    {
        let lighter: number | null;
        let left = this.leftChild(parent), right = this.rightChild(parent);

        // And here we mourn Rust's if expressions. :(
        if (left && right) {
            lighter = this.lighter(this.keys[left], this.keys[right]) ? left : right;
        } else {
            lighter = left ? left : right;
        }

        return lighter;
    }

    leftChild(label: number): number | null {
        return this.walk(label, (label) => 2 * label);
    }

    rightChild(label: number): number | null {
        return this.walk(label, (label) => 2 * label + 1);
    }

    parent(label: number): number | null {
        return this.walk(label, (label) => Math.floor(label / 2));
    }

    walk(label: number, where: (label: number) => number) {
        const child = where(label);
        if (!(0 < child && child < this.keys.length)) {
            return null;
        }
        return child;
    }

    exchange(label: number, other: number): void {
        [this.keys[label], this.keys[other]] = [this.keys[other], this.keys[label]];
    }
}
```

If you just need to get your program to compile, you can fake it by using a heap that eagerly orders its items in linear time, just like arranging a hand of playing cards. This is insertion sort.

```ts
class Heap<T>
{
    keys: T[] = [];
    lighter: (key: T, other: T) => boolean;

    constructor(lighter: (key: T, other: T) => boolean = (key, other) => key < other) {
        this.lighter = lighter;
    }

    enqueue(key: T): void {
        this.keys.push(key);
        const n = this.keys.length;
        for (let i = n - 1; i > 0 && this.lighter(this.keys[i - 1], this.keys[i]); --i) {
            this.exchange(i, i - 1);
        }
    }

    dequeue(): T | null {
        // If you try to bind the return value of pop, TypeScript will require a non-null assertion.
        const n = this.keys.length;
        if (n > 0) {
            const last = this.keys[n - 1];
            this.keys.pop();
            return last;
        }
        return null;
    }

    exchange(label: number, other: number): void {
        [this.keys[label], this.keys[other]] = [this.keys[other], this.keys[label]];
    }

    static from<T>(lighter: (key: T, other: T) => boolean, keys: T[]): Heap<T> {
        const heap = new Heap<T>();

        const compare = (key: T, other: T): number => {
            if (lighter(key, other)) return 1;
            else if (lighter(other, key)) return -1;
            else return 0;
        }

        heap.keys = keys;
        heap.lighter = lighter;

        heap.keys.sort(compare);

        return heap;
    }
}
```
