# HLint 3.0

_Summary: HLint 3.0 uses the GHC parser._

In [June 2019](https://neilmitchell.blogspot.com/2019/06/hlints-path-to-ghc-parser.html) I posted about our intention to move HLint to the GHC parser. Since then a small group of us have been hard at work making the conversion -- first by parsing with both GHC and haskell-src-exts, and finally, with the newly released [HLint 3.0](https://hackage.haskell.org/package/hlint), parsing _only_ with [GHC](https://www.haskell.org/ghc/). As of now, if your code can be parsed with GHC, it can probably be parsed with HLint. As new GHC releases come out, with new features and new forms of syntax, HLint will follow along closely.

The [change list for this release](https://github.com/ndmitchell/hlint/blob/master/CHANGES.txt) records 51 separate items, which is about as many as the last nine HLint releases combined. Of those changes, 11 are breaking changes (the ones marked with `*`). That count omits all the hint conversions, which (hopefully!) aren't user visible. The main API breaks are in the `Language.Haskell.HLint` API, which has switched from [haskell-src-exts types](https://hackage.haskell.org/package/haskell-src-exts) to GHC ones. You can now take a GHC syntax tree and apply HLint to it, or (as before) give HLint the source and have it do the parsing for you. We also took the opportunity to simplify the API while we were at it -- but the underlying functionality remains much the same. We also deleted a small number of command line flags that were no longer useful, and were never used very much. If you have difficulty converting to the new API or relied on some removed functionality, [raise a bug](https://github.com/ndmitchell/hlint/issues).

What was especially nice about this conversion process, and the development of HLint in general, is that it is increasingly becoming a real team, where my role is more reviewer than coder. There have been [21 distinct contributors](https://github.com/ndmitchell/hlint/graphs/contributors?from=2019-06-10&to=2020-05-13&type=c) since the start of the GHC conversion, but I'd like to particularly call out a few major pieces of work that have been completed:

* [Shayne Fletcher](https://github.com/shayne-fletcher) is responsible for the [`ghc-lib-parser` library](https://hackage.haskell.org/package/ghc-lib-parser) that makes it feasible to use a single GHC API across multiple GHC versions. Without that, using the GHC library would be at least double the work (and just wouldn't be feasible). Shayne also did a lot of the conversion, mapping many of the rule types from haskell-src-exts to GHC. These API's are surprisingly different given they have the same underlying representation.
* [Georgi Lyubenov](https://github.com/googleson78) did most of the conversions that Shayne didn't do.
* [Joseph C. Sible](https://github.com/josephcsible) has made sure that while efforts were focused on a complete rewrite of the code base, the underlying hints have continued to improve, removing incorrect hints, adding useful additional hints.
* [Ziyang Liu](https://github.com/zliu41) has focused on the refactoring side of HLint. The [initial refactoring work](https://mpickering.github.io/gsoc2015.html) was completed by Matthew Pickering as part of GSoC 2015. Since then it's had mild attention at best. Ziyang has stepped into the gap, importantly adding tests, CI and improving the refactorings in lots of places. It now feels like a real part of HLint.

We hope you enjoy HLint 3.0 and beyond!

PS. You may spot we're already on [HLint 3.0.2 on Hackage](https://hackage.haskell.org/package/hlint) - thanks to [Ryan Scott](https://github.com/RyanGlScott) for already finding a few bugs.