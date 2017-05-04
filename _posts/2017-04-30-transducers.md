---
title: "What's a Transducer?"
layout: default
tags: [allonge]
---

In [Using iterators to write highly composeable code][previous-post], we took a look at a data transformation and analysis algorithm, and saw that the obvious [staged approach] was highly decomposed, but presented a performance problem in that it created excess duplicates of the entire data set.

Whereas the [singe pass approach] was much more memory-efficient, but the code was entangled and monolithic. On a problem small enough to fit in a blog post this isn't a massive problem, but it's easy to see how such an approach in production leads to highly coupled, fragile code that cannot be easily factored or decomposed.

[previous-post]: http://raganwald.com/2017/04/19/incremental.html
[staged approach]: http://raganwald.com/2017/04/19/incremental.html#I
[singe pass approach]: http://raganwald.com/2017/04/19/incremental.html#II

We concluded by looking at a [stream approach]. In the stream approach, we process the data in stages, but by using iterators and generators, we were able to process the data one datum at a time. This gave us the factorability of the staged approach, with the memory footprint of the single pass approach.

[stream approach]: http://raganwald.com/2017/04/19/incremental.html#III

[![a matrix dream](/assets/images/matrix-dream.jpg)](https://www.flickr.com/photos/gi/127757006)

Now we're going to look at another very interesting approach for building composeable pipelines of transformations without incurring a memory penalty. Let's start with reducing (a/k/a "folding"):

### reducers

A **reducer** is a function that takes an accumulation and a value, and folds the value into the accumulation. For example, if `[1, 2, 3]` is an accumulation ,and `4` is a value, `(acc, val) => acc.concat([val]);` is a reducer that returns `[1, 2, 3, 4]`:

```javascript
const acc = [1, 2, 3];
const val = 4;
const reducer = (acc, val) => acc.concat([val]);

reducer(acc, val)
  ///=> 1, 2, 3, 4
```

`(acc, val) => acc.concat([val])` is a reducer that returns the *catenation* of a list and a value.

Likewise, `(acc, val) => acc.add(val)` is a reducer that `.add`s a value to an accumulation. It works for any object that has a `.add` method and returns itself from `.add`, like [Set.prototype.add]:

```javascript
const acc = new Set([1, 2, 3]);
const val = 4;
const reducer = (acc, val) => acc.add(val);

reducer(acc, val)
  ///=> Set{1, 2, 3, 4}
```

[Set.prototype.add]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set/add

Here is a function that makes an array out of any iterable using our catenation reducer:

```javascript
const toArray = iterable => {
  const reducer = (acc, val) => acc.concat([val]);
  const seed = [];
  let accumulation = seed;

  for (value of iterable) {
    accumulation = reducer(accumulation, value);
  }

  return accumulation;
}

toArray([1, 2, 3])
  //=> [1, 2, 3]
```

We can extract our `reducer` and `seed` variables as parameters to create a *reduction* function:

```javascript
const reduce = (iterable, reducer, seed) => {
  let accumulation = seed;

  for (const value of iterable) {
    accumulation = reducer(accumulation, value);
  }

  return accumulation;
}

reduce([1, 2, 3], (acc, val) => acc.concat([val]), [])
  //=> [1, 2, 3]
```

In JavaScript, arrays have a `.reduce` method built in, and they behave exactly like our `reduce` function:

```javascript
[1, 2, 3].reduce((acc, val) => acc.concat([val]), [])
  //=> [1, 2, 3]
```

Now, `(acc, val) => acc.concat([val])` makes a lot of excess copies of things, and in JavaScript, we can substitute `(acc, val) => { acc.push(val); return acc; }`.[^comma]

[^comma]: `(acc, val) => (acc.push(val), acc)` is more pleasing semantically, but the comma operator is confusing to those who haven't seen its regular use, and usually best avoided in production code.

Either way, what we get is a reducer that accumulates values into an array. Let's give it a name:

```javascript
const arrayOf = (acc, val) => { acc.push(val); return acc; };

reduce([1, 2, 3], arrayOf, [])
  //=> [1, 2, 3]
```

Here's yet another reducer:

```javascript
const sumOf = (acc, val) => acc + val;

reduce([1, 2, 3], sumOf, 0)
  //=> 6
```

We can write reducers that reduce an iterable of one type (such as an array) into another type (such as a number).

### decorating reducers

JavaScript makes it easy to write functions that return functions. Here's a function that makes a reducer for us:

```javascript
const joinedWith =
  separator =>
    (acc, val) =>
      acc == '' ? val : `${acc}${separator}${val}`;

reduce([1, 2, 3], joinedWith(', '), '')
  //=> "1, 2, 3"

reduce([1, 2, 3], joinedWith('.'), '')
  //=> "1.2.3"
```

JavaScript also makes it easy to write functions that take functions as arguments. *Decorators* are JavaScript functions that take a function as an argument and return another function that is semantically related to its argument. For example, this function takes a binary function and decorates it by adding one to its second input:

```javascript
const incrementSecondArgument = binaryFn => (x, y) => binaryFn(x, y + 1);

const power = (base, exponent) => base ** exponent;

const higherPower = incrementSecondArgument(power);

power(2, 3)
  //=> 8

higherPower(2, 3)
  //=> 16
```

`higherPower` is `power`, decorated to add one to its `exponent`. Thus, `higherPower(2, 3)` produces the same result as `power(2, 4)`. We have been working with binary functions already, of course. Reducers are binary functions. Can we decorate them? Yes!

```javascript
reduce([1, 2, 3], incrementSecondArgument(arrayOf), [])
  //=> [2, 3, 4]

const incremented =
  iterable =>
    reduce(iterable, incrementSecondArgument(arrayOf), []);

incremented([1, 2, 3])
  //=> [2, 3, 4]
```

### mappers

We have produced a *mapper*, a function that takes an iterable and returns a mapping from the iterable's values to the incremented iterable's values. We map values all the time in JavaScript, but of course we want to do more than just increment. Let's take another look at `incrementSecondArgument`:

```javascript
const incrementSecondArgument = binaryFn => (x, y) => binaryFn(x, y + 1);
```

Since we're using it to decorate reducers, let's give it some more relevant names:

```javascript
const incrementValue = reducer => (acc, val) => reducer(acc, val + 1);
```

Now we see at a glance that `incrementValue` takes a reducer as an argument and returns a reducer that increments its value before reducing it further. We can extract the "incrementing" logic into a parameter:

```javascript
const map =
  fn =>
    reducer =>
      (acc, val) => reducer(acc, fn(val));

const incrementValue = map(x => x + 1);

reduce([1, 2, 3], incrementValue(arrayOf), [])
  //=> [2, 3, 4]
```

Although it looks unfamiliar to people not used to the idea of a function taking a function as an argument and returning a function that takes a function as an argument, we can write `map(x => x + 1)` anywhere we can write `incrementValue`, therefore we can write:

```javascript
reduce([1, 2, 3], map(x => x + 1)(arrayOf), [])
  //=> [2, 3, 4]
```

And because our `map` decorator can decorate any reducer, we can also join the increments of the numbers from one to three into a  string or sum them:

```javascript
reduce([1, 2, 3], map(x => x + 1)(joinedWith('.')), '')
  //=> "2.3.4"

reduce([1, 2, 3], map(x => x + 1)(sumOf), 0)
  //=> 9
```

[![Pythagoras Tree](/assets/images/pythagoras-tree.png)](https://commons.wikimedia.org/wiki/File:Pythagoras_tree_1_1_13_Summer.svg)

Armed with all we've seen so far, what is the sum of the squares of the numbers from one to ten?

```javascript
const squares = map(x => power(x, 2));
const one2ten = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

reduce(one2ten, squares(sumOf), 0)
  //=> 385
```

### filters

Let's go back to our first reducer:

```javascript
const arrayOf = (acc, val) => { acc.push(val); return acc; };

reduce(one2ten, arrayOf, 0)
  //=> [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

What if we want an array of just the numbers greater than five? Easily done:

```javascript
const bigUns = (acc, val) => {
  if (val > 5 ) {
    acc.push(val);
  }
  return acc;
};

reduce(one2ten, bigUns, [])
  //=> [6, 7, 8, 9, 10]
```

Naturally, we can combine what we already have to produce an array of the squares of the numbers greater than five:

```javascript
reduce(one2ten, squares(bigUns), [])
  //=> [9, 16, 25, 36, 49, 64, 81, 100]
```

This is not what we wanted! We have the *squares* that are greater than five, rather than the squares of the numbers that are greater than five. We want to do the selecting of numbers before we do the squaring, not after. This is easily done, and the insight is that what we want is a decorator that selects numbers, and we can use that to decorate the reducer:

```javascript
reduce(one2ten, squares(arrayOf), [])
  //=> [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

const bigUnsOf =
  reducer =>
    (acc, val) =>
      (val > 5) ? reducer(acc, val) : acc;

reduce(one2ten, bigUnsOf(squares(arrayOf)), [])
  //=> [36, 49, 64, 81, 100]
```

`bgUnsOf` is rather specific. Just as we did with `map`, let's extract the predicate function:

```javascript
reduce(one2ten, squares(arrayOf), [])
  //=> [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

const filter =
  fn =>
    reducer =>
      (acc, val) =>
        fn(val) ? reducer(acc, val) : acc;

reduce(one2ten, filter(x => x > 5)(squares(arrayOf)), [])
  //=> [36, 49, 64, 81, 100]
```

We can make all kinds of filters, and name them if we want. Or not:

```javascript
reduce(one2ten, filter(x => x % 2 === 1)(arrayOf), [])
  //=> [1, 3, 5, 7, 9]
```

With all this in hand, the sum of the squares of the odd numbers from one to ten is:

```javascript
reduce(one2ten, filter(x => x % 2 === 1)(squares(sumOf)), 0)
  //=> 165
```

### composing decorators

The essential character of this pattern is that we can take a reducer and compose it with as many mappers, filters, or other decorators that we cook up, and we end up with a reducer. Decorators *compose* with each other. We can even be explicit about this in a delightful way:

```javascript
const compositionOf = (acc, val) => (...args) => val(acc(...args));

const compose = (...fns) =>
  fns.reduce(compositionOf, x => x);

const squaresOfTheOddNumbers = compose(
  filter(x => x % 2 === 1),
  squares
);

reduce(one2ten, squaresOfTheOddNumbers(sumOf), 0)
  //=> 165
```

Yes, this formulation of `compose` uses `reduce` and in turn is used in our `reduce`.

Being able to compose decorators lets us decompose complex and highly coupled code into smaller units with a single responsibility that we can name if we choose.

### so what's a transducer?

Given reductions written in this style:

```javascript
reduce(one2ten, squaresOfTheOddNumbers(sumOf), 0)
```

We can note that we have four separate elements: A decorator for the reducer (which may be a composition of decorators), a seed, and an iterable. We can express the same thing like this:

```javascript
const reduce = (decorator, reducer, seed, iterable) => {
  const decoratedReducer = decorator(reducer);
  let accumulation = seed;


  for (const value of iterable) {
    accumulation = decoratedReducer(accumulation, value);
  }

  return accumulation;
}

reduce(squaresOfTheOddNumbers, sumOf, 0, one2ten)
  //=> 165
```

This does not resemble the `.reduce` method, or the way `reduce` functions are usually written, so we need some new names:

```javascript
const transduce = (transducer, reducer, seed, iterable) => {
  const decoratedReducer = transducer(reducer);
  let accumulation = seed;

  for (const value of iterable) {
    accumulation = decoratedReducer(accumulation, value);
  }

  return accumulation;
}

transduce(squaresOfTheOddNumbers, sumOf, 0, one2ten)
  //=> 165
```

And there you have it:[^clojure] A *reducer* is the kind of function you’d pass to `.reduce`—it takes an accumulated result and a new input, and returns a new accumulated result. A *transducer* is a function that decorates a reducer. Transducers compose to produce a new transducer.

[^clojure]: See https://clojure.org/reference/transducers

So, if someone asks you what a "transducer" is, you might reply:

![What's the problem?](/assets/images/what-s-the-problem.png)

---

### the complete code

```javascript
const arrayOf = (acc, val) => { acc.push(val); return acc; };

const sumOf = (acc, val) => acc + val;

const setOf = (acc, val) => acc.add(val);

const map =
  fn =>
    reducer =>
      (acc, val) => reducer(acc, fn(val));

const filter =
  fn =>
    reducer =>
      (acc, val) =>
        fn(val) ? reducer(acc, val) : acc;

const compose = (...fns) =>
  fns.reduce((acc, val) => (...args) => val(acc(...args)), x => x);

const transduce = (transducer, reducer, seed, iterable) => {
  const decoratedReducer = transducer(reducer);
  let accumulation = seed;

  for (const value of iterable) {
    accumulation = decoratedReducer(accumulation, value);
  }

  return accumulation;
}
```

### the transducer approach to tracking user transitions

(see [Using iterators to write highly composeable code][previous-post] for context.)

```javascript
const logContents = `1a2ddc2, 5f2b932
f1a543f, 5890595
3abe124, bd11537
f1a543f, 5f2b932
f1a543f, bd11537
f1a543f, 5890595
1a2ddc2, bd11537
1a2ddc2, 5890595
3abe124, 5f2b932
f1a543f, 5f2b932
f1a543f, bd11537
f1a543f, 5890595
1a2ddc2, 5f2b932
1a2ddc2, bd11537
1a2ddc2, 5890595`;

const asStream = function * (iterable) { yield * iterable; };

const lines = str => str.split('\n');
const streamOfLines = asStream(lines(logContents));

const datums = str => str.split(', ');
const datumize = map(datums);

const userKey = ([user, _]) => user;

const pairMaker = () => {
  let wip = [];

  return reducer =>
    (acc, val) => {
      wip.push(val);

      if (wip.length === 2) {
        const pair = wip;
        wip = wip.slice(1);
        return reducer(acc, pair);
      } else {
        return acc;
      }
  }
}

const sortedDecoration =
  (xfMaker, keyFn) => {
    const decoratedReducersByKey = new Map();

    return reducer =>
      (acc, val) => {
        const key = keyFn(val);
        let decoratedReducer;

        if (decoratedReducersByKey.has(key)) {
          decoratedReducer = decoratedReducersByKey.get(key);
        } else {
          decoratedReducer = xfMaker()(reducer);
          decoratedReducersByKey.set(key, decoratedReducer);
        }

        return decoratedReducer(acc, val);
      }
  }

const userTransitions = sortedDecoration(pairMaker, userKey);

const justLocations = map(([[u1, l1], [u2, l2]]) => [l1, l2]);

const stringify = map(transition => transition.join(' -> '));

const transitionKeys = compose(
  stringify, justLocations, userTransitions, datumize
);

const countsOf =
  (acc, val) => {
    if (acc.has(val)) {
      acc.set(val, 1 + acc.get(val));
    } else {
      acc.set(val, 1);
    }
    return acc;
  }

const greatestValue = inMap =>
  Array.from(inMap.entries()).reduce(
    ([wasKeys, wasCount], [transitionKey, count]) => {
      if (count < wasCount) {
        return [wasKeys, wasCount];
      } else if (count > wasCount) {
        return [new Set([transitionKey]), count];
      } else {
        wasKeys.add(transitionKey);
        return [wasKeys, wasCount];
      }
    }
    , [new Set(), 0]
  );

greatestValue(
  transduce(transitionKeys, countsOf, new Map(), streamOfLines)
)
  //=>
    [
      "5f2b932 -> bd11537",
      "bd11537 -> 5890595"
    ],
    4
```

### further reading

- [Understanding Transducers in JavaScript](https://medium.com/@roman01la/understanding-transducers-in-javascript-3500d3bd9624)

### notes
