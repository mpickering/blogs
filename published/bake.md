# Bake - Continuous Integration System

_Summary: I've written a continuous integration system, in Haskell, designed for large projects. It works, but probably won't scale yet._

I've just released [bake](https://github.com/ndmitchell/bake#readme), a continuous integration system - an alternative to Jenkins, Travis, Buildbot etc. Bake eliminates the problem of "broken builds", a patch is never merged into the repo before it has passed all the tests.

Bake is designed for large, productive, semi-trusted teams:

* _Large teams_ where there are at least several contributors working full-time on a single code base.
* _Productive teams_ which are regularly pushing code, many times a day.
* _Semi-trusted teams_ where code does not go through manual code review, but code does need to pass a test suite and perhaps some static analysis. People are assumed to be fallible.

**Current state:** At the moment I have a rudimentary test suite, and it seems to mostly work, but Bake has never been deployed for real. Some optional functionality doesn't work, some of the web UI is a bit crude, the algorithms probably don't scale and all console output from all tests is kept in memory forever. I consider the design and API to be complete, and the scaling issues to be easily fixable - but it's easier to fix after it becomes clear where the bottleneck is. If you are interested, take a look, and then [email me](http://community.haskell.org/~ndm/contact/).

To give a flavour, the web GUI looks of a running Bake system looks like:

![](https://raw.githubusercontent.com/ndmitchell/bake/master/screenshot-part.png)

## The Design

Bake is a Haskell library that can be used to put together a continuous integration server. To run Bake you start a single server for your project, which coordinates tasks, provides an HTTP API for submitting new patches, and a web-based GUI for viewing the progress of your patches. You also run some Bake clients which run the tests on behalf of the server. While Bake is written in Haskell, most of the tests are expected to just call some system command.

There are a few aspects that make Bake unique:

* Patches are submitted to Bake, but are not applied to the main repo until they have passed all their tests. There is no way for someone to "break the build" - at all points the repo will build on all platforms and all tests will pass.
* Bake scales up so that even if you have 5 hours of testing and 50 commits a day it will not require 250 hours of computation per day. In order for Bake to prove that a set of patches pass a test, it does not have to test each patch individually.
* Bake allows multiple clients to run tests, even if some tests are only able to be run on some clients, allowing both parallelisation and specialisation (testing both Windows and Linux, for example).
* Bake can detect that tests are no longer valid, for example because they access a server that is no longer running, and report the issue without blaming the submitted patches.


## An Example

The test suite provides both [an example configuration](https://github.com/ndmitchell/bake/blob/master/src/Example.hs) and [commands to drive it](https://github.com/ndmitchell/bake/blob/master/src/Test.hs). Here we annotate a slightly simplified version of the example, for lists of imports see the original code.

First we define an enumeration for where we want tests to run. Our server is going to require tests on both Windows and Linux before a `patch` is accepted.

    data Platform = Linux | Windows deriving (Show,Read)
    platforms = [Linux,Windows]

Next we define the `test` type. A `test` is something that must pass before a `patch` is accepted.

    data Action = Compile | Run Int deriving (Show,Read)

Our type is named `Action`. We have two distinct types of tests, compiling the code, and running the result with a particular argument. Now we need to supply some information about the tests:

    allTests = [(p,t) | p <- platforms, t <- Compile : map Run [1,10,0]]
    
    execute :: (Platform,Action) -> TestInfo (Platform,Action)
    execute (p,Compile) = matchOS p $ run $ do
        cmd "ghc --make Main.hs"
    execute (p,Run i) = require [(p,Compile)] $ matchOS p $ run $ do
        cmd ("." </> "Main") (show i)

We have to declare `allTests`, then list of all tests that must pass, and `execute`, which gives information about a test. Note that the `test` type is `(Platform,Action)`, so a test is a platform (where to run the test) and an `Action` (what to run). The `run` function gives an `IO` action to run, and `require` specifies dependencies. We use an auxiliary `matchOS` to detect whether a test is running on the right platform:

    #if WINDOWS
    myPlatform = Windows
    #else
    myPlatform = Linux
    #endif

    matchOS :: Platform -> TestInfo t -> TestInfo t
    matchOS p = suitable (return . (==) myPlatform)

We use the `suitable` function to declare whether a test can run on a particular client. Finally, we define the `main` function:

    main :: IO ()
    main = bake $
        ovenGit "http://example.com/myrepo.git" "master" $
        ovenTest readShowStringy (return allTests) execute
        defaultOven{ovenServer=("127.0.0.1",5000)}

We define `main = bake`, then fill in some configuration. We first declare we are working with Git, and give a repo name and branch name. Next we declare what the tests are, passing the information about the tests. Finally we give a host/port for the server, which we can visit in a web browser or access via the HTTP API.


## Using the Example

Now we have defined the example, we need to start up some servers and clients using the command line for our tool. Assuming we compiled as `bake`, we can write `bake server` and `bake client` (we'll need to launch at least one client per OS). We can view the state by visiting `http://127.0.0.1:5000` in a web browser.

To add a patch we can run `bake addpatch --name=cb3c2a71`, using the SHA1 of the commit, which will try and integrate that patch into the `master` branch, after all the tests have passed.
