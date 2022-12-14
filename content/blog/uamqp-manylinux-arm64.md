---
title: "Build uAMQP Python wheel for arm64v8"
date: 2022-12-10T15:27:25Z
draft: false
tags: [python, arm, manylinux, microsoft, azure]
featured: true
image: "https://i.ibb.co/XXQxrhs/sujith-devanagari-ksb4-BNq5nfg-unsplash.jpg"
alt: "Old cart wheel"
summary: "Build Microsoft uAMQP Python package painlessly with Manylinux docker image"
---

TL;DR
-----

See this [gist](https://gist.github.com/luca-drf/30921511c18559fd6f5c4fa7b94e4615)


Problem
-------

Most of Microsoft Azure's Python packages have
[uAMQP](https://pypi.org/project/uamqp/)
([azure-uamqp-python](https://github.com/Azure/azure-uamqp-python)) as a
dependency, hence if you're developing any automation involving Azure with
Python, you'd almost certainly need to install it in your pipeline. Under the
hood, Python `uAMQP` uses its C counterpart
[azure-uamqp-c](https://github.com/Azure/azure-uamqp-c) as an extension,
therefore that's required as an appropriate byte-compiled library for the target
architecture, system and Python version the pipeline is running on.

In most cases, third-party Python packages that requires C extensions are
shipped in archives where the Python code is packaged alongside the compiled
extensions. Such archives are called [wheels](https://pythonwheels.com/) and can
be installed as any other third-party package, using
[Pip](https://pip.pypa.io/en/stable/), given that pre-built wheels targeting
architecture, system and Python version `Pip` is installing on, are provided and
available on the PyPI repository for the package.

Microsoft is publishing `uAMQP` Python wheels targeting Windows, Linux and
MacOS, and for all live major Python 3 versions (3.7+). The only problem is that
the only targeted architecture is x86_64 (see last
[uAMQP release](https://pypi.org/project/uamqp/#files) on PyPI).

To be fair the source code is well engineered and can be fairly easily compiled
for Linux on Arm architectures, in fact `uAMQP` GitHub repo, features
build scripts for Python 3.5 on armV7 (which could probably work on a Raspberry
PI 2
[see here](https://github.com/Azure/azure-uamqp-python/blob/main/build_armv7l.sh))
but the [Manylinux one](https://github.com/Azure/azure-uamqp-python/blob/main/build_many_linux.sh)
is strictly targeting x86_64.

So as arm64v8 based servers are becoming widely adopted by public Cloud
providers, I find quite peculiar the lack of arm64 pre-built wheels on PyPI.
It's definitely not a technical challenge/burden for `uAMQP` maintainers (I
think), and I hope they'll start providing them, and so are other devs on
[azure-uamqp-python's Issues on GitHub](https://github.com/Azure/azure-uamqp-python/issues/349).

Don't get me wrong, I'm really glad the direction Microsoft took in the last
decade, heavily investing on Open Source and Python. Kudos to them!


A solution
----------

So in the meantime, if you need to build `uAMQP` for your Linux system running
on Arm, below is a build script that should do the trick. It requires docker and
an image provided by the amazing [Manylinux project](https://github.com/pypa/manylinux).

{{< gist luca-drf 30921511c18559fd6f5c4fa7b94e4615 "build-uamqp-arm-wheel.sh" >}}


### Cross compile aarch64 on x86_64

Note that you can use the same script to cross compile for aarch64 on a Linux
x86_64 machine, thanks to the very clever
[qemu-user-static](https://github.com/multiarch/qemu-user-static) docker image
by simply running the following container before the above script:

```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

I'm not going to go into details on how the above works as it's a bit out of the
scope of this post, but I strongly recommend to everyone
[reading about it](https://github.com/multiarch/qemu-user-static/blob/master/docs/developers_guide.md)
as what they did is a beautiful hack and, in my opinion, shows why running
containers in _privileged_ mode can be both powerful and extremely dangerous at
the same time.
