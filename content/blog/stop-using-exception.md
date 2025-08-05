---
title: "Stop Using Exception"
date: 2025-06-22T14:42:27+01:00
tags: [python]
draft: false
featured: true
image: "https://i.ibb.co/DDdTQtg7/brett-jordan-XWar9-Mb-NGUY-unsplash-65.jpg"
alt: "Scrabble tiles reading 'OWN YOUR ERROR' on a white paper background"
summary: "Please refrain yourself from raising [`Exception`](https://docs.python.org/3/library/exceptions.html#Exception)
in Python. It doesn't matter if ChatGPT or Claude says it's okay to do it, it's
almost always a very bad idea."
description: "Blog post on how catching/raising Python Exception is almost always a bad idea."
---


TL;DR
-----

Please stop raising (and in most cases catching) top level [`Exception`](https://docs.python.org/3/library/exceptions.html#Exception)
in Python.

There is basically no good reason for raising `Exception` in code written for a
library, as for catching it, probably the only acceptable case would be in the
outermost layer of an application in which no exception should cause the
termination of the interpreter (e.g. a Web server) and with the unconditional
assumption that the exception is logged somewhere.

Also, if you're writing examples for a Python course or guide and your
code snippets catch and/or raise `Exception` please, please, please either state
very clearly that in most cases that's an anti-pattern or, you know, use a
different exception type 😉.


Introduction
------------

I know there are a gazillion blog posts, videos and courses on the Interwebs
telling you that catching and/or raising [`Exception`](https://docs.python.org/3/library/exceptions.html#Exception)
in Python is a no-no. And yet, I've recently stumbled upon some commercial (and
fairly successful) Python library doing it, with predictably terrible results. I
won't reveal the name of the library as I don't like to naming and shaming, and
I've already sent some feedback about that their way.

So why do experienced Python programmers' grumble when they see `except
Exception` in a library? And why do their right eyes start twitching when they
see `raise Exception`?


The Case
--------

As I was working with a commercial library I ended up having to handle the following
exception being raised by the vendor HTTP client (**note**: I'm only reporting the
last two formatted lines of the trace stack as the rest is not important):

```
... omitted trace stack ...

File /my_volume/lib/python3.11/site-packages/vendor/module/utils.py:160, in HTTPUtils.send_request(url, method, token, params, json, verify, auth, data, headers)
    158     response.raise_for_status()
    159 except Exception as e:
--> 160     raise Exception(
    161         f"Response content {response.content}, status_code {response.status_code}"
    162     )
    163 return response.json()

Exception: Response content b'{"error_code":"NOT_FOUND","message":"Resource my-resource not found.","details":[{"@type":"type.googleapis.com/google.rpc.RequestInfo","request_id":"8da16872-3641-43cf-ab6b-3a10559bd6a3","serving_data":""}]}', status_code 404
```

The above is a great example of how **NOT** to write library code, and if the
reason is not clear, please continue reading.

So, the above error was thrown when calling the method `get_resource()` of an
instance of said vendor client. The client class basically abstracts a series of
common operations on said vendor platform and that under the hood are performed
via HTTP requests.

In my code I was trying to get a vendor's `Resource` object and had something like:

```python
client = VendorClient()
resource = client.get_resource("my-resource")
```

As the resource named "my-resource" was not found, the client threw an
exception, and that's okay if it wasn't for the *type* of such exception which
was indeed `Exception`.

That's bad design because `Exception` being the base class of most of Python's
exceptions makes it really hard for the user to handle, unless of course the
user is happy to simply catch the exception, probably log it, then either pass
or raise. See below:

```python
client = VendorClient()
try:
    resource = client.get_resource("my-resource")
except Exception as e:
    # At this point we can't statically infer the type of the error.
    # If the type was TimeoutError we might wanted to handle it with a retry...
    # If the type was ValueError we might wanted to add some context to it and re-raise...
    # But we've got Exception instead so statically we can only tell that something
    # went wrong with client.get_resource() and that's it.
```

For example, try to write a function that reliably returns `True` if
"my-resource" exist and `False` if it doesn't exist using `get_resource()`
(note that `VendorClient` doesn't implement a "`resource_exists()`" method).

Basically the only thing you can do is somehow parse the error message. I've
asked the vendor to provide an example for the above case and below is what they
suggested:

```python
def resource_exists(client, resource_name):
    try:
        client.get_resource(resource_name)
        return True
    except Exception as e:
        if "NOT_FOUND" in str(e):
            return False
        else:
            raise e
```

Ugh, can you see the problem now? Having to rely on the error message is an
extremely brittle solution; error messages are not designed to be interpreted
by machines, they're designed for humans! The structure and content of error
messages is likely to change as it's not part of the public API of a library. If
a vendor library changes the *type* of an exception raised by a public function
they're effectively introducing a breaking change, but changing the error
message of a raised exception isn't (and shouldn't be) a change that breaks
user's code. Is perfectly okay to not even unit-test complete error messages.

But with this vendor's library, error messages are basically public API.

The other annoying drawback of calling a function/method that raises `Exception`
is that forces the user to catch `Exception` hence any other operation in
the `try` block that raises an exception will be caught, forcing the user to
have a dedicated `try` block for `get_resource()` and be more verbose.


A better design
---------------

Let's have another look at the code reported in the error trace stack, see below:

```python
    response.raise_for_status()
except Exception as e:
    raise Exception(
        f"Response content {response.content}, status_code {response.status_code}"
    )
```

If you've used [requests](https://requests.readthedocs.io/en/latest/)
library a bit you probably have noticed the
[`raise_for_status()`](https://requests.readthedocs.io/en/latest/api/#requests.Response.raise_for_status)
call in the first line, and you probably know that the exception raised is
[`requests.HTTPError`](https://requests.readthedocs.io/en/latest/_modules/requests/exceptions/#HTTPError)
which is a very good and feature-rich object (e.g. it contains a reference to
the `Response` objects that raised it for introspection).

So why catching it and raising a generic `Exception`? Also, why catching
`Exception` in the second line when we know that `raise_for_status()` only
raises `HTTPError`? Who knows 🤷🏻‍♂️, maybe the code was written by an AI.

I'd fix the above code by removing the `except` block and let
`requests.HTTPError` propagate. The exception can then be caught at the
`VendorClient` level and wrapped in a custom exception, say `NotFoundResource`
alongside extra `VendorClient`-specific context.

For example:

```python
# Vendor custom exceptions

class ClientError(IOError):
    def __init__(self, *args, **kwargs):
        self.request_error = kwargs.pop("request_error", None)
        super().__init__(*args, **kwargs)

class NotFoundResource(ClientError):
    pass


# VendorClient.get_resource()

def get_resource(name: str, ...):
...
    try:
        response = HTTPUtils.send_request(...)
    except requests.HTTPError as e:
        if e.response.status_code == 404:
            raise NotFoundResource(request_error=e)
...
```

Here's how `resource_exists()` implementation would look like:

```python
def resource_exists(client, resource_name):
    try:
        client.get_resource(resource_name)
        return True
    except NotFoundResource:
        return False
```

Much nicer don't you think?


Conclusion
----------

Catching a specialised exception and then re-raising `Exception` (the most
generic) type, should almost always be avoided. Doing so makes it significantly
harder to handle the resulting exception effectively and adds a significant
amount of brittleness.

Before wrapping code in a `try/except` block, consider whether that's the right
place to handle the exceptions raised by the wrapped code. Remember that
sometimes *less is more*, and it might be better to let the exception propagate
to a higher scope.

When handling exception in library code, try to catch explicitly and precisely
the *type* expected, if possible, avoid catching classes that are too generic;
the risk being trapping exception unintentionally and ending up with
[Error Hiding](https://en.wikipedia.org/wiki/Error_hiding).

When raising an exception, first consider raising a Python
[built-in exception](https://docs.python.org/3/library/exceptions.html#exception-hierarchy),
and then consider a custom one only if the built-ins are too generic (although
they are usually enough).
