# py-s-lang
### Python-interfacing, simple, somewhat shell-like language

Give your Python scripts a shell-like way to be used:

    some_function arg1 arg2 \
        --keyword-argument=value \
        --other-keyword-arg "other value"
    other_function $0 and other stuff  # $0 is the last result. Any Python object.

If you only use a single line, you can get a (very very) lightweight
(as in missing lots of features) alternative to
[click](https://palletsprojects.com/p/click/) or [invoke](https://www.pyinvoke.org/).
When running from files, you can also use py-s-lang as a simple sandbox.
Automatically make your scripts py-s-lang runnables with the built-in ``insert_runner``
command:

    ./run.py insert_runner path/to/your/file.py
    path/to/your/file.py function_in_there arguments --with keywords

## Contents

- [How to use](#how-to-use)
- [How it works](#how-it-works)
  - [Details: Python side](#details-python-side)
  - [Details: py-s-lang side](#details-py-s-lang-side)
- [The future](#the-future)

## How to use

First, write your Python script. Add some functions/callables you want to expose to
py-s-lang annotating your arguments. As of now, types are only used as converters
(like ``type`` in ``argparse.ArgumentParser.add_argument``), so you'll want to avoid
``bool``, ``dict``, ``list`` and other stuff that doesn't quite work with strings.
I might add special support for it sometime. I reccommend you put the exposed functions into ``__all__``, as otherwise all functions will be exposed.

Note that functions need to be introspectable, so using functions written in C
(e.g. the builtin ``print`` in CPython) won't work.

Test everything by writing a test script using your exposed functions and
running ``python3 py_s_lang.py -f path/to/file.py path/to/test.pysl``. If it works, well done.

If you want to add invoker/runner functionality to your script, run ``./run.py insert_runner path/to/script.py``.
You should see some new code at the end of the file. Test by
running ``path/to/script.py func_name arguments``.

Also see the examples.

### Details: Python side

- ``prepare_functions`` takes a (namespace) dictionary and wraps all callables with
    names in ``__all__`` (if present, otherwise all callables) with ``FunctionWrapper``
- ``FunctionWrapper`` wraps a function and provides the methods:
    - ``parse`` takes a sequence of string arguments and transforms them into a (args, kwargs)
        pair. Function parameters whose annotation ``is bool`` are handled as
        switches: ``--some-option`` is True and ``--no-some-option`` is False.
        For positional boolean parameters, there is no extra support yet.
        As of now, there is no special support for other types, but this might change to
        having different behaviour for more types (e.g. collecting arguments for a list)
    - ``apply`` takes a (args, kwargs) pair and applies it to the wrapped function.
        Before application, types are converted to those specified as type hints
- ``interpret`` takes a dictionary fo functions and a sequence of commands, a command being
    ``['func_name', 'some', 'arguments']``. It executes them sequentially, keeping track
    of the last results (used for ``$0``, ``$1``, ... substitution).
- ``prepare_code`` transforms a big block of text into a list of commands, handling
    backslash escapes, quoting, commants and line continuation

### Details: py-s-lang side

- commands are one each per line
- lines may be continued with a single ``\`` charcter at the end
- end-of-line comments with ``#`` are allowed. Note lines are continued before comments are removed.
- use quoting for arguments with spaces
- use backslash escapes like ``\n``, ``\t``, ``\xff``, ...
- pass keyword arguments like shell options, i.e. ``--name=value`` or ``--name value``
- everything after a lone ``--`` is a positional argument
- you can pass keyword boolean arguments as flags: ``--option`` for True
  and ``--no-option`` for False. For positional boolean parameters,
  you will have to use an empty string ``''`` for False and any text
  for True. Since support may be expanded, I suggest using something like
  "yes", "true" or "on" for True values.
- you can set variables with ``name= value``, where ``value`` will be be handled
  according to substitution rules or with ``name= command at least --one arg``
  where the value returned by ``command`` with arguments will be assigned to ``name``.
  In the unlikely case you need to have a named variable with the output of a
  command that takes no parameters, you can just run the command once and ``name= $0``
  (see next bullet on why that works).
  ``name`` should be a valid identifier, although currently nearly every string
  works, as some characters may get special meanings in the future.
- you may also use ``$N`` (with any number of digits) substitutions,
  where ``N`` is ``commands up - 1``, i.e. 0 is the result of the last command, 1 the one before etc.
- substitutions are always evaluated. Use ``$$`` for a literal ``$``. If your substitution has
    directly following alphanumeric/underscore characters, use braces: ``${123}456`` will substitute ``$123``
- If a substitution appears alone in an argument, the raw Python object is passed to the function.
    Otherwise, it if first made a string. Note that annotation types are handled later.
    In ``func $0 arg$1 'arg with $2 spaces' --keyword=$3 --other ${4}keyword``,
    ``$0`` and ``$3`` are substituted as Python objects.

## How it works

Read the Python side datails part of "How to use" or
see the [detailed implementation information](https://github.com/mik2k2/py-s-lang/blob/master/py_s_lang.py).

## The future
a.k.a. What I would like to add some time

- Expand ``bool`` handling to positional arguments (via a ``bool`` subclass?)
- Add list and dict constructors (include with variable assignment maybe?)
- Add set of common commands / some Python builtins
- Add options to ``run.py insert_runner`` to use something other than ``globals()``
- Add ``if``/``while`` commands
- Add command substitution
- Don't expose functions starting with ``_`` (low priority bc. you can just use ``__all__``)
- Add support for [PEP 593](https://www.python.org/dev/peps/pep-0593/) ``typing.Annotated`` (as soon as I decide to get Python 3.9)
