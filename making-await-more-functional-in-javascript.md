---
title: Making Await More Functional in JavaScript
published: true
description: Explores the problems with await (including try/catch "hell") and introduces a library that provides await error handling.
tags: javascript, async, await, functional
date: 2019-08-23
---

In [The Problem with Promises in Javascript](https://dev.to/craigmichaelmartin/the-problem-with-promises-in-javascript-5h46) I looked at how the API and design of promises felt casually dangerous to writing responsible and safe code.

I [included a section](https://dev.to/craigmichaelmartin/making-javascript-promises-more-functional-jp3) proposing a library ([fPromise](https://github.com/craigmichaelmartin/fpromise)) which used a functional approach to overcome these problems.

After it was published,  [Mike Sherov](https://twitter.com/mikesherov) was kind enough to respond to a tweet about the article and offered his take on it: that I under-appreciated the value of the async/async syntax (that it abstracts out the tricky then/catch API, and returns us to "normal" flow) and that the problems that remain (ie, bad error handling) are problems with JavaScript itself (which TC39 is always evolving).

I am very grateful for his thoughts on this, and helping elucidate a counter-narrative to the one I proposed!!

Here's what Mike says:

{% twitter 1164169660973092864 %}

{% twitter 1164180648044716032 %}

{% twitter 1164170411044679682 %}


Lets look at an example from the Problems article:

```javascript
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user)} has been created`);
  } catch {
    createToast(`User could not be saved`));
  }
};
```

I had balked at this, as the try was "catching" too much, and used the point that if `displayName` threw, the user would be alerted that no user was saved, even though it was. But - though the code is a bit monotonous - this is overcome-able - and was a bad job out me for not showing.

If our catch is smart about error handling, this goes away.

```javascript
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user)} has been created`);
  } catch (err) {
    if (err instanceof HTTPError) {
      createToast(`User could not be saved`));
    } else {
      throw err;
    }
  }
};
```

And if the language's evolution includes better error handling, this approach would feel better:

```javascript
// (code includes fictitious catch handling by error type)
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user)} has been created`);
  } catch (HTTPError as err) {
    createToast(`User could not be saved`));
  }
};
```

While this is much better, I still balk about having too much in the try. I believe catch's *should* only catch for the exception they intend to (bad job out of me in the original post), but that the scope of what is being "tried" should be as minimal as possible.

Otherwise, as the code grows, there are catch collisions:

```javascript
// (code includes fictitious catch handling by error type)
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user)} has been created`);
    const mailChimpId = await postUserToMailChimp(user);
  } catch (HTTPError as err) {
    createToast(`Um...`));
  }
};
```

So here is a more narrow approach about what we are catching:

```javascript
// (code includes fictitious catch handling by error type)
const handleSave = async rawUserData => {
  try {
    const user = await saveUser(rawUserData);
    createToast(`User ${displayName(user)} has been created`);
    try {
        const mailChimpId = await postUserToMailChimp(user);
        createToast(`User ${displayName(user)} has been subscribed`);
    } catch (HTTPError as err) {
        createToast(`User could not be subscribed to mailing list`));
    }
  } catch (HTTPError as err) {
    createToast(`User could not be saved`));
  }
};
```

But now we find ourselves in a try/catch block "hell". Lets try to get out of it:

```javascript
// (code includes fictitious catch handling by error type)
const handleSave = async rawUserData => {
  let user;
  try {
    user = await saveUser(rawUserData);
  } catch (HTTPError as err) {
    createToast(`User could not be saved`));
  }
  if (!user) {
    return;
  }
  createToast(`User ${displayName(user)} has been created`);

  let mailChimpId;
  try {
    await postUserToMailChimp(rawUserData);
  } catch (HTTPError as err) {
    createToast(`User could not be subscribed to mailing list`));
  }
  if (!mailChimpId) {
    return;
  }
  createToast(`User ${displayName(user)} has been subscribed`);
};
```

Despite that this is responsible and safe code, it feels the most unreadable and like we're doing something wrong and ugly and working uphill against the language. Also, remember this code is using a succinct fictitious error handler, rather than the even more verbose (real) code of checking the error type and handling else re-throwing it.

Which is (I believe) exactly Mike's point, that error handling (in general) needs improved, and exactly my point - that doing async code with promises is casually dangerous, as it makes dangerous code clean and ergonomic, and responsible code less readable and intuitive.

So, how could this be better? What if there was -

## Await catch handling

What if we could do something like this?

```javascript
// (code includes fictitious await catch handling by error type)
const handleSave = async rawUserData => {
  const [user, httpError] = await saveUser(rawUserData) | HTTPError;
  if (httpError) {
    return createToast(`User could not be saved`));
  }
  createToast(`User ${displayName(user)} has been created`);

  const [id, httpError] = await saveUser(rawUserData) | HTTPError;
  if (httpError) {
    return createToast(`User could not be subscribed to mailing list`));
  }
  createToast(`User ${displayName(user)} has been subscribed`);
};
```

This reads nicely and is safe and responsible! We are catching exactly the error type we intend to. Any other error causes the await to "throw". 

And it could be used with multiple error types. Eg,
```javascript
// (code includes fictitious catch handling by error type)
const [user, foo, bar] = await saveUser(rawUserData) | FooError, BarThing;
```

## How close can we get to this in userland?

Pretty close. Introducing [fAwait](https://github.com/craigmichaelmartin/fawait) (as in functional-await).

```javascript
const {fa} = require('fawait');
const [user, httpError] = await fa(saveUser(rawUserData), HTTPError);
const [user, foo, bar] = await fa(saveUser(rawUserData), FooError, BarThing);
```

Thanks for reading!

{% github craigmichaelmartin/fawait %}
