---
layout: post
title: "How async should have been"
description: ""
date: 2020-06-07
tags: python
writing_time:
  writing: "0"
  proofreading: "0"
  decorating: "0"
republished: []
---

In the last few years `async` keyword and semantics made its way into many popular programming languages: [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function), [Rust](https://doc.rust-lang.org/std/keyword.async.html), [C#](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await), and [many others](https://en.wikipedia.org/wiki/Async/await) languages that I don't know or don't use.

Of course, Python also has `async` and `await` keywords since `python3.5`.

In this article, I would love to provide my opinion about this feature, think of alternatives, and provide a new solution.


## Colours of functions

When introducing `async` functions into the languages, we actually end up with a split world. Now, one functions start to be red (or `async`) and old ones continue to be blue (sync).

The thing about this division is that blue functions cannot call red ones.
Red ones potentially can call blue ones. In Python, for example, it is partially true. Async functions can only call sync non-blocking functions. Is it possible to tell whether this function is blocking or not by its definition? Of course not! Python is a scripting language, don't forget about that!

This division creates two subsets of a single language: sync and async ones.
5 years passed since the release of `python3.5`, but `async` support is not even near to what we have in the sync python world.

Read [this brilliant piece](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) if you want to learn more about colors of functions.


## Code duplication

Different colors of functions lead to a more practical problem: code duplication.

Imagine, that you are writing a CLI tool to fetch sizes of web pages. And you want to support both sync and async ways of doing it.

Let's start with the sync pseudo-code:

```python
def fetch_resource_size((url: str) -> str:
    response = client_get(url)
    response.raise_for_status()
    return len(response.content)
```

Looking pretty good! Now, let's add its async counterpart:

```python
async def fetch_resource_size(url: str) -> str:
    response = await client_get(url)
    response.raise_for_status()
    return len(await response.content)
```

It is basically the same code, but filled with `await` keywords!
And I am not making this up, just compare code sample in `httpx` tutorial:

- [Sync](https://www.python-httpx.org/quickstart/) code
- vs [Async](https://www.python-httpx.org/async/) code

They show exactly the same picture.


## Abstraction and Composition

Ok, we find ourselves in a situation where we need to rewrite all sync code and add `async` and `await` keywords here and there, so our program would become asynchronous.

These two principles can help us in solving this problem.

First of all, let's rewrite our imperative pseudo-code into a functional pseudo-code. This will allow us to see the pattern more clearly:

```python
def fetch_resource_size(url: str) -> Abstraction[str]:
    return client_get(
        url,
    ).map(
        lambda response: response.raise_for_status(),
    ).map(
        lambda response: len(response.content),
    )
```

Now the steps are pretty clear! Notice how easily we went from several independent steps into a single pipeline of function calls. We have also changed the return type of this function: now it returns some `Abstraction`.

Now, let's rewrite our code to work with async version:

```python
def fetch_resource_size(url: str) -> AsyncAbstraction[str]:
    return client_get(
        url,
    ).map(
        lambda response: response.raise_for_status(),
    ).map(
        lambda response: len(response.content),
    )
```

Wow, that's mostly it! The only thing that is different is the `AsyncAbstraction` return type. Other than that, our code stayed exactly the same. We also don't need to use `async` and `await` keyword anymore. We don't use `await` at all (that's the whole point of our journey!), and `async` functions do not make any sense without `await`.

The last thing we need is to decide which client we want: async or sync one. Let's fix that!

```python
def fetch_resource_size(
    client_get: Callable[[str], AbstactionType[Response]],
    url: str,
) -> AbstactionType[str]:
    return client_get(
        url,
    ).map(
        lambda response: response.raise_for_status(),
    ).map(
        lambda response: len(response.content),
    )
```

Our `client_get` is now an argument of a callable type that receives a single URL string as an input and returns some `AbstractionType` over `Response` object. This `AbstractionType` is either `Abstraction` or `AsyncAbstraction` we have already seen on the previous samples.

When we pass `Abstraction` our code works like a sync one, when `AsyncAbstraction` is passed, the same code automatically starts to work asynchronously.


## IOResult and FutureResult

Luckily, we already have the right abstractions in [`dry-python/returns`](https://github.com/dry-python/returns)!

Let me introduce to you type-safe, `mypy`-friendly, framework-independent, pure-python tool to provide you awesome abstractions you can use in any project!

### Sync version

One can rewrite this pseudo-code as real python code. Let's start with the sync version:

# TODO: explain what `.map` means
# TODO: add `pip install` command

```python
from typing import Callable

import httpx

from returns.functions import tap
from returns.io import IOResultE, impure_safe
from returns.result import safe

def fetch_resource_size(
    client_get: Callable[[str], IOResultE[httpx.Response]],
    url: str,
) -> IOResultE[int]:
    return client_get(
        url,
    ).bind_result(
        lambda response: safe(tap(httpx.Response.raise_for_status))(response),
    ).map(
        lambda response: len(response.content),
    )

print(fetch_resource_size(
    impure_safe(httpx.get),
    'https://sobolevn.me',
))
# => <IOResult: <Success: 27972>>
```

We have changed a couple of things to make our pseudo-code real:

1. We now use [`IOResultE`](https://returns.readthedocs.io/en/latest/pages/io.html) which is a functional way to handle sync `IO` that might fail. Remember, [exceptions are not always welcome](https://sobolevn.me/2019/02/python-exceptions-considered-an-antipattern)! In a traditional approach, no one cares about exceptions. We do ❤️
2. We changed one `.map` call to `.bind_result` for correct type composition, we have also changed how `raise_for_status` is called. Because in `httpx` it does not return any value and we still want to chain it, `tap` helps us with that. We also mark it as `safe`, now it won't raise any exceptions. Instead, it will return `Result` values: `Success[Response]` if the request was successful, and `Failure(reason)` if it was the case
3. We use [`httpx`](https://github.com/encode/httpx/) that can work with sync and async requests

Now, let's try the async version!

### Async version

```python
from typing import Callable

import anyio
import httpx

from returns.functions import tap
from returns.future import FutureResultE, future_safe
from returns.result import safe

def fetch_resource_size(
    client_get: Callable[[str], FutureResultE[httpx.Response]],
    url: str,
) -> FutureResultE[int]:
    return client_get(
        url,
    ).bind_result(
        safe(tap(httpx.Response.raise_for_status)),
    ).map(
        lambda response: len(response.content),
    )

page_size = fetch_resource_size(
    future_safe(httpx.AsyncClient().get),
    'https://sobolevn.me',
)
print(page_size)
print(anyio.run(page_size.awaitable))
# => <FutureResult: <coroutine object async_map at 0x10b17c320>>
# => <IOResult: <Success: 27972>>
```

Notice, that we have exactly the same result,
but now our code works asynchronously.
And its core part didn't change at all!

However, it has some important notes:

1. We changed sync `IOResultE` into async [`FutureResultE`](https://returns.readthedocs.io/en/latest/pages/future.html) and `impure_safe` to `future_safe`
2. We now also use `AsyncClient` from `httpx`
3. We also require to run the resulting `FutureResult` value. To demonstrate that this approach works with any async library (`asyncio`, `trio`, `curio`), I am using [`anyio`](https://github.com/agronholm/anyio) utility

### Combining the two

And now I can show you how you can combine
these two versions into a single type-safe API.

Sadly, [Higher Kinded Types](https://github.com/dry-python/returns/issues/408) and proper [type-classes](https://github.com/dry-python/classes) are work-in-progress, so we would use regular `@overload` function cases:

```python
from typing import Callable, Union, overload

import anyio
import httpx

from returns.functions import tap
from returns.future import FutureResultE, future_safe
from returns.io import IOResultE, impure_safe
from returns.result import safe

@overload
def fetch_resource_size(
    client_get: Callable[[str], IOResultE[httpx.Response]],
    url: str,
) -> IOResultE[int]:
    """Sync case."""

@overload
def fetch_resource_size(
    client_get: Callable[[str], FutureResultE[httpx.Response]],
    url: str,
) -> FutureResultE[int]:
    """Async case."""

def fetch_resource_size(
    client_get: Union[
        Callable[[str], IOResultE[httpx.Response]],
        Callable[[str], FutureResultE[httpx.Response]],
    ],
    url: str,
) -> Union[IOResultE[int], FutureResultE[int]]:
    return client_get(
        url,
    ).bind_result(
        safe(tap(httpx.Response.raise_for_status)),
    ).map(
        lambda response: len(response.content),
    )
```

# TODO: examplain `@overloads`

```python
# Sync:
print(fetch_resource_size(
    impure_safe(httpx.get),
    'https://sobolevn.me',
))
# => <IOResult: <Success: 27972>>

# Async:
page_size = fetch_resource_size(
    future_safe(httpx.AsyncClient().get),
    'https://sobolevn.me',
)
print(page_size)
print(anyio.run(page_size.awaitable))
# => <FutureResult: <coroutine object async_map at 0x10b17c320>>
# => <IOResult: <Success: 27972>>
```

# TODO: give more details about its usage

`mypy` is pretty happy about it too:

```
» mypy async_and_sync.py
Success: no issues found in 1 source file
```

# TODO: show `mypy` error
# TODO: intermediate conclusion

I hope, that this quick demo shows how awesome `async` programs can be!
Feel free to try new `dry-python/returns@0.14` release, it has lots of other goodies!


## Other awesome features

Speaking about goodies, I want to highlight several of features I am most proud of:

1. [Typed `partial` and `@curry`](https://returns.readthedocs.io/en/latest/pages/curry.html) functions

```python
from returns.curry import curry, partial

def example(a: int, b: str) -> float:
    ...

reveal_type(partial(example, 1))
# note: Revealed type is 'def (b: builtins.str) -> builtins.float'

reveal_type(curry(example))
# note: Revealed type is 'Overload(def (a: builtins.int) -> def (b: builtins.str) -> builtins.float, def (a: builtins.int, b: builtins.str) -> builtins.float)'
```

Which means, that you can use `@curry` like so:

```python
@curry
def example(a: int, b: str) -> float:
    return float(a + len(b))

assert example(1, 'abc') == 4.0
assert example(1)('abc') == 4.0
```

2. [Functional pipelines with type-inference](https://returns.readthedocs.io/en/latest/pages/pipeline.html)

You can now use functional pipelines with full type inference that is augmentated by a custom `mypy` plugin:

```python
from returns.pipeline import flow
assert flow(
    [1, 2, 3],
    lambda collection: max(collection),
    lambda max_number: -max_number,
) == -3
```

We all know how hard it is to work with `lambda`s in typed code because its arguments always have `Any` type. And this breaks regular `mypy` inference.

Now, we always know that `lambda collection: max(collection)` has `Callable[[List[int]], int]` inside this pipeline. And `lambda max_number: -max_number` is just `Callable[[int], int]`.

3. [`RequiresContextFutureResult`](https://returns.readthedocs.io/en/latest/pages/context.html#requirescontextfutureresult-container) for [typed functional dependency injection](https://sobolevn.me/2020/02/typed-functional-dependency-injection)

It is an abstraction over `FutureResult` we have already covered in this article.
It might be used to explicitly pass dependencies in a functional manner in your async programs.


## To be done

However, there are more things to be done before we can hit `1.0`:

1. We need to implement Higher Kinded Types or their emulation, [source](https://github.com/dry-python/returns/issues/408)
2. Adding proper type-classes, so we can implement needed abstractions, [source](https://github.com/dry-python/classes)
3. We would love to try the `mypyc` compiler. It will potentially allow us to compile our typed-annotated Python program to binary. And as a result, simply dropping in `dry-python/returns` into your program will make it several times faster, [source](https://github.com/dry-python/returns/issues/398)
4. Find new ways to write functional Python code, like our on-going investigation of ["do-notation"](https://github.com/dry-python/returns/issues/392)


## Conclusion

Composition and abstraction can solve any problem. In this article, I have shown you how they can solve a problem of function colors and allow people to write simple, readable, and still flexible code that works. And type-checks.

Check out [`dry-python/returns`](https://github.com/dry-python/returns), provide your feedback, learn new ideas, maybe even [help us to sustain this project](https://github.com/sponsors/dry-python/)!

And as always, [follow me on GitHub](https://github.com/sobolevn/) to keep up with new awesome Python libraries I do!
