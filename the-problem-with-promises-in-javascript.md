---
title: The Problem with Promises in JavaScript
published: true
description: Promises co-mingle rejected promises and unintended runtime exceptions, while having an API which encourages casually dangerous code and which lacks a convenient API to work safely with the data. 
tags: promises, javascript, async, await
date: 2019-08-16
---

Spending a lot of time in Node recently, I keep coming accross 3 recurring problems with promises:
- Promises have an API which encourages casually dangerous code
- Promises lack a convenient API to safely work with data.
- Promises co-mingle rejected promises and unintended runtime exceptions

While the `await` syntax is happy addition to the language, and part of a solution to these problems, its value - increasing readability and keeping control in the original callstack (ie allowing for returns) - is unrelated to the second two issues, and only sometimes mitigating of the first problem.


## Promises have an API which encourages casually dangerous code.

Lets take an example of saving a user:

```javascript
// Promises (without using await)
// Casually dangerous code
const handleSave = rawUserData => {
  saveUser(rawUserData)
    .then(user => createToast(`User ${displayName(user)} has been created`))
    .catch(err => createToast(`User could not be saved`));
};
```

This code looks readable and explicit: a clearly defined path for success and for failure.

However, while trying to be explicit, we have attached our `catch` not just to the `saveUser` request, but also to the success path. Thus, if the then throws (eg, the displayName function throws) then the user will be notified that no user was saved, even though it was.

> One might think, lets switch the order of the then/catch, so that the catch is attached to the saveUser call directly. This introduces another issue, we'll look at further below (as the third issue).

Using await doesn't necessarily help. It is agnostic to using the API correctly, and because of its block scoping it also makes it easier and prettier to write it dangerously as above:


```javascript
// Promises with Async/Await doesn't necessarily help
// Casually dangerous code
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user) has been created`);
  } catch {
    createToast(`User could not be saved`));
  }
};
```

Because of the block scoping, it is more convenient to include the createToast line in the try, but then this code has the same issue as above.

The responsible refactor of this using native promises _looks_ worse/ugly/bad/complicated. Lets look at the case of not using `await` first.

For the case of not using `await`, two anonymous functions in the correct order (error function first? success function first?) must be passed to the then, which feels less organized than using an explicit `catch` block:

```javascript
// Promises done responsibly _look_ worse/ugly/bad/complicated :(
const handleSave = rawUserData => {
  saveUser(rawUserData)
    .then(
      user => createToast(`User ${displayName(user)} has been created`),
      err => createToast(`User could not be saved`));
    );
};
```

To be clear, this isn't a bad API in itself. But considering the rightful intention of being explicit as a developer, there is a temptation of using a named function for each, rather than one `then` with the two callbacks. The responsible code is less explicit and readable than dangerous code - **it is temptingly dangerous to misuse the API - while feeling more explicit and readable!**

The responsible refactor using `async`/`await` _looks even more so_ wrong/ugly/bad/complicated. Having to define variables in a higher scope feels like a bad control flow. It feels like we're working against the language:

```javascript
// Promises done responsibly _look_ worse/ugly/bad/complicated :(
const handleSave = async rawUserData => {
  let user;
  try {
    user = await saveUser(rawUserData);
  } catch {
    createToast(`User could not be saved`));
  }
  createToast(`User ${displayName(user)} has been created`);
};
```

Notice the code above isn't even correct. We'd need to return from the `catch` (something I try to avoid as it further confuses control flow - especially if there is a finally) or wrap everything after the try if an `if (user) { /*...*/ }` block - creating another block. It feels like we're working uphill.


<hr>


It's also worth noting that the API is _also_ unintuitive (but this time the other way!) when chaining multiple `then`s.

Whereas the examples above are dangerous because the `catch` is meant to be attached to the "root" async call (the HTTP request) - **there is also a danger with long chains of thinking the `catch` is associated with the most recent then.**

(It's neither attached to the root promise nor the most recent promise - it is attached to the entire chain preceding it.)

For example:

```javascript
// Casually dangerous code
const userPostHandler = rawUserData => {
  saveUser(rawUserData)
    .then(sendWelcomeEmail)
    .catch(queueWelcomeEmailForLaterAttempt)
};
```

which looks and reads cleanly, compared to the responsible:

```javascript
// Promises done responsibly _look_ worse/ugly/bad/complicated :(
const userPostHandler = rawUserData => {
  saveUser(rawUserData)
    .then(user =>
      sendWelcomeEmail(user)
        .catch(queueWelcomeEmailForLaterAttempt)
    );
};
```

<hr>

Lets go further with the example above, to see one last way the API is casually dangerous: lets add logging for if the user can't be created:

```javascript
// Dangerous code
const userPostHandler = rawUserData => {
  saveUser(rawUserData)
    .catch(writeIssueToLog)
    .then(sendWelcomeEmail)
    .catch(queueWelcomeEmailForLaterAttempt)
};
```

What we want is to write the issue to our logs if the user save fails.

However, because our catch doesn't re-throw or explicitly reject, it returns a resolved promise and so the next then (sendWelcomeEmail) will run, and because there is no user, it will throw, and we'll create a queued email for a non-existing user. 

**The casual promise API makes unintentionally recovering from an exception easy/sleek/elegant.**

Again, the fix looks bad:

```javascript
// Promises done responsibly _look_ worse/ugly/bad/complicated :(
const userPostHandler = rawUserData => {
  saveUser(rawUserData)
    .then(
      writeIssueToLog,
      user =>
          sendWelcomeEmail(user)
            .catch(queueWelcomeEmailForLaterAttempt)
      );
};
```

Wrapping up this section, we've seen how promise's API for handling errors while seemingly sleek, is casually dangerous: both due to the readability and convenience of catching separately from the `then` (ie, using an explicit catch function - which if in a chain includes errors not just from the "root" promise, nor from the most recent promise, but from any promise in the chain), as well as by fostering an unintentional recovery of errors.

While the addition of the `async` operator can help, it does so within a try scope - making the right code look disjointed, and irresponsible code (placing too much in the try) look cleaner/sleeker.

I would prefer an API which at a minimum optimizes aesthetics and readability (by working with the language) for the responsible behavior, and preferably which precludes irresponsible or casually dangerous code.

## Promises lack a convenient API to safely work with data.

In the section above, we looked at how the existing promise API is temptingly dangerous (using two explicit named functions vs one with anonymous parameters for each function), and how it fosters unintentionally recovering from errors.

This second case is a problem only because the promise API doesn't offer more helpers.

In the last example above where our `.catch(logError)` inadvertently resolved the error, what we were really wanting was something else: a `tap` side-effect function for errors.

## Promises co-mingle rejected promises and unintended runtime exceptions

Apart from how the API is structured - promises have another major flaw: **they treat unintentional native runtime exceptions and intentional rejected promises - which are two drastically different intentions - in the same "path".**

```javascript
const userPostHandler = rawUserData => {
  saveUser(userData)
    .then(() => response.send(204))
    .then({email} => postEmailToMailChimp(email))
    .catch(logError)
};
```

What this code is trying to express is pretty straightforward. (I want to save a user and post their email to my mailchimp list and log if there is an issue).

However, I accidentally typo'd the function name as "MailChimp" instead of "Mailchimp" - and rather than the runtime error alerting me while developing - I now have to hope that I look at the log - which I intended for mailchimp issues, not basic programming issues!

In explaining the root issue here with promises, I abbreviated the behavior slightly: promises treat all errors (not just native errors) the same as rejected promises. Treating `throw` and `Promise.reject` synonymously seems reasonable. What does not seem reasonable is using this one "path" to handle two worlds-different "types" of errors without distinction: "strategic" errors (eg `saveUser(user)` throwing a custom Integrity error), and basic javascript runtime errors (eg saveUsr(user) having a typo and throwing a ReferenceError). These are two fundamentally different realties, but they are bundled together in the same "rejected promise" path.

With promises, there are really three paths: the data "path", a non-native error "path" (eg, custom, business-logic errors), and a native error "path", yet the API does not make this distinction: and treats all errors and rejected promises the same.

<hr>

[Two updates]

[Update] This article previously continued with a theoretical section on what "better" Promises might look like... "What comes next is one (out of an infinite number of solutions to these problems - and probably a really bad one) thought experiment on what might be a solution.. which turned into a library." If you're interested you can see read it here, [Making JavaScript Promises More Functional](https://dev.to/craigmichaelmartin/making-javascript-promises-more-functional-jp3)

[Update] [Mike Sherov](https://twitter.com/mikesherov) was kind enough to respond to a tweet about this article and offered his take on this: that I under-appreciated the value of the `async`/`async` syntax (that it abstracts out the tricky `then`/`catch` API, and returns us to "normal" flow) and that the problems that remain (ie, bad error handling) are problems with JavaScript itself (which TC39 is always evolving). I'm midway through an article about this (which will include Mike's tweets to give him his full voice) and will provide a link when done. Thanks, Mike, for your thoughts on this, and helping elucidate a counter-narrative to this one!!
