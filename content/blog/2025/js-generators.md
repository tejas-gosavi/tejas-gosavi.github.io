---
title: "Why I Obsessed Over JavaScript Generators"
date: 2025-06-26
draft: false
summary: 'My journey discovering how JavaScript generators changed the way I write cleaner, more efficient code'
slug: 'why-i-obsessed-over-js-generators'
---

So here's how my obsession with JavaScript generators started...

A few days back, I was working on a task where I had to write a data-fix/migration script. At first, it looked easy and ran well for a few users, but then we had to run it for all users. That's when it started taking more time and causing memory and CPU issues, as it had to fetch all users' data and store it while processing.

I understood that to handle this efficiently, we needed to process data in batches. We considered a database-level approach using cursors, but after studying them more, I found out that they can also cause performance issues. I started searching for better approaches, and that's when I discovered a solution with generators.

So, enough of the past. Let's focus on the present.

## Understanding the Foundations

Before diving into generators, we need to understand a few concepts like **iterator** and **iterable**. Let's get started.

### What are Iterators?

Simple definition: any object that follows the iterator protocol. This means any object which has a method called `next()` that returns an object containing two keys named `value` and `done`.

- `value` refers to the current value for the corresponding iteration
- `done` indicates whether the iteration is finished or not

Here's an example:

```js {linenos=inline}
const getUsers = {
  index: 0,
  next() {
    const users = ["Tejas", "Omkar", "Yash", "Manthan"];
    return {
      value: users.at(this.index++),
      done: this.index > users.length,
    };
  },
};

getUsers.next() // {value: 'Tejas', done: false}
getUsers.next() // {value: 'Omkar', done: false}
getUsers.next() // {value: 'Yash', done: false}
getUsers.next() // {value: 'Manthan', done: false}
getUsers.next() // {value: undefined, done: true}
// ...            this still continues if called
getUsers.next() // {value: undefined, done: true}
```

In the above snippet, you can see we're indicating that iteration ends when the index is out of array bounds using the `done` key, but we're still able to call the `next()` method. There's no automatic termination here.

By the way, we can also have an infinite iterator like this:

```js {linenos=inline}
const iAmInfinity = {
  index: 0,
  next() {
    return {
      value: this.index++,
      done: false,
    };
  },
};

iAmInfinity.next() // {value: 0, done: false}
iAmInfinity.next() // {value: 1, done: false}
iAmInfinity.next() // {value: 2, done: false}
iAmInfinity.next() // {value: 3, done: false}
iAmInfinity.next() // {value: 4, done: false}
// ... continues indefinitely
```

So, this is the iterator protocol. Not exciting, right? Same as functions and closures? But bear with me for the next 10 minutes—you'll understand what I'm getting at.

### What are Iterables?

Again, simple definition: any object that follows the iterable protocol. This means any object which has a method called `[Symbol.iterator]()` that returns an iterator.

Here's an example:

```js {linenos=inline}
const getUsers = {
  [Symbol.iterator]() {
    return {
      index: 0,
      next() {
        const users = ["Tejas", "Omkar", "Yash", "Manthan"];
        return {
          value: users.at(this.index++),
          done: this.index > users.length,
        };
      },
    };
  },
};
```

Now you might ask: what's the difference? Just some weird syntax change? But wait a minute...

You know what `for...of` takes as input to loop over? That's right—an iterable!

So you can do the following with a custom-defined iterable:

```js {linenos=inline}
for (const user of getUsers) {
  console.log(user); // "Tejas", "Omkar", "Yash", "Manthan"
}
```

And since we can do similar things with arrays, objects, etc., this means I can also destructure it like:

```js {linenos=inline}
const [u1, u2, ...others] = getUsers;
console.log(u1); // "Tejas"
console.log(u2); // "Omkar"
console.log(others); // ['Yash', 'Manthan']
```

## Enter Generators

Now generators enter the scene. They are similar to iterators and iterables but with cleaner syntax.

```js {linenos=inline}
function* getUsers() {
  yield "Tejas";
  yield "Omkar";
  yield "Yash";
  yield "Manthan";
}

const getUsersGenerator = getUsers();

getUsersGenerator.next() // {value: 'Tejas', done: false}
getUsersGenerator.next() // {value: 'Omkar', done: false}
getUsersGenerator.next() // {value: 'Yash', done: false}
getUsersGenerator.next() // {value: 'Manthan', done: false}
getUsersGenerator.next() // {value: undefined, done: true}
// ...            this continues if called
getUsersGenerator.next() // {value: undefined, done: true}

// We can also use them in for..of
for (const user of getUsers()) {
  console.log(user); // "Tejas", "Omkar", "Yash", "Manthan"
}
```

As you can see, it's the same as a normal function but with a `*` at the end of the `function` keyword, which acts as an identifier for generator functions. We also see a keyword called `yield` which acts as the return value for the current iteration. Using `yield`, we can start and pause our processing/execution.

Normal functions run from start to end, but using `yield` we can pause the function and restart its execution from the paused step. It continues working until it reaches the last `yield` statement or `return` statement.

## Async Generators: The Real Magic

There are also **Asynchronous Generators** which handle async processing, and here's how I used them to solve my use case:

```js {linenos=inline}
async function* getUsersInBatch(batchSize = 10) {
  const usersCount = await getAllUsersCount();
  const totalBatches = Math.ceil(usersCount / batchSize);

  for (let batchNumber = 0; batchNumber < totalBatches; batchNumber++) {
    const offset = batchNumber * batchSize;
    const limit = Math.min(batchSize, usersCount - offset);

    const usersBatch = await getUsers(limit, offset);
    
    yield {
      usersBatch,
      batchNumber: batchNumber + 1,
      totalBatches
    };
  }
}

// Usage
for await (const { usersBatch, batchNumber, totalBatches } of getUsersInBatch()) {
  await performDataFixForUsers(usersBatch); // some function for processing
  console.log(`Completed ${batchNumber}/${totalBatches} batches`);
}
```

This might look like a lot to digest, but let me explain how it works step by step:

1. **Initialize the generator**: We call `getUsersInBatch()`, which returns a generator object
2. **Calculate batches**: We first get the total count of users to determine batch count, then calculate the actual number of batches needed to fetch all users
3. **Process in batches**: We use a loop to fetch users with `limit` and `offset`, and after fetching every batch, we `yield` that batch
4. **Pause and resume**: The `yield` statement is inside a `for` loop, which means this function continues working until all batches are completed (all users are fetched)

Here's the beautiful part: we pause execution after fetching every batch and give that batch to a function for processing only that particular batch. Once processing is finished, we restart execution—the `batchNumber` is incremented and we fetch a new batch based on the new `limit` and `offset`.

Also, this generator function is generic, meaning we can use it in multiple use cases where we need to fetch users in batches and perform some processing on them.

## Why This Approach Won

This solution solved my original problem perfectly:
- **Memory efficient**: Only one batch is loaded in memory at a time
- **Pausable execution**: Processing can be paused and resumed
- **Clean code**: Much more readable than manual pagination logic
- **Reusable**: The generator can be used across different scenarios

So yes, that's how I fell in love with generators.

See you in next blog post!