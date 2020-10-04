---
title: "Customizing the Python Debugger"
date: 2020-10-03T21:04:28-04:00
categories: ["Python", "Technical"]
tags: ["pdb", "custom", "debug", "debugger", "debugging", "python", "memory", "pympler"]
summary: Creating custom commands for the Python Debugger (`pdb`).
---

I've used several IDEs over the years (Visual Studio, Eclipse, MonoDevelop, XCode - even though I kinda regret that last one), but when I started working with Python, I made the decision to use VSCode only as a text editor on steroids, doing everything else inside the terminal. That means I never use VSCode's python debugger, using pdb-ish packages like [pdbpp](https://github.com/pdbpp/pdbpp) instead.

However, I've always wondered exactly they work, as normally those tools retain all of the original pdb functionalities while enhancing the experience with all sorts of cool stuff. I did some digging, and in this post I'll explain how to create a tiny pdb-ish implementation, adding a simple command to print how much memory is being used by the args of the current function.

_All source code created for this post can be found [here](https://github.com/maurcz/posts/tree/main/python-customizing-pdb)._

## Basics

One of the most basic things a young python developer will learn is how to put a breakpoint in the source code. For python < 3.7, the most common way to do this is with `import pdb; pdb.set_trace`. This will import the standard `pdb` module and call function `set_trace()`, which will throw you into python's debugger as soon as you code hits that line. If you decide to use an alternative debugger like `pudb` or `ipdb`, you'll have to adjust that command slightly, since the module/function you'll import will probably be different (e.g. `import ipdb; ipdb.set_trace()`). 

Luckily, versions >= 3.7 can start using the `breakpoint()` built-in function, which has support for non-standard debugging tools as described in [PEP 553](https://www.python.org/dev/peps/pep-0553/). This means it's possible to control `breakpoint()`'s behavior via environment variables, instead of having to use a completely different command in case you decide to switch debuggers. I won't use `breakpoint()` in this post to simplify the explanation (I haven't yet seen much people using this in the wild), but everything in here supports `breakpoint()` as long as the env var is correctly set up.

Going back to `pdb`, turns out `set_trace` is just a simplified interface for the `Pdb` class, as we can see in `pdb`'s [source code](https://github.com/python/cpython/blob/3.8/Lib/pdb.py#L1609). The [docs](https://docs.python.org/3/library/pdb.html) will also state that all `pdb` functionality is defined inside the `Pdb` class, making the code extensible by nature. 

What's interesting about the `Pdb` class is that it implements two other classes `Bdb` and `Cmd` (as if we didn't already have enough acronyms). [Bdb](https://docs.python.org/3/library/bdb.html#module-bdb) is python's debugger framework, which provides basic capabilities like setting breakpoints; [Cmd](https://docs.python.org/3/library/cmd.html#module-cmd) is a framework for writing line-oriented command interpreters. This means you don't necessarily need to inherit from `Pdb` if you want to create your custom debugger - you can do all the heavy lifting yourself using `bdb` and `cmd`. This is not the aim of this post though, we'll just extend `pdb`'s functionalities into our own tiny custom debugger module.


## Overriding the prompt

Let's start with a very simple example: overriding the default `(Pdb)` prompt. We'll need two new python files: `dummy.py`, a dummy script for breakpoints and `debug.py`, which will host our new debugger. Here are the contents of my `dummy.py`, but it can be literally anything you want:

```python
from random import random


def print_the_things(number, text, big_list):
    import pdb; pdb.set_trace()

    print(f"Number: {number}, Text: {text}, List: {big_list}")


print_the_things(
    number=10,
    text="Blame and lies, contradictions arise",
    big_list=[random() for _ in range(100000)]
)
```

We'll hit the breakpoint only once as soon as we enter the function. This is the prompt I'm getting:

![pdb-prompt](/images/002/pdb-prompt.png)

To customize the prompt, we'll need to create a _new_ class that inherits `Pdb` and changes one of the inherited properties during initialization. Here's the code of `debug.py`:

```python
import pdb
import sys


class Debug(pdb.Pdb):
    def __init__(self, *args, **kwargs):
        super(Debug, self).__init__(*args, **kwargs)
        self.prompt = "[extended-pdb] "


def stop():
    debugger = Debug()
    debugger.set_trace(sys._getframe().f_back)
```

What's going on with that code?
- The `__init__` function will call `Pdb`'s initializer through `super()`. We must do this _before_ we throw in our prompt customization, as `Pdb` initializes `self.prompt` with `'(Pdb) '` ([source](https://github.com/python/cpython/blob/3.8/Lib/pdb.py#L142))
- We an then set `self.prompt` to whatever we want
- `stop()` is a simplified interface to our custom debugger, analogous to `pdb.set_trace()`. 

Now we need to import our _custom_ debugger inside `dummy.py`, calling our _custom_ breakpoint function. Here's the modified code:

```python
from random import random


def print_the_things(**kwargs):
    import debug; debug.stop()  # custom breakpoint

    for key, value in kwargs.items():
        print(f"Arg: {key}, Value: {value}")


print_the_things(
    number=10,
    text="Blame and lies, contradictions arise",
    huge_list=[random() for _ in range(100000)]
)
```

Now if we run our dummy script...

![pdb-prompt-modified](/images/002/pdb-prompt-modified.png)

... we get a custom prompt! What's cool about this structure is that this custom debugger will retain _all_ of pdb's capabilities. You'll still have `n`, `s`, `r` and everything else.

## Building a custom command

Now that we got the basic structure going, we can push a bit more by creating a new command to our custom debugger. 

Earlier I mentioned that `Pdb` is built on top of other two classes, one of them being `Cmd`. This command class dictates how commands should be implemented to be recognized when an user enters the command on a debugging session. The structure is pretty simple: if your debugger class inherits `Cmd` and has a `do_something` method, `something` becomes a new command you can use in the debugger. Since we're extending `Pdb`, which in turn implements `Cmd`, we'll also have to abide by those rules. 

For reference, [this link](https://github.com/python/cpython/blob/3.8/Lib/pdb.py#L1253) shows how `Pdb` implements the long list command. Notice how it has a "shortcut" for long list called `ll`, created by referencing the original method: `do_ll = do_longlist`.

The name of the custom command will be `args_memory_usage`, and the intent is to print inside the debugging session how much memory the args of the current function are consuming. The `sys` module has a function called `getsizeof` that could have been used for that, but while doing some research, I found that it [won't really](https://stackoverflow.com/questions/449560/how-do-i-determine-the-size-of-an-object-in-python) be that precise in all cases. Fortunately, there's a really small dependency we can use for more accurate consumption stats called [Pympler](https://pypi.org/project/Pympler/), and we'll use pympler's `asizeof` for the task.

I also wanted to make sure we're not just throwing bytes numbers at the user by default, as those can be a bit hard to understand for heavy objects. I wrote a pretty basic bytes/kb/mb convertor (I'm using 1000 as base and I'll save yourself from all the discussions about 1000 vs 1024, SI, kibibytes, mebibytes and more), which will also be used to show how to pass args to custom commands. 

Here's our revamped `debug.py`:

```python {linenos=true}
import inspect
import pdb
import sys

from pympler import asizeof


class Debug(pdb.Pdb):
    def __init__(self, *args, **kwargs):
        super(Debug, self).__init__(*args, **kwargs)
        self.prompt = "[extended-pdb] "

    def format_bytes(self, value, format):
        unit_case_sensitive = "kb" if format == "" else format
        unit = unit_case_sensitive.upper()

        if unit == "KB":
            converted = value / 1000
        elif unit == "MB":
            converted = value / 1000000
        else:
            converted = value
            unit = "bytes"

        return f"{converted:.2f} {unit}"

    def do_args_memory_usage(self, arg):
        co = self.curframe.f_code
        dict = self.curframe_locals
        n = co.co_argcount + co.co_kwonlyargcount
        if co.co_flags & inspect.CO_VARARGS: n = n+1
        if co.co_flags & inspect.CO_VARKEYWORDS: n = n+1

        self.message("---- Args memory usage ----")
        for i in range(n):
            name = co.co_varnames[i]
            if name in dict:
                arg_size = asizeof.asizeof(dict[name])
                self.message('%s = %s' % (name, self.format_bytes(arg_size, arg)))
            else:
                self.message('%s = *** undefined ***' % (name,))

    do_amu = do_args_memory_usage


def stop():
    debugger = Debug()
    debugger.set_trace(sys._getframe().f_back)
```

Where did all that code inside `do_args_memory_usage` come from? 

Well, `pdb` already has a pretty good [implementation](https://github.com/python/cpython/blob/3.8/Lib/pdb.py#L1134) for a command that will print all the args of the current function (`a`). It gets the function according to the current state of the debugger, finds the function's args and print them one by one. The `arg` parameter from `do_args_memory_usage` is the string that's entered _after_ the command inside the debugging session (e.g. in `amu mb`, `arg` would be `mb`).

Aside from the name of the function (and the shorthand `do_amu` in line **43**), the only changes I had to make to the original long list implementation are lines **38** and **39**, where I get the size in bytes of the variable and format according to the user input.

The formatter function is extremely simple and naive to a certain degree (it doesn't handle all common units and forces `bytes` for unrecognized args), but providing a perfect implementation for that is beyond the scope of this post. The command now should be good enough to show basic consumption for function args.

Let's check how that looks in the terminal:

![amu-cmd](/images/002/amu.png)

This next pic shows the long version of the command and the fallback to `bytes`:

![amu-cmd-with-long-command](/images/002/amu-2.png)

Aaand now we have a brand new command built on top of `Pdb`! You can follow this same pattern to extend any commands you want from regular `Pdb` - the source code of the `pdb` module is not that hard to understand, and following the pattern above you should be able to customize any standard pdb functionalities you want.

## One caveat

Something to be aware of when playing with these customizations is that _any_ custom debugger will have to be manually installed for any new virtual/conda environment you create. That is also true for any of its dependencies. This means that you'll have to wrap your custom debugger in a package with `setup.py` and the likes, to make a custom debugger functional in the wild. 

Currently I'm a heavy user of `pdbpp` and `conda` environments, so I have an alias in my `.bashrc` to install the debugger for every new conda environment I create. If you don't use a dependency manager like Poetry, keep in mind that your debugger and it's dependencies will show up in you `pip freeze`, which might not be optimal in some cases. 

As long as you can tame the custom debugger dependencies, this whole process can easily be managed.


## Next steps

The two main reasons I became a user of `pdbpp` are the built-in support to themes through [Pygments](https://pygments.org/) and the sticky mode (which helps me with not getting _totally lost_ while debugging). Before writing this post I had no idea how easy it was to create basic extensions to pdb, so I'm tempted so spend some time trying to make my own implementations of those features, for learning purposes. But not today - the avalanche of acronyms in this post kinda made my head hurt a bit. Hopefully that wasn't the case for you.

I hope you enjoyed the post, see you in the next one!


-------
#### Tech summary
- Language: Python
- Main module: `pdb`
- Dependencies: [Pympler](https://pypi.org/project/Pympler/)
