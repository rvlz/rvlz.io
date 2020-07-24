---
title: "Pytest and Requests Patterns"
date: 2020-06-26T12:16:28-05:00
draft: false
tags:
  - "python"
  - "testing"
  - "pytest"
  - "requests"
---

Though the importance of writing tests cannot be overstated, streamlining
the process without sacrificing quality can help us spend more time writing
production code—the more exciting part for sure! Below are some simple
patterns for testing code that relies on the popular Python library, Requests.

Before continuing make sure you have a good understanding of Pytest fixtures and
the Requests library. For good introductions, see the
[Pytest documentation](https://docs.pytest.org/en/stable/getting-started.html) and
[Requests documentation](https://requests.readthedocs.io/en/master/user/quickstart/).

## Getting started

If you want to follow along, create a virtual environment and install the
required packages.

```shell
$ mkdir pytest-patterns
$ cd pytest-patterns
$ python -m venv venv
$ source venv/bin/activate
```
```shell
(venv) $ pip install requests pytest responses
```

## Source code

First let's take a look at the functions we'll be testing.

```python
# facts.py
import requests

URL = "http://numbersapi.com/%s/math"


def get_math_fact(number):
    """Get one math fact."""
    response = requests.get(URL % number)
    return response.content.decode()


def get_many_math_facts(numbers):
    """Get multiple math facts."""
    facts = []
    for number in numbers:
        response = requests.get(URL % number)
        facts.append(response.content.decode())
    return facts

```

`get_math_fact` receives an integer, builds a url using the integer, sends
a request, and returns the content of the request, a fact about the integer.
Similarly, `get_many_math_facts` performs the same function on each iteration of
the integer list.

## Responses library primer

If you're familiar with the Responses library, feel free to skip this section.

Although Responses is a library for mocking out the Requests library, I like to
think of it as small configurable server whose state can be checked in a test case.
There are two ways you can set it up: decorate a test case with `responses.activate`
or use the context manager, `responses.RequestsMock`, within a test case. Since the
context manager works nicely with Pytest fixtures, we'll use it throughout the post.

Let's set up the context manager inside a test case called `test_foo`.

```python
import responses
import requests


def test_foo():
    with responses.RequestsMock() as rsps:
        # code goes here
```

To register a mock response, describe the types of requests that you'd like Responses to 
intercept:

```python
def test_foo():
    """Test foo function."""
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            "http://foobar.com/",
            json={"message": "hello world"},
            status=200,
        )
```

Every GET request that hits the endpoint won't raise an exception, and `responses` will
proceed to respond with `{"message": "hello"}` in its body. Assertions can then be made
on the returned data and the `rsps` object.

```python
def test_foo():
    """Test foo function."""
    with responses.RequestMock() as rsps:
        rsps.add(
            responses.GET,
            "http://foobar.com/",
            json={"message": "hello world"},
            status=200,
        )
        response = request.get("http://foobar.com/")
        data = responses.json()
        # assertions on body content
        assert data["message"] == "hello world"
        # assertions on responses object
        assert len(rsps.calls) == 1
        assert rsps.status_code == 200
```

There is no limit on the number of responses that can be registered, which is great
when testing sequences of requests, like in `get_many_math_facts`, but we can already
see how this can lead to code that isn't DRY.

## Testing code the tedious way

Let's try testing `get_math_fact`.

```python
# test_facts.py
import responses

from facts import get_math_fact, get_many_math_facts


def test_get_math_fact():
    """Test function returns a single math fact."""
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            "http://numbersapi.com/4/math",
            body="4 is the first positive non-Fibonacci number.",
            status=200,
        )
        fact = get_math_fact(4)
        assert fact == "4 is the first positive non-Fibonacci number."
        assert len(rsps.calls) == 1
```

Now run the tests.

```shell
(venv) $ pytest
platform darwin -- Python 3.8.2, pytest-5.4.3, py-1.9.0, pluggy-0.13.1
rootdir: /Users/rvlz/Developer/code/rvlz/pytest-patterns
collected 1 item                                                               

test_facts.py .                                                          [100%]

============================== 1 passed in 0.20s ===============================

```

Voila!

It works, but it doesn't look too pretty. The test case dedicates half of its
body to registering the response, distracting the reader from the important
bits of the test case: the assert statements. Also keep in mind in real-world projects
the vast majority of test cases won't be happy paths. To account for cases, say, when
the server returns 4xx status codes requires registering more responses.

But things could be worse. Now let's try testing `get_many_math_facts` with input
`[1, 2, 3]`.

```python
# test_facts.py

def test_get_many_facts():
    """Test function can get multiple math facts."""
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            "http://numbersapi.com/1/math",
            body="1 is the multiplicative identity.",
            status=200,
        )
        rsps.add(
            responses.GET,
            "http://numbersapi.com/2/math",
            body="2 is a primorial, as well as its own factorial.",
            status=200,
        )
        rsps.add(
            responses.GET,
            "http://numbersapi.com/3/math",
            body="3 is the number of spatial dimensions we live in.",
            status=200,
        )
        numbers = [1, 2, 3]
        facts = get_many_math_facts(numbers)
        assert facts == [
            "1 is the multiplicative identity.",
            "2 is a primorial, as well as its own factorial.",
            "3 is the number of spatial dimensions we live in.",
        ]
        assert len(rsps.calls) == 3

```

Run the tests.

```shell
(venv) $ pytest
platform darwin -- Python 3.8.2, pytest-5.4.3, py-1.9.0, pluggy-0.13.1
rootdir: /Users/rvlz/Developer/code/rvlz/pytest-patterns
collected 2 items                                                              

test_facts.py ..                                                         [100%]

============================== 2 passed in 0.18s ===============================

```

Again, it all works. But it's quite tedious rewriting all those response registrations.
We need a better way.

## Pytest fixtures to the rescue!

First let's remove the context declarations from the test cases and put them in a
single fixture.

```python
# test_facts.py
import pytest  # new
import responses as _responses  # new (to use 'responses' identifier elsewhere)

from facts import get_math_fact, get_many_math_facts


@pytest.fixture
def responses():
    """Set up responses fixture."""
    with _responses.RequestsMock() as rsps:
        yield rsps


def test_get_math_fact(responses):
    """Test function returns a single math fact."""
    responses.add(
        _responses.GET,
        "http://numbersapi.com/4/math",
        body="4 is the first positive non-Fibonacci number.",
        status=200,
    )
    fact = get_math_fact(4)
    assert fact == "4 is the first positive non-Fibonacci number."
    assert len(responses.calls) == 1


def test_get_many_facts(responses):
    """Test function can get multiple math facts."""
    responses.add(
        _responses.GET,
        "http://numbersapi.com/1/math",
        body="1 is the multiplicative identity.",
        status=200,
    )
    responses.add(
        _responses.GET,
        "http://numbersapi.com/2/math",
        body="2 is a primorial, as well as its own factorial.",
        status=200,
    )
    responses.add(
        _responses.GET,
        "http://numbersapi.com/3/math",
        body="3 is the number of spatial dimensions we live in.",
        status=200,
    )
    numbers = [1, 2, 3]
    facts = get_many_math_facts(numbers)
    assert facts == [
        "1 is the multiplicative identity.",
        "2 is a primorial, as well as its own factorial.",
        "3 is the number of spatial dimensions we live in.",
    ]
    assert len(responses.calls) == 3

```

* Note that the identifier `responses` replaces `rsps` in both test cases

Make sure the tests pass.

```shell
(venv) $ pytest
platform darwin -- Python 3.8.2, pytest-5.4.3, py-1.9.0, pluggy-0.13.1
rootdir: /Users/rvlz/Developer/code/rvlz/pytest-patterns
collected 2 items                                                              

test_facts.py ..                                                         [100%]

============================== 2 passed in 0.21s ===============================

```

Ok, it looks better and more resuable, but a lot of repetitive code still lingers. 
To make our code DRY, let's handle response registration outside the test cases by 
taking advantage of an awesome pytest feature: passing data from a test case
to a fixture.

Implementing this feature requires adding a parameter—which we'll call `data`—to
the fixture `responses`. It'll accept the registration requirements for our test
cases' responses.

```python
@pytest.fixture
def responses(data):
    """Set up responses fixture."""
    with _responses.RequestsMock() as rsps:
        yield rsps
```

With the assumption that `data` is a list of tuples consisting of a number and a fact
about that number, we can finish writing the fixture.

```python
@pytest.fixture
def responses(data):
    """Set up responses fixture."""
    with _responses.RequestsMock() as rsps:
        for number, fact in data:
            rsps.add(
                _responses.GET,
                "http://numbersapi.com/%s/math" % number,
                body=fact,
                status=200,
            )
        yield rsps
```

Finally we need to pass the requirements from the test cases using
`pytest.mark.parametrize`.

```python
single_fact_data = [
    (4, "4 is the first positive non-Fibonacci number.")
]
many_facts_data = [
    (1, "1 is the multiplicative identity."),
    (2, "2 is a primorial, as well as its own factorial."),
    (3, "3 is the number of spatial dimensions we live in."),
]


@pytest.mark.parametrize("data", [single_fact_data])
def test_get_math_fact(responses):
    """Test function returns a single math fact."""
    fact = get_math_fact(4)
    assert fact == "4 is the first positive non-Fibonacci number."
    assert len(responses.calls) == 1


@pytest.mark.parametrize("data", [many_facts_data])
def test_get_many_facts(responses):
    """Test function can get multiple math facts."""
    numbers = [1, 2, 3]
    facts = get_many_math_facts(numbers)
    assert facts == [
        "1 is the multiplicative identity.",
        "2 is a primorial, as well as its own factorial.",
        "3 is the number of spatial dimensions we live in.",
    ]
    assert len(responses.calls) == 3
```

Run the tests.

```shell
(venv) $ pytest
platform darwin -- Python 3.8.2, pytest-5.4.3, py-1.9.0, pluggy-0.13.1
rootdir: /Users/rvlz/Developer/code/rvlz/pytest-patterns
collected 2 items                                                              

test_facts.py ..                                                         [100%]

============================== 2 passed in 0.21s ===============================

```

That's it! Our test cases have neat, concise, and resusable code! For more details
checkout the [repo](https://github.com/rvlz/pytest-patterns) on github.

Have questions or suggestions? Send me an email at ricardo@rvlz.io.

Thanks!
