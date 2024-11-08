# `return` in `finally` considered harmful

### TL;DR

The semantics of `return`, `break` and `continue` in a `finally` block are
surprising. This document describes an analysis of their use in real world
code, which was conducted in order to assess the cost and benefit of blocking
these features in future versions of Python. The results show that

1. they are not used often.
2. when used, they are usually used incorrectly.
3. code authors are typically receptive and quick to fix the code when
   the error is pointed out to them.

My conclusion is that it is both desireable and feasible to make this
a `SyntaxWarning`, and laster a `SyntaxError`.

## Introduction

The semantic of `return`, `break` and `continue` are surprising for many
developers. The [documentation](https://docs.python.org/3/tutorial/errors.html#defining-clean-up-actions)
mentions that

- If the finally clause executes a break, continue or return statement, exceptions are not re-raised.
- If a finally clause includes a return statement, the returned value will be the one
from the finally clause’s return statement, not the value from the try clause’s return statement.

Both of these behaviours cause confusion, but the first one is particularly dangerous
becuase a swallowed exception is more likely to slip through testing, than an incorrect
return value.

In 2019, [PEP 601](https://peps.python.org/pep-0601/) proposed to change Python to emit a
`SyntaxWarning` for a few releases and then turn it into a `SyntaxError`. The PEP was
rejected in favour of viewing this as a programming style issue, to be handled by linters
and [PEP8](https://peps.python.org/pep-0008/).
Indeed, PEP8 now recommends not to used control flow statements in a `finally` block, and
linters such as [pylint](https://pylint.readthedocs.io/en/stable/),
[ruff](https://docs.astral.sh/ruff/) and
[flake8-bugbear](https://github.com/PyCQA/flake8-bugbear) flag them as a problem.

It is not disputed that `return`, `break` and `continue` in a `finally` clause
should be avoided, the question is whether changing Python to forbid it now is worth
the churn.  What was missing at the time that PEP 601 was rejected, was an understanding
of how this issue manifests in practice. Are people using it? How often are they
using it incorrectly? How much churn would the proposed change create?

The purpose here is to bridge that gap by evaluating real code (the 8000 most popular
packages on PyPI) to answer these question. I believe that the results show that PEP 601
should now be implemented.

## Method

The analysis is based on the 8000 most popular PyPI packages, in terms of number
of downloads in the last 30 dats. They were downloaded on the 17th-18th of
October, using
[a script](https://github.com/faster-cpython/tools/blob/main/scripts/download_packages.py)
written by Guido van Rossum, which in turn relies on Hugo van Kemenade's
[tool](https://hugovk.github.io/top-pypi-packages/)
that creates a list of the most popular packages.

Once downloaded, a [second script](https://github.com/iritkatriel/finally/blob/main/scripts/ast_analysis.py)
was used to construct an AST for each file, and traverse it to identify `break`,
`continue` and `return` statements which are directly inside a `finally` block.
By *directly*, I mean that the script looks for a `return` from a function that
contains the `finally`, rather than a function that is defined entirely in the
finally block. Similarly, it looks for `break` or `continue` in a `finally` which
is nested in a loop, rather than in a loop which is entirely in a `finally` block.
I then found the current source code for each occurence, and categorized it. For
cases where the code seems incorrect, I made and issue in the project's bug
tracker. The responses to these issues are also part of the data collected in
this investigation.

## Results

I decided not to include a list of the instances in an appendix, of concern that
it would make this report look like a shaming exercise.  Instead I will describe
the results in general terms, but will mention that some of the problems I found
appear in very popular libraries, including a cloud security application.
For those incluned, it should not be hard to replicate my analysis, since I
provided links to the scripts I used in the Method section.

The projects examined contained a total of 120,964,221 lines of Python code,
and among them the script found 203 instances of `return`, `break` or `continue`
in a `finally` block. Most of these were `return`s, a handful were `break`s, and
none were `continue`. Of these:

- 46 are correct, and appear in tests that target this pattern as a feature (e.g.,
tests for linters that detect it).
- 8 seem correct - either intentionally swallowing exceptions or appearing where
an active exception cannot occur. Despite being correct, it is not hard to rewrite
them to avoid the bad pattern, and it would make the code clearer: deliberately
swallowing exceptions can be more explicitly done with `except BaseException:`,
and `return` which doesn't swallow exceptions can be moved after the `finally`
block.
- 149 were clearly incorrect, and can lead to unintended swallowing of exceptions.
These are analyzed in the next section.

### The Error Cases

Many of the error cases followed this pattern:

```pycon
    try:
        ...
    except SomeSpecificError:
        ...
    except Exception:
        logger.log(...)
    finally:
        return some_value
```

Code like this is obviously incorrect because it deliberately logs and swallows
`Exception` subclasses, while silently swallowing `BaseExceptions`. The intention
here is either to allow `BaseException`s to propagate on, or (if the author is
unaware of the `BaseException` issue), to log and swallow all exceptions. However,
even if the `except Exception` was changed to `except BaseException`, this code
would still have the problem that the `finally` block swallows all exceptions
raised from within the `except` block, and this is probably not the intention
(if it is, that can be made explicit with another `try`-`except BaseException`).

Another variation on the issue iin real code looks like this:

```pycon
        try:
            ...
        except:
            return NotImplemented
        finally:
            return some_value
```

Here the intention seems to be to return `NotImplemented` when an exception is
raised, but the `return` in the `finally` block would override the one in the
`except` block.

## Author reactions

Of the 149 incorrect instances of `return` or `break` in a `finally` clause, 27
were out of date, in the sense that they do not appear in the main/master branch
of the library, as the code has been deleted or fixed by now. The remaining 122
are in 73 different packages, and I created an issue in each one to alert the
authors to the issues. Within two weeks, 40 of the issues received a reaction
from the code maintainers:

- 15 issues had a PR opened to fix the problem.
- 20 received reactions acknowleding the problem as one worth looking into.
- 3 replied that the code is no longer maintained so this won't be fixed.
- 2 closed the issue as "works as intended".

One issue was linked to a pre-existing open issue about non-responsiveness to Ctrl-C,
conjecturing a connection.

Two of the issue were labelled as "good first issue".

## Discussion

The first thing to note is that `return`/`break`/`continue` in a `finally`
block is not something we see often: 203 instance in over 120 million lines
of code. This is, possibly, thanks to the linters that warn about this.

The second observation is that most of the usages were incorrect: 73% in our
sample (149 of 203).

Finally, the author responses were overwhelmingly positive. Of the 40 responses
received within two weeks, 35 acknowledged the issue, 15 of which also created
a PR to fix it. Only two thought that the code is fine as it is, and three
stated that the code is no longer maintained so they will not look into it.

## Conclusion

The results indicate that `return`, `break` and `continue` in a finally block

- are rarely used.
- when they are used, they are usually used incorrectly.
- code authors are receptive to changing their code, and tend to find it easy to do.

This indicates that it is both desireable and feasible to change Python to emit
a `SyntaxWarning`, and in a few years a `SyntaxError` for these patterns.

## Acknowledgements

I thank:

- Alyssa Coghlan for
[bringing this issue my attention](https://discuss.python.org/t/pep-760-no-more-bare-excepts/67182/97).

- Guido van Rossum and Hugo van Kemenade for the
[script](https://github.com/faster-cpython/tools/blob/main/scripts/download_packages.py)
that downloads the most popular PyPI packages.

- The code authors I contacted about shoddy `finally` blocks in their code for their
responsiveness and grace.
