# Announcing Shake 0.17

I'm delighted to announce Shake 0.17. As always, the full changelog is [on GitHub](https://github.com/ndmitchell/shake/blob/master/CHANGES.txt), but I'd like to highlight three areas that have seen most attention.

## Error Messages

Error messages now support GHC's `HasCallStack` feature, giving code locations in error messages. As an example, let's define rules for both `*.txt` and `overlap.*`, then try and build `overlap.txt`. With Shake 0.17 we get the far more informative error:

    Error when running Shake build system:
    at Example.hs:50:46-55:
    * Depends on: overlap.txt
    * Raised the exception:
    Build system error - key matches multiple rules:
    Key type:       FileQ
    Key value:      overlap.txt
    Rules matched:  2
    Rule 1:         "overlap.*" %> at Example::21:94-106:
    Rule 2:         ".txt" %> at Example::24:94-106:
    Modify your rules so only one can produce the above key

We can see where the dependency was introduced (line 50), where the rules were defined (lines 21 and 24), and what their patterns were.

## The Database module

The new module `Development.Shake.Database` provides operations for working with open Shake databases - meaning you can now open the database, run some actions against, and shut it. Unlike before, you can now run against an open database repeatedly, and query the resulting database for live or erroneous files. When combined with the new feature that `/dev/null` for `shakeFiles` results in no on-disk representation of Shake, you can create an in-memory database, execute it many times, then throw it away. These features aren't targetted at build systems, but allow reuse of Shake in other domains.

If you are using the Database module, and have a way to observe changes interactively, the `deprioritize` function may be of use, to cause Shake to build some unimportant rules last.

This work was supported by [Digital Asset](https://www.digitalasset.com/).

## Changes to Builtin Rules

Most users shouldn't need to define their own types of rules, but for those who do, the biggest improvement is probably the better documentation in `Development.Shake.Rule`, complete with a worked example. At the same time, the builtin rules have changed slightly in incompatible ways - the docs explain the new state. These changes are intended to set the stage for Cloud Shake, following the pattern described in [Build Systems a la Carte](https://github.com/snowleopard/build/releases/download/icfp-final/build-systems.pdf). I hope that a forthcoming release of Shake will provide an actual implementation of Cloud Shake.
