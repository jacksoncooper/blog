---
title: Queues in JavaScript
date: 2022-07-03
layout: layouts/post.njk
---

JavaScript is adorable. It's got abusable C-style syntax like [the comma operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Comma_Operator) and you can fit the entirety of its inheritance model in your head. JavaScript is also adorable because its standard library is a bit threadbare. It doesn't have queues.

That's okay, let's write our own.

### Attempt 1: Just use objects

To keep the weird syntax from TypeScript simple, our queue will hold strings.

```ts
type Queue = { [property: symbol]: Item };
type Item = string;

const queue = {
    enqueue: function(this: Queue, value: Item): void {
        this[Symbol()] = value;
    },

    dequeue: function(this: Queue): Item | null {
        const values = Object.getOwnPropertySymbols(this);
        if (values.length > 0) {
            const property = values[0], oldest = this[property];
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

### Attempt 2: Be functional

