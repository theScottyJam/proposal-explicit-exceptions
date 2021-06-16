# Explicit Exceptions

## Status

Champion(s): None

Author(s): Scotty Jamison

Stage: -1

## Definitions

To simplify communication, we're going to be using the following definitions throughout this proposal:

**Error:** A runtime bug. The end user should never attempt to recover from these, but may choose to catch them for reporting/logging purposes. An function author may explicitly throw helpful errors, such as `throw new Error('Please provide an object with a "data" property')`, or just let the function trip over itself, causing more cryptic, but equally valid errors to occur, such as `"Cannot read property 'data' of undefined"`.

**Exception:** A specific task may fail with an exception, which the end user can choose to catch and handle if they wish to recover from the failure. Exceptions must have all of the following properties:
* It must be possible to programmatically distinguish one exception from another (i.e. they have an "error code" property, or they're subclassed)
* Exceptions are part of a function's API. The API author should never make changes related to when and which exceptions are thrown, that would cause end-user code to break. If the API author does not wish to guarantee stability, then they should throw a normal error instead (which can't be programmatically distinguished from anything else except through unstable comparisons against the error message, forcing the end-user to acknowledge the instability of their code if they choose to catch and handle it).
* Exceptions should leave the program in a good state, so that recovery can happen. If an API author can't guarantee this, they should throw an error instead.

**Propagating an exception:** Calling a function, receiving an exception, and just forwarding it back to your caller

**Escalating an exception:** Calling a function, receiving an exception, and choosing to throw a generic error in it's place, because you don't want this exception to be part of your function's API (e.g. `throw new Error(myException.message))`. There's a number of reasons to do this, like, you might not wish to support a particular exception as part of this function's API, or, you know the function you called wasn't support to fail because of how you called it.

## Motivation

Javascript's system of invisibly propagating thrown objects works great for programmer errors (it's original purpose), but has lots of issues when it comes to exceptions. The current error system just wasn't built for the kind of failure recovery needs that the community has, as can be seen by the following issues:

1. It's near impossible to figure out what exceptions a particular function might throw. How can you keep the API stable if you don't even know what the API is?

2. It's difficult to refactor code without making breaking changes. For example, say you want to make a public-facing function stop using function f internally, and use g instead, as it better suits what you're trying to achieve. Because it's difficult to know what exceptions f() and g() throw, it will be difficult to know if this will change the API or not, potentially causing end-user code to break.

3. thrown exceptions are difficult to contain and control. Throwing a new exception deep down in the low levels of your project can cause who knows how many of your public-facing functions to start throwing that new exception. The public will stumble into these new exceptions and start relying on them. After all, it's reasonable to expect that a programmatically distinguishable exception is meant to be singled out, caught, and handled. You've accidentally added exceptions to your API (not a breaking change) and can't remove them without it being considered a breaking change. In other words, all API changes should be intentional, weather it's a breaking change or not.

## Description

This proposal seeks to fix the above issues with the current error system in Javascript by:
* Making it clear what exceptions a function can throw, and _enforce_ this behavior.
* Making exception propagation explicit

This is an opt-in feature that project owners can choose to use, if they wish to have added stability within their own project.

To start using this feature, simply declare which exception types you want to be part of your function's API, by putting them in a `throws` declaration. Omitting the `throws` keyword is equivalent to having no exceptions as part of your API.

```javascript
function getUser(id) throws UserNotFoundEx, PermissionDeniedEx {
  ...
}
```

When an instance of a new Exception class is thrown, the callstack will start unwinding. At each step up the call stack, the function signature will be checked to ensure the exception being thrown was expected (it's type was found in the signature). If this particular exception was not listed, then it'll automatically be escalated into an error, right in the middle of this unwinding process, and the throw processes will continue with a generic error instance instead.

For example:

```javascript
// Create our custom exception
class NotFoundEx extends Exception {}

function badInternalLogic(id) throws NotFoundEx {
  // NotFoundEx is thrown
  throw new NotFoundEx('User not found!')
}

function getUser(id) throws NotFoundEx {
  // NotFoundEx continues to propagate from badInternalLogic()
  return badInternalLogic(id)
}

// It's expected that the system user always exists,
// so we don't put NotFoundEx in our function signature.
// (it's encouraged to leave exceptions out when you want them to escalate)
function getSystemUser() {
  // NotFoundEx continues to propagate from getUser()
  // but won't leave this function body, because NotFoundEx
  // is not declared in getSystemUser()'s signature
  return getUser(SYSTEM_USER_ID)
}

function main() {
  // A normal Error instance is received at this point,
  // which contains the same message as the original exception
  // (potentially with an added "Unexpected Exception:" added to it)
  // but all distinguishing properties have been stripped away.
  getSystemUser()
}

main()
```

There may be situations where the same function body might call two different functions which throw the same exception, and you might want to propagate one while escalating the other. This is possible by using the convenient "escalates" keyword after a function call, which lets you list exceptions you feel should be escalated. For example:

```javascript
function editUserUsingSystemUser(userId, edits) throws NotFoundEx {
  // getUser() can throw NotFoundEx
  const user = getUser(userId)

  // The system user should always exist, so we escalate this particular
  // NotFoundEx. If we did not, and the system was misconfigured,
  // then the caller of editUserUsingSystemUser() may end up
  // catching and recover from this exception,
  // thinking that it was the provided userId that did not exist,
  const systemUser = getUser(SYSTEM_USER_ID) escalates NotFoundEx

  ...
}
```

In all of these examples, we've been throwing custom exceptions that have been subclassed from a new Exception class. Both the `throws` and `escalates` keywords only work with instances of this Exception class. The Exception class itself is just a subclass of Error.

## Benefits

Providing exception safety natively in Javascript using the approach described above has a number of benefits, including:
* It's very easy to see and document which exceptions a particular function may throw.
* It's very easy to escalate exceptions that you don't want as part of your API, giving you a lot of control over your own API.
* It's very easy to escalate exceptions that should never be thrown, preventing issues where the end user accidentally recovers from an exception that should have been an error.
* It's very easy to refactor your high-level functions. Looking up which exceptions various function calls provide is now a trivial task, enabling the code writer to better understand the repercussions of making different code changes. This is especially true if you have editor tooling that automatically looks up a function's exception list for you.
* It's easy to add or remove exceptions from lower-level code, without accidentally making changes to higher-level functions or your public facing API.

## Examples

To help get used to the syntax and to see how these concepts can be used in real code, here's a handful of examples demonstrating throwing, catching, handling, and escalating exceptions.

```javascript
class NotFoundEx extends Exception {}
class AccountRemovedEx extends Exception {}

function getUser(userId) throws NotFoundEx, AccountRemovedEx {
  if (!users.has(userId)) {
    throw new NotFoundEx(`User ${userId} not found!`)
  }

  const user = users.get(userId)

  if (user.deleted) {
    throw new AccountRemovedEx()
  }

  return user
}

function getUserOrDefault(userId, defaultValue) throws AccountRemovedEx {
  try {
    return getUser(userId)
  } catch (ex) {
    if (ex instanceof NotFoundEx) {
      return defaultValue
    }
    throw ex
  }
}

function getSystemUser() throws AccountRemovedEx {
  // The NotFoundEx will be auto-escalated, because
  // the system user should always exist.
  return getUser(SYSTEM_USER_ID)
}

function setUser(userId, newUserData) throws NotFoundEx, AccountRemovedEx {
  ...
}

function updateUser(userId, propName, newValue) throws NotFoundEx, AccountRemovedEx {
  const user = getUser(userId)
  user[propName] = newValue
  // It should be impossible for setUser() to throw NotFoundEx.
  // We know this user exists, we just fetched them.
  setUser(userId, user) escalates NotFoundEx
}

// All exceptions will be escalated, because the exception
// might occur in the middle of the updates, and the system
// might get left in a bad state.
function updateUsers(userIds, propName, newValue) {
  for (const userId of userIds) {
    const user = getUser(userId)
    user[propName] = newValue
    setUser(userId, user)
  }
}
```

## Comparison

### General Differences

Exception safety is generally a feature found in typesafe languages, because it is the type system that provides exception safety _but it doesn't have to be this way_. The concept of automatic exception escalation (exceptions auto-escalate when they're not found in the function signature) is the lifeblood of this proposal, and provides the needed machinery to bring exception safety into dynamically-typed languages like Javascript. Because this proposal uses unique machinery to provide exception safety, it can offer certain benefits that are unique to this proposal, which other typesafe languages do not offer. Some of these benefits include:
* It's very easy to escalate exceptions. Most languages require you to single-out an exception and manually escalate it.
* It's easy to start throwing new exceptions, without counting it as a breaking change. Type-safe languages provide exception safety by forcing you to acknowledge _all_ exceptions a function might throw/return, which in turn makes any changes to what a function throws/returns a breaking change. Some languages are worse than others at this point, but generally you can't just add a new exception in one function without being required to update the consumers of this function.

These benefits show that exception safety can actually be more user-friendly when implemented with some runtime behavior. Note that there will still be room for improvement by languages such as TypeScript - see footnote #1 for more details.

### Comparing against Java

The Java developers among you may have noticed some striking similarities between the this proposal and Java's checked exceptions. It's important to note that while syntactically they look similar, the underlying machinery is very different, allowing us to sidestep many of the criticisms that face Java's exception system today.

Here's a small example of Java's checked exceptions:

```java
public String getUserInput() throws IOException {
  ...
}
```

Any exceptions you want to force the caller to explicitly handle must go into the function signature, and is called a "checked exception". Anything else is an unchecked exception. The type checker will then force all callers to explicitly handle the exception, manually escalate it (by throwing a RuntimeException, which is unchecked), or propagate it by adding it to your function signature.

Here are some of the major criticisms of checked exceptions, and how the proposed explicit-exception system avoids these issues:

1. Adding a new exception to a type signature is considered a breaking change in Java, and all client code has to update. Similarly, removing an exception type is also considered a breaking change.

    We'll just focus on the "add new exception" case here, but the exact same logic applies for removing exceptions too. The issue behind Java's philosophy is that causing a function to throw a new exception is often not a breaking change, depending on the nature of the change. For example, if the change caused something that used to be an error to now be an exception, then nothing breaks if the client doesn't handle it. The proposed system handles this issue by only requiring the caller to acknowledge exceptions that they wish to handle or propagate, everything else is automatically escalated. If a particular function starts throwing an exception where an error used to be thrown, then callers will remain unaffected because that exception will escalate right back into an error unless the caller chooses to catch or propagate it. For the scenarios when adding an exception really is a breaking change (e.g. you made something that used to work correctly throw an exception instead), we'll let the API designers handle this the same way they handle other type of breaking change - they need to communicate this change to the user, or deprecate/remove the entire function and make a new function.

2. Checked exceptions may force users to manually escalate exceptions they know shouldn't ever happen in the first place.

    Many Java developers will recognize the common pattern of catching an exception, just to manually escalate it as an unchecked exception like this `throw new RuntimeException(caughtCheckedException)`. This is done because the particular API you're using is forcing you to acknowledge this exception (it's a checked exception), and yet, you know you can't handle it, or, you know that it shouldn't ever get thrown because of how you're using the API. This extra verbosity can be frustrating and can make some libraries annoying to use. Issues such as this has caused the common advice "limit how many checked exceptions your library throws" to get circulated around, because you want to keep your library user-friendly. None of this applies in the current proposal. You can freely throw as many exceptions as you want, and the caller can simply ignore their existence to cause them to be escalated.

3. Checked exceptions are verbose

    A safer exception system will always require the programmer to explicitly state their intentions in their code, but this isn't necessarily a bad thing. We've tried to structure the proposal in a way to reduce the biggest verbosity pain points. Feedback is always welcome if you still feel like the syntax is still too clumsy, and have ideas to improve it further. Also keep in mind that this explicit-exception proposal is _opt-in_, users who wish to quickly prototype a project can ignore this feature entirely, sacrificing stability for a quicker, initial development speed. Even third-party libraries can choose to just throw whatever they want (for backwards compatibility purposes, or because they don't want to opt-in) - There's no disadvantage for a library consumer to use a function that throws custom exceptions instead these proposed exceptions. See Footnote #2 to learn why.

4. Checked exceptions integrate poorly with higher-order functions.

    Javascript is full of higher-order functions, so we want to make sure we address this in a good, clean way. This will involve some yet-to-happen in-depth discussion on how to best achieve a solution to this problem, afterwhich we'll update this section and explain what the plan will be.

### Comparing against functional languages

Many languages from the function family (such as Haskell, ReScript, Rust, etc) use the "either monad" to deal with exceptions. Here are a handful of examples of how the either monad looks in practice.

**Haskell**
```haskell
data DivisionEx = BottomIsZero | TopAndBottomAreZero

divide :: Double -> Double -> Either DivisionEx Double
divide 0 0 = Left TopAndBottomAreZero
divide _ 0 = Left BottomIsZero
divide top bottom = Right (top / bottom)

responseAsString (Left BottomIsZero) = "Bottom is zero"
responseAsString (Left TopAndBottomAreZero) = "Both are zero"
responseAsString (Right n) = show n

main = putStrLn $ responseAsString $ divide 2 4
```

**rescript**
```re-script
type divisionEx =
  | BottomIsZero
  | TopAndBottomAreZero

let overlySafeDivide = (top: float, bottom: float): result<float, divisionEx> => {
  switch (top, bottom) {
    | (0.0, 0.0) => Error(TopAndBottomAreZero)
    | (_, 0.0) => Error(BottomIsZero)
    | _ => Ok(top /. bottom)
  }
}

Js.Console.log(switch (overlySafeDivide(2.0, 4.0)) {
  | Error(TopAndBottomAreZero) => "Both are zero"
  | Error(BottomIsZero) => "Bottom is zero"
  | Ok(x) => x -> Js.Float.toString
})
```

In these languages, exceptions are returned instead of thrown. Often, an [algebraic data type](https://en.wikipedia.org/wiki/Algebraic_data_type) is used to allow an end-user to easily distinguish one exception from another, and to allow the type system to help out. While this system is often preferred over Java, it still comes with a few rough patches:
* Due to how the type system works in most implementations, there normally is not a way to aggregate exceptions from multiple sources (see [this stackoverflow question](https://stackoverflow.com/questions/61851984/how-can-i-compose-error-types-in-haskell)). For example, if we ignore a few non-friendly workarounds, one can't make a function that returns either an exception from the IO family of exceptions, or some other DivisionByZero exception.
* It forces you to put your exception handling logic right in the middle of your normal code, which can break the flow of the logic.
* It can be a little verbose to escalate unwanted exceptions
* Causing functions to start throwing new exceptions, or stop throwing a particular one is a breaking change if the exceptions are ever used in the language's match construct without a default case (a common scenario).

It's felt that the current proposal will provide almost the same power as the either monad (in regards to exception safety), but with less verbosity, and without the pitfalls of Java's exception system. However, specifically trying to integrate the automatic-exception-propagation into an either-monad type of exception system has not yet been discussed, and would be an important avenue to explore in the future. Note that simply returning exceptions in the current version of Javascript has been discussed - the shortcomings of this approach are noted in footnote #3.

## Implementation

The proposed Exception class's implementation will roughly correspond to the following Javascript:

```javascript
class Exception extends Error {
  constructor(code, message = code) {
    super(message)
    this.code = code
    this.name = this.constructor.name
  }
}
```

Note that setting `this.name` to `this.constructor.name` will allow users to subclass the exception, and have it automatically take on the name of the subclass. We want this Exception class to be as user-friendly to subclass as possible.

Remember that A function without a `throws` declaration is equivalent to one with an empty `throws` (i.e. all exceptions are escalated). These functions are roughly transpiled as follows:

```javascript
// before
function getUser(userId) throws NotFound, ServiceUnavailable {
  f()
  return g()
}

// after
function getUser(userId) {
  try {
    f()
    return g()
  } catch (ex) {
    if (!(ex instanceof Exception)) {
      // Not an exception, propagate it
      throw ex
    } else if (ex instanceof NotFound || ex instanceof ServiceUnavailable) {
      // An exception declared in "throws", propagate it
      throw ex
    } else {
      // Escalate it
      throw new Error(`Unexpected exception: ${ex.message}`)
    }
  }
}
```

The `escalates` keyword is roughly transpiled as follows:

```javascript
// before
f() escalates NotFound, ServiceUnavailable

// after
try {
  f()
} catch (ex) {
  if (ex instanceof NotFound || ex instanceof ServiceUnavailable) {
    // Escalate it
    throw new Error(`Unexpected exception: ${ex.message}`)
  } else {
    // Let it pass through
    throw ex
  }
}
```

Weather an escalated exception should create a fresh stack trace from the point of escalation, or copy over the stack trace from the original exception is an open question.

## Potential future improvements

Handling a single exception will be a common task, and currently it's a bit of a chore to do due to the fact that you have to catch all errors, single out what you want, and rethrow anything else. Here's an example syntax that we could choose to pursue to simplify this task:

```javascript
try {
  f()
} catch NotFound, Unavailable (ex) {
  // Handle either of these exceptions
  // Any other errors/exceptions will continue to be propagated.
}
```

## Footnotes

1. **What type-safety can add**

    There's an on-going [typescript thread](https://github.com/microsoft/TypeScript/issues/13219) discussing what it would look like to add exception safety to typescript. There seems to be a lot of support for the idea, but difficulty deciding how to proceed. As previously discussed, typesafe exception systems (including what's being proposed in the typescript thread) usually provide exception safety by requiring users to explicitly treat _all_ exceptions, while this proposal provides exception safety through automatic escalation. The addition of this proposal won't steal the thunder from TypeScript and friends, they will still be able to build on top of the proposed runtime system to make an even better exception system then what was previously possible.

    If typescript were to follow through and implement exception safety as it's currently proposed, users would be required to manually escalate exceptions every time they don't want to deal with them - this is one of the dislikes of Java's checked exception system, and in general, one of the verbosity issues that people dislike about exception safety. If Javascript natively supports automatic-escalation, then typeScript would be able to add type-safety to this runtime behavior instead, adding additional benefits such as the below points, without the extra verbosity of manual exception escalation that all other safe-exception systems require.
      * Type safety can ensure that you're not accidentally adding something extra to the throws declaration that can't be thrown.
      * Type safety can ensure that you're not using the "escalates" keyword to escalate something that'll never be thrown.

2. **Oh great, now I've got to update my library to support this?**

    Not so fast! As it turns out, it actually does not benefit the end-user if your third-party library throws exceptions from this proposed exception system or not, due to the way we tend to translate exceptions across domain boundaries. The proper way to handle exceptions from a third-party library is right at the call site. If the user wishes to continue to propagate the exception, they should translate it into their own custom exception that's specific to domain they're operating in, before passing it on (e.g. they might turn a database's EntryNotFound exception into a custom UserNotFound one). Any exceptions that aren't handled at the call site are effectively treated as escalated exceptions, because the user should never handle third-party exceptions at any location but the call site. Notice that this process of translating exceptions at the call site actually brings all of the benefits of an explicit-exception system? They're explicitly listing the exceptions they want to propagate or handle, and letting everything else get "escalated". Because of this, there's no real advantage to having your third-party library throw subclassed instances of Exception from this proposal over any other kind of exception.

3. **Can't you get exception safety today by simply returning exceptions instead of throwing them?**

    No, not in a type-unsafe language. All that would happen is that you would know that a particular function provides exceptions, but you have know way of knowing what exceptions it provides, unless those particular exceptions are being handled on the spot. It's also pretty verbose to use this system. Here's an example:

    ```javascript
    // This is using a return-exception system, that's possible today.
    // Notice there's no easy way to tell what exceptions can be returned
    function readConfigFile(confPath) {
      const { ex: resolveEx, value: newPath } = resolvePath('./config/', confPath)
      if (resolveEx) return { ex: resolveEx, value: null }
      return readFile(newPath) // Also returns an object of the shape { ex: ..., value: ... }
    }

    // This throws exceptions using the proposed exception system
    function readConfigFile(confPath) throws InvalidPath, FileNotFound, PermissionsDenied {
      return readFile(path.resolve('./config/', confPath))
    }
    ```