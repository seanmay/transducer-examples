# Transducing (_in a really, really big nutshell_)

Transducing is a fantastic tool when you become comfortable and competent using `.map` and `.filter` and `.reduce`, and you would also like a free performance boost to go along with your cleaner, simpler code. The performance boost I'm talking about is the one you would get by moving back to using `for` loops; only having to loop over your array once, but getting to use separate map and filter functions, like you were chaining.

If that sounds interesting to you, read on.

### The Simple Case of Map

In the grand scheme, optimizing a chain of `.map` calls is not hard, if our concern is that we are looping over thousands of items multiple times.

```javascript
const lotsOfResults = lotsOfNumbers
  .map(addOne)
  .map(double)
  .map(square)
  .map(toFormattedString);
```
If there were 50,000 items in that array, that would be 200,000 iterations (50,000 &times; 4 calls) to arrive at the final result. In terms of performance, this is nowhere near as bad as ugly inner-loops, or bad recursion. It's linear; the type of stuff you leave off of your general "big-O" estimate, because it matters that little in the grand scheme of performance. But if you have 8 chained map calls, 8n<sup>2</sup> is still worse than 1n<sup>2</sup>. Especially if you improve the n<sup>2</sup> part, and you still need to squeeze more performance out, or you know you are working with hundreds of thousands of iterations, and your core algorithm can't get much better, but you can change how many times you operate.

Regardless, the key to getting all of the `.map` calls for the price of one is to realize the following:
```javascript
xs.map(f).map(g).map(h);

// is exactly the same as
xs.map(x => h(g(f(x))));

// which can be rewritten as
xs.map(compose(h, g, f));
```
This trick is only good if those maps are back-to-back, of course. As soon as you have a filter or a reduce in your chain, you need to let them do their job, before optimizing the next chain of maps.

```js
// 7 loops
xs.map(f).map(g).map(h)
  .filter(isA).filter(isB)
  .map(i).map(j);

// becomes

// 4 loops 
xs.map(compose(h, g, f))
  .filter(isA).filter(isB)
  .map(compose(j, i));
```
We could also do the same for `.filter`, if we had an "and" function that took a bunch of tests and ran each one and `&&` the results together.

```js
// 3 loops
xs.map(compose(h, g, f))
  .filter(x => isA(x) && isB(x)) // some and(isA, isB) helper
  .map(compose(j, i));
```
Using just `.map` and `.filter`, there's nothing we can do to further improve that case. There is still some magic up the `.reduce` sleeve, though.

### Enter Reduce

Reduce (or "Fold"; different, but basically the same) is the overpowered superhero of the FP world. If you weren't already aware, you can replace `.map` with a reducer, and `.filter` with a reducer, and while it might get messy, practically any other operation could be replaced with a reducer (Redux, anyone?). The only challenge is first wrapping our heads around _how_.

```js
const getTotalAge = people =>
  people.reduce((total, person) => {
    const age = person.age;
    return total + age;
  }, 0);

const getListOfAges = people =>
  people.reduce((ages, person) => {
    const age = person.age;
    return ages.concat(age);
  }, []);
```
Those two functions look very similar to me. Like you could replace all of the variable names, and the only difference would be the return statement, and the starting value of the accumulator.

Let's have a look at some other cases:

```js
// addition
const sum = (total, value) => total + value;
// multiplication
const product = (total, value) => total * value;
// joining strings
const joinStrings = (str, segment) => `${str}${segment}`;
// joining arrays
const concatArrays = (a, b) => a.concat(b);
// decisions
const and = (decision, predicate) => decision && predicate;
const or = (decision, alternative) => decision || alternative;
```
All of those reducers look tiny and simple. But I think that there is a pattern worth pointing out. That is, they all follow the pattern of:

```js
(x, y) => doSomething(x, y);
```
and when I look at that, I want to write it as 
```js
f => (x, y) => f(x, y);
```
That sounds completely crazy for these small examples. It would make no sense at all, and if yuu write it out, you'll find that you would just end up repeating the exact same function, in place of `f`...
But what happens if we look at the original examples?
