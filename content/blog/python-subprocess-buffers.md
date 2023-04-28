---
title: "Capture Python subprocess output in real-time"
date: 2022-11-20T17:49:41Z
draft: false
tags: [python, subprocess, unix]
featured: true
image: "https://i.ibb.co/QDGLbd2/jake-walker-MPKQi-Dp-Myq-U-unsplash-resized.jpg"
alt: "Generic linux console output"
summary: "The Interwebs are full of recipes on how to capture and stream the output of a Python subprocess in real-time. Most of them don't work, so here's how to do it (with Python 3)"
description: "Blog post on how to capture Python subprocess output in real-time."
---

TL;DR
-----

Here's how to _read_ and _print_ a subprocess _stdout_ in "real-time", or in
other words, capture the subprocess' _stdout_ as soon as bytes are written to it.

```python
# parent_process.py
from subprocess import Popen, PIPE

with Popen(["python", "child_process.py"], stdout=PIPE) as p:
    while True:
        # Use read1() instead of read() or Popen.communicate() as both blocks until EOF
        # https://docs.python.org/3/library/io.html#io.BufferedIOBase.read1
        text = p.stdout.read1().decode("utf-8")
        print(text, end='', flush=True)


# child_process.py
from time import sleep

while True:
    # Make sure stdout writes are flushed to the stream
    print("Spam!", end=' ', flush=True)
    # Sleep to simulate some other work
    sleep(1)
```

If you'd like to learn more about Python's I/O, buffers configuration, and a
real life problem that inspired this blog post, keep reading :)


Problem
-------

On a Python group chat I've read an interesting question, I'm reporting an
edited version below:
> I have a script that opens a program with Popen. stdout is redirected to a
> PIPE. The script reads few lines on stdout to discover how to connect to the
> program using a socket. Unfortunately, at some point later the stdout pipe
> gets full as it isn't read, and it blocks the subprocess.

That behaviour is expected, in fact, it's mentioned in Python's subprocess docs
for [`Popen.wait()`](https://docs.python.org/3/library/subprocess.html#subprocess.Popen.wait)

> This will deadlock when using stdout=PIPE or stderr=PIPE and the child process
> generates enough output to a pipe such that it blocks waiting for the OS pipe
> buffer to accept more data.

(Note: I have omitted the last sentence about `Popen.communicate()` as it's not
relevant for our case, I'll go back to it in more detail later.)

So, how can we read the first few lines written by a subprocess on its _stdout_,
save them and throw away the rest while the subprocess is running and writing
without stopping it?

**TL;DR** (part 2), take me to the [solution](#another-but-better-solution)

A solution
----------

### Using a text file instead of a pipe

We could redirect our subprocess _stdout_ to a file instead of a pipe, read the
first few lines and forget about the rest until the subprocess terminate and
then delete the file.

That would possibly look something like this:

```python
from subprocess import Popen
from time import sleep

max_lines_to_read = 10
lines_read = 0

with open("my_command.out", "w") as subprocess_out:
    with Popen(["my_command"], stdout=subprocess_out) as process:
        with open("my_command.out", "r") as subprocess_in:
            while True:
                text = subprocess_in.read()
                if not text:
                    sleep(1)
                    continue
                if lines_read < max_lines_to_read:
                    if text.endswith("\n"):
                        # TODO: Store or use the whole line
                        lines_read += 1
                else:
                    break

    # TODO: Write the rest of the logic here and terminate `process` if needed
```

**Note**: I didn't use `readline()` or `readlines()` because they would behave
just like `read()` if our subprocess doesn't terminate each _write_ with a _new
line_ (i.e. use one _write_ per line) so it's less confusing to simply use
`read()` and look for `\n` ourselves.

This solution works, but if we're only interested in few lines there's really
no point in having that file on disk, what if it ends up being several gigabytes
and the subprocess having to run for days? You don't want to be that person who
forced IT to impose stricter quotas on your VMs mounts, do you? 😉

No, we're dealing with a stream of data, and we should be coding accordingly.

So, how can we stream the output of a subprocess as it gets generated, rather
than waiting for it to terminate and print it all?


### What does the official Python documentation suggests?

If we read Python's official documentation (as all good Pythonistas always do)
for [subprocess module](https://docs.python.org/3/library/subprocess.html),
we're strongly encouraged to use `Popen.communicate()` for writing/reading
piped subprocesses STDIN/STDOUT. That doesn't quite work the way we expect
though, in fact `communicate()` seems to be blocking and even calling it with a
timeout `communicate(timeout=2)` doesn't seem to work as bytes aren't returned
while the pipe is open and being written. Bummer.

Unfortunately Python's official documentation doesn't offer any alternative
solution, "There should be one -- and preferably only one -- obvious way to do
it." the Zen of Python says, although using `Popen.communicate()` to read the
_stdout_ of a piped subprocess is all but obvious. Sorry Zen of Python and
official docs, but we have to find another way.


Python buffers and I/O
----------------------

While trying to figure out why `Popen.communicate()` didn't work as expected
I've refreshed my knowledge on POSIX pipes and buffering strategies in _libc_.
There are essentially three kinds of streams:

- Unbuffered (characters are transmitted individually, as soon as possible)
- Line buffered (characters are transmitted in blocks, when _new line_ is encountered)
- Fully buffered (characters are transmitted in blocks of arbitrary size)

See [GNU Buffering Concepts](https://www.gnu.org/software/libc/manual/html_node/Buffering-Concepts.html).

Typically, POSIX pipes are _fully buffered_ streams, while streams attached to a
TTY are usually _line buffered_. It's important to remember that, especially
when redirecting _stdout_ to a pipe or a file (instead of a terminal).

Python follows the same strategies when implementing its buffers, and it's also
worth remembering that an extra layer of internal buffering might occur on both
reads and writes.

Lastly, it's important to remember that _stdout_ streams in Python can be
handled by different `io` classes, depending on the type of stream/buffering
strategy.


### Line buffered streams (TTY)

Consider the following program, I've called it `spam_one_line.py`:

```python
# spam_one_line.py

import sys
from time import sleep

for _ in range(8):
    print("Spam!", end=' ')  # Printed string ends with a space instead of the default
    sleep(1)
print("Lovely Spam! Wonderful Spam!")
print("Line written", file=sys.stderr)
```

What do you think the output of this program will be on your terminal? Or more
importantly, **when** do you think those characters will appear?

**Spoiler alert**: two lines will appear at the same time:

```
Spam! Spam! Spam! Spam! Spam! Spam! Spam! Spam! Lovely Spam! Wonderful Spam!
Line written
```

That's because _stdout_ and _stderr_ are both attached to a TTY and that by
default means `sys.stdout` and `sys.stderr` are instances of
[`io.TextIOWrapper`](https://docs.python.org/3/library/io.html#io.TextIOWrapper)
(which is the same type of instance that is returned by `open()` when opening a
text file) but with `line_buffering=True`. Hence, characters are _flushed_ onto
the underlying binary buffer when _new line_ is encountered.

It's easy to check whether a stream is attached to a TTY as `io.IOBase` class
implements `isatty()` method that can be invoked on all its subclasses; in this
case `sys.stdout.isatty() == True`.

So what if we want to "print immediately" on _stdout_? Well, one way to do it
is to call `print()` with `flush=True`:

```python
print("Spam!", end=' ', flush=True)
```

From Python 3.7 onwards, another way is to _reconfigure_ `sys.stdout` to disable
the interpreter's buffer and transmit all the subsequent writes to the system
buffer:

```python
sys.stdout.reconfigure(write_through=True)
```

### Fully buffered streams (pipe)

What buffering strategy and what type of stream is Python implementing when a
Python process is invoked using `subprocess.Popen` and its _stdout_ is
redirected to a pipe instead of being attached to a TTY?

Let's first refresh what a pipe is and how it works:

In very simple terms, a pipe is a mechanism for multiprocess communication
provided by the OS. It has two separate ends, a _writing_ and a _reading_ one.
The data is handled in a first-in, first-out (FIFO) order.

So when we call `subprocess.Popen` and redirect the subprocess' _stdout_ to a
pipe, `Popen` first creates the pipe, which means creating the two ends as two
separate binary file descriptors pointing to the same file (one in reading mode
and one in writing mode); then _forks_ the calling process (creating a child
process which will share both file descriptors), redirect the child process
_stdout_ to the file descriptor pointing at the writing end of the pipe, and
finally _exec_ the program that should run as the child process.

For more info see _libc's_
[Pipes and FIFOs](https://www.gnu.org/software/libc/manual/html_node/Pipes-and-FIFOs.html),
[Creating a pipe](https://www.gnu.org/software/libc/manual/html_node/Creating-a-Pipe.html) and
[Pipe atomicity](https://www.gnu.org/software/libc/manual/html_node/Pipe-Atomicity.html)
documentation.

So for example if we instantiate a `process` object as:

```python
with subprocess.Popen(["my_command"], stdout=subprocess.PIPE) as process:
    ...
```
Then `process.stdout` will hold the _reading_ end of the pipe while (assuming
"my_command" is another Python program) the child process' `sys.stdout` will
hold the _writing_ end.

It's important to notice that the _reading_ end is handled by an instance of
[`io.BufferedReader`](https://docs.python.org/3/library/io.html#io.BufferedReader)
as it's open in binary reading mode while the _writing_ end will still be
handled by an instance of `io.TextIOWrapper` (again, assuming the child process
runs a Python program) but in this case both `sys.stdout.isatty()` and
`sys.stdout.line_buffering` will evaluate to `False`.


### Configuring a subprocess piped STDOUT

Okay, so, using a `io.BufferedReader` instance in our use case isn't great,
because we basically want to read lines from _stdout_ as if it was attached to a
TTY. So, is there a way to reconfigure our piped subprocess _stdout_ buffering
strategy? Luckily, this time, the answer can be found by reading
[Popen docs](https://docs.python.org/3/library/subprocess.html#subprocess.Popen)
and its many, many options; in fact, setting `bufsize=1` and
`universal_newlines=True` when invoking `Popen`, will change the _reading_ end
of our pipe's type to `io.TextIOWrapper` and the underlying buffer will be line
buffered. Note that the wrapper's buffer on top of the binary one, instead, will have
`line_buffering=False` (which is a bit confusing but coherent).

So, having `io.TextIOWrapper` instead of `io.BufferedReader` as our _reading_
end, make `Popen.communicate()` non-blocking and behave as we expect? Sadly, no.
But, we can read directly from the `stdout` stream of our subprocess, remember?
And since our _reading_ end (`process.stdout`) is an instance of
`io.TextIOWrapper` and the buffering strategy is _line buffered_ we can call
`readline()` on it and expect it to block until a full line is available on the
buffer and return it.

So now we should have all what we need to solve our initial problem in a better
way.


Another (but better) solution
-----------------------------

Instead of dumping our subprocess output to a file, reading the first few lines
and forgetting about the following ones; we could consume the subprocess' output
in a separate thread, send the first few lines to the parent process using a
queue and then continue to consume the rest of the output in the thread (and
discarding it). This way we'd use only the memory we need.


```python
from subprocess import Popen, PIPE
from threading import Thread
from queue import SimpleQueue

def consume_output(p, q, max_lines):
    line_count = 0
    while p.poll() is None:
        line = p.stdout.readline()
        if line_count < max_lines:
            q.put(line)
            line_count += 1

def main(max_lines):
    program_output = []
    line_count = 0
    with Popen(["my_command"], stdout=PIPE, bufsize=1, universal_newlines=True) as p:
        q = SimpleQueue()
        t = Thread(target=consume_output, args=(p, q, max_lines))
        t.start()
        while True:
            line = q.get()
            program_output.append(line)
            line_count += 1
            if line_count == max_lines:
                break

        # TODO: Write the rest of the logic here and do what you need with `program_output`

        p.terminate()
        t.join()  # Blocks until t terminates

if __name__ == "__main__":
    main(max_lines=2)
```

Note: we're using
[`Popen.poll()`](https://docs.python.org/3/library/subprocess.html#subprocess.Popen.poll)
to check whether the child process is running or has terminated, in case it has
been terminated, the thread shall also terminate.

Also note that if "my_command" is a Python program as well, you'll have to remember
to flush prints that you want to transmit immediately, because since `sys.stdout`
has been redirected to a pipe then `sys.stdout.line_buffering == False` and the
buffer will be flushed only when the underlying binary buffer is full.

```python
# spam_many_lines.py

import sys
from time import sleep

while True:
    for _ in range(8):
        print("Spam!", end=' ', flush=True)
        sleep(1)
    print("Lovely Spam! Wonderful Spam!", flush=True)
    print("Line written", file=sys.stderr)
```

Problem solved.

That's great, but the title of this blog post mentions capturing output in "real
time"; so what if the child process doesn't atomically writes full lines? For
example, what if we want to capture the output of one of those command line
programs that print their progress on one line (e.g. with a progress-bar)? In
that case reading line-by-line wouldn't be of much use.


Real-Time output
----------------

In an earlier [section](#fully-buffered-streams-pipe), we talked about our
subprocess piped _stdout_ being handled by a `io.BufferedReader` instance; that
is the default mode `subprocess.Popen` instantiates our `process.stdout`.

By default, `io.BufferedReader`, handles a fully buffered, binary stream, and
implements a `read()` method, although that blocks until EOF if
called with a negative o no parameter, otherwise if called with a positive
integer `n` will block until `n` bytes are read.

I must say, `io.BufferedReader.read()` behaviour is not very clear from the
official Python documentation in my opinion especially because other read
methods like `io.TextIOWrapper.read()` or `os.read()` return "up to" `n` bytes
when called with a positive integer, which means they won't block.

So, is there a way to read bytes from a binary buffer as soon as they're
written, without having to wait for EOF (hence before the writing process closes
the pipe), and without having to read byte by byte with `read(1)` (which is not
very efficient)?

Luckily, that can be achieved by using a different _read_ method:
`io.BufferedReader.read1()` (even though, again, the official Python
documentation is not super clear).

Another option would be calling `os.read()` passing the subprocess' _stdout_ file
descriptor and a positive `n` in order to read at most `n` bytes per call.


Let's write a simple program that just reads the subprocess _stdout_ and prints
it in real-time:

```python
import sys
from subprocess import Popen, PIPE

with Popen(["my_command"], stdout=PIPE) as p:
    while True:
        text = p.stdout.read1().decode("utf-8")
        print(text, end='', flush=True)
        # TODO: Write the rest of the logic here and terminate `process` if needed
```

That's it!

Note that, alternatively, you could have also used `os.read()` in case you
wanted to read up to a fixed number of bytes, so you could have rewritten the
read line as:

```python
text = os.read(p.stdout.fileno(), 1024).decode("utf-8")
```
(Obviously you'd have to `import os` for that)


More on Python buffers
----------------------

The default buffer size of all concrete `io.Buffered*` classes is
`io.DEFAULT_BUFFER_SIZE` which is platform dependant.

In case you wanted to change buffer strategy for _stdout_ stream, you can
re-initialise it. For example, in the above `spam_many_lines.py` you might want
to set custom buffer settings in case _stdout_ is not attached to a TTY, but
keep default settings otherwise (and/or not want to pass `flush=True` to all
`print()` functions).

So you could first check if `sys.stdout` is attached to a TTY, then disable
`io.TextIOWrapper` buffer and instead set a custom buffer size for the underlying
binary buffer.

```python
if not sys.stdout.isatty():
    buff_size = 8
    sys.stdout = io.TextIOWrapper(
        open(sys.stdout.fileno(), 'wb', buff_size),
        write_through=True,
        encoding="utf-8"
    )
```

Note, _stdout_ buffer will be automatically flushed after 8 bytes have been
written.

And that's all I've got on Python buffers for now.

If you made it here, give yourself a pat on the back and see you soon!