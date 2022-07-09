---
title: Queues in JavaScript
date: 2022-07-03
layout: layouts/post.njk
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

and translate it to TypeScript. Type definitions cannot recursive, but they can be mutually recursive apparently.

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

Although we have types for crying out loud. We know that `oldest` and `newest` can't be `null` at the same time, but the types are decoupled. That's error-prone. We can walk through our old implementation and let the compiler guide us.

```ts
type State = 'empty' | { oldest: Node, newest: Node };

default class Queue
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

## Attempt 3: Be random-access

`TODO`.