# Maximum Patches, Minimum Time

_Summary: Writing a fast continuous integration server is tricky._

When I started writing the continuous integration server [Bake](https://github.com/ndmitchell/bake), I thought I had a good idea of what would go fast. It turned out I was wrong. The problem Bake is trying to solve is:

* You have a set of tests that must pass, which take several hours to run.
* You have a current state of the world, which passes all the tests.
* You have a stream of hundreds of incoming patches per day that can be applied to the current state.
* You want to advance the state by applying patches, ensuring the current state always passes all tests.
* You want to reject bad patches and apply good patches as fast as possible.

I assume that tests can express dependencies, such as you must compile before running any tests. The test that performs compilation is special because it can go much faster if only a few things have changed, benefiting from incremental compilation.

Both my wrong solution, and my subsequent better solution, are based on the idea of a candidate - a sequence of patches applied to the current state that is the focus of testing. The solutions differ in when patches are added/removed from the candidate and how the candidates are compiled.

#### A Wrong Solution

My initial solution compiled and ran each candidate in a separate directory. When the directory was first created, it copied a nearby candidate to try and benefit from incremental compilation.

Each incoming patch was immediately included in the candidate, compiled, and run on all tests. I would always run the test that had not passed for the longest time, to increase confidence in more patches. Concretely, if I have run test `T1` on patch `P1`, and `P2` comes in, I start testing `T2` on the combination of `P1+P2`. After that passes I can be somewhat confident that `P1` passes both `T1` and `T2`, despite not having run `T2` on just `P1`.

If a test fails, I bisect to find the patch that broke it, reject the patch, and immediately throw it out of the candidate.

#### The Problems

There are three main problems with this approach:

* Every compilation starts with a copy of a nearby candidate. Copying a directory of lots of small files (the typical output of a compiler) is fantastically expensive on Windows.
* When bisecting, I have to compile at lots of prefixes of the candidate, the cost of which varies significantly based on the directory it starts from.
* I'm regularly throwing patches out of the candidate, which requires a significant amount of compilation, as it has to recompile all patches that were after the rejected patch.
* I'm regularly adding patches to the candidate, each of which requires an incremental compilation, but tends to be dominated by the directory copy. 

This solution spent all the time copying and compiling, and relatively little time testing.

#### A Better Solution

To benefit from incremental compilation and avoid copying costs, I always compile in the same directory. Given a candidate, I compile each patch in the series one after another, and after each compilation I zip up the interesting files (the test executables and test data). To bisect, I unzip the relevant files to a different directory. On Windows, unzipping is much more expensive than zipping, but that only needs to be done when bisecting is required. I also only need to zip the stuff required for testing, not for building, which is often much smaller.

When testing a candidate, I run all tests without extending the candidate. If all the tests pass I update the state and create a new candidate containing all the new patches.

If any test fails I bisect to figure out who should be rejected, but don't reject until I've completed all tests. After identifying all failing tests, and the patch that caused each of them to fail, I throw those patches out of the candidate. I then rebuild with the revised candidate and run only those tests that failed last time around, trying to seek out tests where two patches in a candidate both broke them. I keep repeating with only the tests that failed last time, until no tests fail. Once there are no failing tests, I extend the candidate with all new patches, but do not update the state.

As a small tweak, if there are two patches in the queue from the same person, where one is a superset of the other, I ignore the subset. The idea is that if the base commit has an error I don't want to track it down twice, once to the first failing commit and then again to the second one.

#### Using this approach in Bake

First, the standard disclaimer: Bake may not meet your needs - it is a lot less developed than other continuous integration systems. If you do decide to use Bake, you should run from the git repo, as the Hackage release is far behind. That said, Bake is now in a reasonable shape, and might be suitable for early adopters.

In Bake this approach is implemented in the `StepGit` module, with the `ovenStepGit` function. Since Bake doesn't have the notion of building patches in series it pretends (to the rest of Bake) that it's building the final result, but secretly caches the intermediate steps. If there is a failure when compiling, it caches that failure, and reports it to each step in the bisection, so Bake tracks down the correct root cause.

I am currently recommending `ovenStepGit` as the "best" approach for combining `git` and an incremental build system with Bake. While any incremental build system works, I can't help but plug [Shake](http://shakebuild.com), because its the best build system I've ever written.
