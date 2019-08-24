---
title: Making JavaScript Promises More Functional
published: true
description: A thought-experiment-turned-library on how to overcome some of the problems with promises in javascript using a functional approach.
tags: promises, javascript, async, await
date: 2019-08-16
---

[_This article was extracted out of [The Problem with Promises in JavaScript](https://dev.to/craigmichaelmartin/the-problem-with-promises-in-javascript-5h46). It was the final section, but given that it is only one possible solution to the problems enumerated, thought it should live separately. After the short introduction, it is unedited from when it was the final section of the mentioned article._]

A few years ago I created a new repo for a Node-backend web app, and spent some time considering how to deal with promises in my code. In other Node side-projects, I had begun to see some recurring issues with promises: that the API seemed to have the best ergonomics when used dangerously, that they lacked a convenient API to safely work with data, and that rejected promises and unintended runtime exceptions were co-mingled and left for the developer to sort out.

You can read more about these issues in [The Problem with Promises in JavaScript](https://dev.to/craigmichaelmartin/the-problem-with-promises-in-javascript-5h46).

This article is one (out of an infinite number of solutions to these problems - and probably a really bad one) thought experiment on what might be a solution.. which turned into a library: [fPromise](https://github.com/craigmichaelmartin/fpromise)

{% github craigmichaelmartin/fpromise %}

If you haven't read the [The Problem with Promises in JavaScript](https://dev.to/craigmichaelmartin/the-problem-with-promises-in-javascript-5h46), you may want to.

So, lets begin from a thought experiment about what better promises might have looked like, and see if we can get there in userland code. By "better" - I mean immune to the problems above.

# What might a "better" Promise implementation look like?

It feels right that `await` throws for native exceptions (just like regularly synchronous code would). What is not ideal is that non-native errors are in that bucket, and so have to be caught, and with the new block scopes decreasing readability and making the code more disjointed. 

Imagine if promises used rejected promises only for native runtime exceptions, and used a special object for data/issues. Lets call that special object an Either. It is iterable to a two element array with data as the first element, issue as the second. To our earlier point, it also specifies methods like map/imap (issue map) and tap/itap (issue tap) which its two implementations (Data and Issue) implement. Data has no-ops for imap and itap. Issue has no-ops for map and tap. `map`/`imap` re-wrap the result as Data/Issue respectively, unless explicitly transformed to the other. The tap methods are side-effect only who's returns are not used.

Promise.resolve creates a "regular" promise wrapping the value in Data. Promise.reject creates a "regular" promise wrapping the value in Issue _if_ the reject is not a native error; otherwise, it creates an actually "rejected" promise.

We could write code like:

```javascript
// Made up API below!

// data-access/user.js
const save = user => db.execute(user.getInsertSQL());
// As long as there is no native Exceptions, this returns a
// promise in the "regular" state.

// service/user.js
const save = data =>
  save(User(data))
    .tap(getStandardLog('user_creation'))   // Fictional
    .map(User.parseUserFromDB)              // Fictional
    .itap(logError);                        // Fictional

// controllers/user.js
const postHandler = async (userDate, response) => {
  // No need to use try/catch, as everything is in the "regular" state
  const [user, error] = await save(userData);  // Fictional
  if (error) {
    const errorToCode = { 'IntegrityError': 422 }; 
    return response.send(errorToCode[error.constructor.name] || 400);
  }
  response.send(204);
  postEmailToMailChimp(user.email).tapError(logError);
};
```

Features of this approach:
- rejected promises are only used for native exceptions, so no need to use a try/catch block - more readable, cohesive code. Everything else is in the "regular" path, but as a Data or Issue.
- `map`, `tap`, `itap` helper utilities which apply the functions to "regular" path promise values. (Remember, map/tap are no-ops on Error, imap/itap are no-ops on Data.)
- "regular" promises values (Data|Either) destructure to an array with the data or issue (but, again, never native runtime errors - those throw (and could here be caught in a try/catch, but no one programs for that level of fear: eg `try { Math.random() } catch (err) { console.log('Just in case I typo-ed the string "Math" } `))
- `await` allows us to stay in the callstack (allowing return)

**This to me feels like promises done right.**

## How close can we get to the above code?

We can actually get pretty close.

We'll
- [x] use promises
- [x] leave the promise prototype untouched
- [x] provide a safe API for using them which isn't casually dangerous
- [x] ensure unintentional runtime errors are not handled (and so throw when awaited)
- [x] provide utility methods for working with the data
- [x] increase readability/cohesion (vs try blocks)
- [x] keeps control in main call block (so returns work)

By providing a safe API within the Promise structure, this "library" we'll make can be used anywhere promises are, without hijacking the prototype or needing to introduce a new primitive.

We'll create an Either type which specify
- `map`
- `imap`
- `tap`
- `itap`
- etc

and ensures it is iterable (destructure-able) to a two element array.

`Data` and `Issue` implement this Either interface.

```javascript
const Data = x => ({
  map: f => Data(f(x)),          // transform the data by applying the fn
  imap: f => Data(x),            // no-op (method targets Issue)
  bmap: (f, g) => Data(f(x)),    // run respective fn on data
  tap: f => (f(x), Data(x)),     // runs side effect fn on data
  itap: f => Data(x),            // no-op (method targets Issue)
  btap: (f, g) => (f(x), Data(x)),// run respective sideeffect fn on data
  val: () => [x],
  isData: true,
  isIssue: false,
  [Symbol.iterator]: function *() { yield x; }
});

const Issue = x => ({
  map: f => Issue(x),            // no-op (method targets Data)
  imap: f => Issue(f(x)),        // transform the issue by applyin the fn
  bmap: (f, g) => Issue(g(x)),   // run respective fn on issue
  tap: f => Issue(x),            // no-op (method target Data)
  itap: f => (f(x), Issue(x)),   // runs side effect fn on issue
  btap: (f, g) => (g(x), Issue(x)),//run respective sideeffect f on issue
  val: () => [, x],
  isData: false,
  isIssue: true,
  [Symbol.iterator]: function *() { yield void 0; yield x; }
});
```

We'll need an `fp` which transforms a current promise to play by our safe rules.

```javascript
const ensureData = data =>
  data instanceof Data ? data : Data(data);

const nativeExceptions = [ EvalError, RangeError, ReferenceError, SyntaxError, TypeError, URIError ];

const ensureIssue = error => {
  if (error instanceof nativeException) {
    throw error;
  }
  return error instanceof Error ? error : Error(error);
};

const fp = promise => promise.then(ensureData, ensureIssue);
```

To make these more functional, we could also add:

```javascript
const map = f => [o => ensureData(o).map(f), o => ensureIssue(o).map(f)];
const imap = f => [o => ensureData(o).imap(f), o => ensureIssue(o).imap(f)];
const bmap = (f, g) => [o => ensureData(o).bmap(f, g), o => ensureIssue(o).bmap(f, g)];
const tap = f => [o => ensureData(o).tap(f), o => ensureIssue(o).tap(f)];
const itap = f => [o => ensureData(o).itap(f), o => ensureIssue(o).itap(f)];
const btap = (f, g) => [o => ensureData(o).btap(f, g), o => ensureIssue(o).btap(f, g)];
```


To re-write the fictional promise code from above, it's pretty straight forward. We:
1. wrap the initial promise with a `fp` to get the promise to play by our rules (again, it remains a completely regular promise).
2. (await promise) before we can call our utility methods. This is because our utility methods are on the Either that the promise resolves to, not the promise itself. To the point above, we are not touching/modifying promises, just layering on top of them.


```javascript
// data-access/user.js
const save = user => fp(db.execute(user.getInsertSQL()))

// service/user.js
const save = async data =>
  (await save(User(data)))
    .tap(getStandardLog('user_creation))
    .map(User.parseUserFromDB)
    .itap(logError)

// controllers/user.js
const postHandler = async (userDate, response) => {
  const [user, error] = await save(userData);
  // ...
}
```

If we wanted to use the more functional approach, no need for initially wrapping the promise:


```javascript
// data-access/user.js
const save = user => db.execute(user.getInsertSQL();

// service/user.js
const save = data => save(data)
  .then(...tap(getStandardLog('user_creation)))
  .then(...map(User.parseUserFromDB))
  .then(...itap(logError))

// controllers/user.js
const postHandler = async (userDate, response) => {
  const [user, error] = await save(userData);
  // ...
}
```

Notice for both of these, all of are conditions are met. We are:
- [x] using promises
- [x] leave the promise prototype untouched
- [x] provide a safe API for using them which isn't casually dangerous
- [x] ensures unintentional runtime errors are not handled
- [x] provides utility methods for working with the data
- [x] increases readability (vs try blocks)
- [x] keeps control in main call block (so returns work)


If we want to move even further in the functional direction, we could:


```javascript
// data-access/user.js
const save = user => db.execute(user.getInsertSQL();

// service/user.js
const save = data => save(data)
  .then(...tap(getStandardLog('user_creation')))
  .then(...map(User.parseUserFromDB))
  .then(...itap(logError))

// controllers/user.js
const postHandler = (userDate, response) =>
  save(userData).then(...map(
    user => //...
    error => //...
  );
```


If you're interested in this fPromise idea, [help with it on github](https://github.com/craigmichaelmartin/fpromise)

{% github craigmichaelmartin/fpromise %}

or check out similar-


# _Actually Good_ Projects in this space

- https://gist.github.com/DavidWells/56089265ab613a1f29eabca9fc68a3c6
- https://github.com/gunar/go-for-it
- https://github.com/majgis/catchify
- https://github.com/scopsy/await-to-js
- https://github.com/fluture-js/Fluture
- https://github.com/russellmcc/fantasydo

# Articles About This Stuff From Smart People:
- https://medium.com/@gunar/async-control-flow-without-exceptions-nor-monads-b19af2acc553
- https://blog.grossman.io/how-to-write-async-await-without-try-catch-blocks-in-javascript/
- http://jessewarden.com/2017/11/easier-error-handling-using-asyncawait.html
- https://medium.freecodecamp.org/avoiding-the-async-await-hell-c77a0fb71c4c
- https://medium.com/@dominic.mayers/async-await-without-promises-725e15e1b639
- https://medium.com/@dominic.mayers/on-one-hand-the-async-await-framework-avoid-the-use-of-callbacks-to-define-the-main-flow-in-812317d19285
- https://dev.to/sadarshannaiynar/capture-error-and-data-in-async-await-without-try-catch-1no2
- https://medium.com/@pyrolistical/the-hard-error-handling-case-made-easy-with-async-await-597fd4b908b1
- https://gist.github.com/woudsma/fe8598b1f41453208f0661f90ecdb98b
