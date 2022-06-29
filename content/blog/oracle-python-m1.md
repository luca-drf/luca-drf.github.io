---
title: "Oracle Client and Python on Apple Silicon"
date: 2022-06-25T14:50:06+01:00
draft: false
tags: [python, oracle, arm, apple m1, docker]
featured: true
summary: "Setting up your Apple Silicon Mac for linking and running x86 software. Build and run x86 Python natively (and with Docker)"
---

As a Web back-end developer, in general, I've been quite happy working on an
Apple M1 platform. AArch64 is fairly well-supported and Rosetta2 fills most of
the gaps left open by the lack of available ARM64 builds.

An example of missing support for darwin/arm64 are Oracle Instant Client libs.
Luckily there are neat ways around it :)


Problem
-------

Recently I wrote a Python program to extract some data from an "old" Oracle DB
11.2. Oracle provides a nice Python library to talk with Oracle DB which was
recently updated and renamed [oracledb](https://github.com/oracle/python-oracledb).
The only problem is that Oracle DB 11.2 is only supported in _Thick mode_ which
requires [Oracle Instant Client](https://www.oracle.com/database/technologies/instant-client.html)
libraries.

At the time of writing, Oracle doesn't distribute Oracle Instant Client arm64
builds for macOS and doesn't seem to be planning to. So I wondered how difficult
would have been to manage x86 libraries and executables alongside native arm64
ones in order to keep most of my development setup and workflows (i.e. avoiding
having to install VMs for developing with x86 libs and x86 Python builds).

After doing some research on the Interwebs, I found two or three solutions that
worked, but all of them involved some hackish setup. In the following guide I'll
show my approach in which I tried to be as neat and safe as possible.

(I'm assuming XCode and Command Line Tools are already installed on your System)

Here we go:


Rosetta2
--------

A good way to run darwin/x86_64 software on Apple Silicon is using [Rosetta2](https://support.apple.com/en-gb/HT211861)
which is very well integrated in macOS and can be installed by simply:

```bash
/usr/sbin/softwareupdate --install-rosetta --agree-to-license
```

You might get a `Package Authoring Error` message, but that shouldn't prevent the
installation to terminate successfully.


x86 Terminal
------------

The Terminal (in my case [iTerm2](https://iterm2.com/)) is my main entry point for
running software when developing, so with Rosetta2 installed I first need an x86
emulated Terminal. That can be achieved by simply "right-clicking" on iTerm2 app,
"Duplicate" it, rename the copied version "iTerm-x86", then select it and
open the info panel (⌘-I), and finally select "Open Using Rosetta" checkbox. Done. Note
that the same can be done with any other macOS app.

It's probably a good idea to update your command prompt to display some kind of
info about the architecture in order not to get confused when you have two
terminals looking exactly the same but running two different environments.

I've added the following code to my zsh theme and included the variable in my
prompt.

```bash
if [ $(arch) = "i386" ]; then
    ARCH_PROMPT="x86"
else
    ARCH_PROMPT="arm"
fi
```

Note: you can always launch an x86 shell within your current shell:

```bash
arch -x86_64 zsh
```
Also note: the environment will be inherited in the new shell session. Personally I find it
safer to keep two separated terminals.


x86 Homebrew
------------

[Mac Homebrew](https://brew.sh/) comes very handy for building and managing OSS
on macOS. Since Mac Homebrew on arm64 is rooted at `/opt/homebrew` instead of
`/usr/local`, it's possible to install x86 software and keep it neatly separated
from arm64. You just need to launch an x86 shell and simply install Mac Homebrew.
It will automatically choose its root based on the detected architecture (in our
case i386).

Then, in order to make sure your environment is properly set, add the following code to
your `.zprofile` (or `.profile`):

```bash
if [ $(arch) = "i386" ]; then
  eval "$(/usr/local/bin/brew shellenv)"
else
  eval "$(/opt/homebrew/bin/brew shellenv)"
fi
```


x86 Python with Pyenv
---------------------

Now that we have an x86 terminal and a dedicated space for our x86 libraries, we
need to build an x86 CPython interpreter. Luckily Pyenv will sort that out for us.

### Pyenv

In order to build Python, we're going to use [Pyenv](https://github.com/pyenv/pyenv)
(which you should be using already for managing different Python versions and
virtualenvs anyway). Don't worry if you already installed an "arm" Pyenv, and
have already populated Pyenv's root; the two executables won't interfere with
each other as they can share the same root (e.g. `~/.pyenv`).

So using our x86 terminal, let's install Pyenv in the x86 space:

```bash
brew install pyenv pyenv-virtualenv
```

Note: you might need to reload the shell in order for `pyenv-virtualenv` to work.
Also note: brew will take care of openssl and readline dependencies installing
their x86 versions in `/usr/local/opt`.

Let's also install [Pyenv Alias](https://github.com/s1341/pyenv-alias) a handy
Pyenv plugin for labelling Python versions, so we can keep arm and x86 Python builds
separated under `$(pyenv root)/versions`:

```bash
git clone https://github.com/s1341/pyenv-alias.git $(pyenv root)/plugins/pyenv-alias
```

### Python

Now we can build an x86 Python interpreter using our x86 terminal and Pyenv.
Also, let's give it a custom name, so it won't interfere with other arm versions:

```bash
VERSION_ALIAS="3.10.3_x86" pyenv install 3.10.3
```

If everything worked fine, you should be able to run a Python REPL and check the
interpreter platform using the `platform` module:

```python
>>> import platform
>>> platform.machine()
'x86_64'
>>>
```
Note that even when using an x86 terminal, an arm built Python interpreter would
print `'arm64'`.


Oracle Instant Client & Python
------------------------------

Finally, we have all that we need to install Oracle Instant Client, and calling it
from Python.

So, using our x86 terminal, let's install Oracle Instant Client libraries with
`brew`. We'll need to add a new tap first:

```bash
brew tap InstantClientTap/instantclient
```

Then we can install the required libs (in my case, I only needed the basic-lite
version):

```bash
brew install instantclient-basiclite
```

At the time of writing, the above command should install Instant Client Basic Lite
version 19.8.0.0.0.

Let's now create a virtualenv for our project:

```bash
pyenv virtualenv 3.10.3_x86 oracle_client_app
```

And activate it:

```bash
pyenv activate oracle_client_app
```

Let's install the new [python-oracledb](https://github.com/oracle/python-oracledb)
which has replaced `cx_Oracle` (although the latter would work as well):

```bash
pip install oracledb
```

Then test that Instant Client is correctly loaded. To do that, open a python
REPL and then:

```python
>>> import oracledb
>>> oracledb.init_oracle_client()
>>> oracledb.clientversion()
(19, 8, 0, 0, 0)
>>>
```

Nice, now we can start developing our app!


Docker
------

As the app I was developing was going to be deployed as a Docker container, I
also wondered if there was an easy way to build an x86 Docker image on my M1 Mac;
and perhaps, spin up an x86 container as well, so I could "smoke test it" locally.

The answer is, well, in this case yes!


### Python Toy App

For the purpose of this guide the app will simply _init_ Oracle Client and then
print its version on stdout.

```python
# oracle_client_app.py

import oracledb

oracledb.init_oracle_client()
version = oracledb.clientversion()
print(f"Oracle Client version: {version[0]}.{version[1]}.{version[2]}")
```

### Dockerfile

Now we need a Dockerfile that installs all our dependencies:

```dockerfile
FROM python:3.10.3-slim-buster

# Install Oracle Instant Client
WORKDIR /opt/oracle

RUN apt-get update \
    && apt-get install -y libaio1 wget unzip \
    && wget https://download.oracle.com/otn_software/linux/instantclient/1915000/instantclient-basiclite-linux.x64-19.15.0.0.0dbru-2.zip \
    && unzip instantclient-basiclite-linux.x64-19.15.0.0.0dbru-2.zip \
    && rm -f instantclient-basiclite-linux.x64-19.15.0.0.0dbru-2.zip \
    && cd /opt/oracle/instantclient_19_15 \
    && rm -f *jdbc* *occi* *mysql* *README *jar uidrvci genezi adrci \
    && echo /opt/oracle/instantclient_19_15 > /etc/ld.so.conf.d/oracle-instantclient.conf \
    && ldconfig \
    && apt-get purge -y --auto-remove wget unzip

WORKDIR /app

COPY oracle_client_app.py /app

# Install GCC for compiling python-oracledb and then install it
RUN apt-get update \
    && apt-get install -y gcc \
    && apt-get clean \
    && pip install --no-cache-dir oracledb \
    && apt-get purge -y --auto-remove gcc

CMD ["python", "oracle_client_app.py"]
```

Note: at the time of writing, the latest Oracle Client version for Linux64 is 21.6.0


### Build and Run

We can build our x86 image by using Docker's `--platform` parameter:

```bash
docker build --platform linux/amd64 -t oracle_client_app
```

And then run it using the same parameter:

```bash
docker run --platform linux/amd64 oracle_client_app:latest
```

Which, if everything went well, should print:

```bash
Oracle Client version: 19.15.0
```

Et voilà! :)


Conclusion
----------

This neat solution wouldn't have been possible without Mac Homebrew maintainers
separating x86 installations from arm ones, kudos to them.

Massive kudos to Apple for Rosetta2 and XCode working seamlessly in emulation mode.

And last but not least, it's really nice to see Docker Desktop for macOS
emulating x86 out of the box on Apple Silicon.

Happy coding!