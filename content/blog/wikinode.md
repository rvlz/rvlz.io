---
title: "Wikinode Python Package"
date: 2020-06-25T20:21:24-05:00
draft: false
tags:
  - "python"
  - "wikipedia"
---

Hey, I recently released a mini python library called wikinode. Its sole goal is to
facilitate communication with Wikipedia's REST API. The client focuses on getting article
summaries, but in the future it will have extended search functionality.

Before taking a closer look, let's install the package. There are two ways to do this.
First you can install the package from PyPI:

```shell
$ pip install wikinode
```

Or you can clone the git repository and install it directly:

```shell
$ git clone https://github.com/rvlz/wikinode.git
$ cd wikinode
$ python setup.py install
```

## Quick start

In tech we always hear a lot buzzwords thrown around. One I always hear is *The Cloud*.
Though I understand the general idea, I'd like a more precise definition. So let's open 
up *Terminal* (or the terminal emulator of your choosing) and start the Python interpreter.

```shell
$ python
```

Next import *wikinode*. Also, to get nice JSON output import *pprint*.

```python
>>> import wikinode
>>> from pprint import pprint
```

To get a single summary use `wikinode.fetch`:

```python
>>> summary = wikinode.fetch("cloud computing")
>>> pprint(summary, sort_dicts=False)
{'title': 'Cloud computing',
 'description': 'Form of Internet-based computing that provides shared '
                'computer processing resources and data to computers and other '
                'devices on demand',
 'extract': 'Cloud computing is the on-demand availability of computer system '
            'resources, especially data storage and computing power, without '
            'direct active management by the user. The term is generally used '
            'to describe data centers available to many users over the '
            'Internet. Large clouds, predominant today, often have functions '
            'distributed over multiple locations from central servers. If the '
            'connection to the user is relatively close, it may be designated '
            'an edge server.',
 'query': 'Cloud computing'}
```

I wanted to read more, so I looked up the corresponding article online. There were a few
words I stumbled across that weren't so clear to me: *economies of scale*, *hardware
virtualization*, and *service-oriented architecture*. But I don't want long-winded
explanations.

To get only short descriptions of multiple summaries use `wikinode.fetch_many` and set the
`short` parameter to True.

```python
>>> topics = [
... "economies of scale",
... "hardware virtualization",
... "service-oriented architecture",
... ]
>>> summaries = wikinode.fetch_many(topics, short=True)
>>> pprint(summaries, sort_dicts=True)
[{'title': 'Economies of scale',
  'description': 'Cost advantages obtained via scale of operation',
  'query': 'economies of scale'},
 {'title': 'Hardware virtualization',
  'description': 'The virtualization of computers or operating systems',
  'query': 'hardware virtualization'},
 {'title': 'Service-oriented architecture',
  'description': 'architectural pattern in software design',
  'query': 'service-oriented architecture'}]
```

And there you have it! If you want to learn more, check out the 
[documentation](https://wikinode.readthedocs.io/en/latest/) or the
[source code](https://github.com/rvlz/wikinode).

Have questions or suggestions? Send me an email at ricardo@rvlz.io.

Thanks!
