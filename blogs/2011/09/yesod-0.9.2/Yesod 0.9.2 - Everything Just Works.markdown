We are pleased to announce the release of Yesod version 0.9.2.
This release includes some important new features.

## Re-compiling your application

One of the worst aspects of developing a web application in Haskell is constantly re-compiling. Our solution to this in Yesod is the "yesod devel" server. It automatically re-compiles your application when a file changes and restarts your application. Unfortunately it always had problems, particularly on Windows. Luite Stegeman committed an improved version that should just work. Let us know if you have any issues. The new devel server takes a --dev flag to use cabal-dev, which is what I was most excited about. Yesod users can finally work on one application without worrying about breaking another. Using this feature requires no changes to your application code, and nothing different than using the existing yesod devel server. Just `cabal update && cabal install yesod`.

One outstanding issue is re-compiling (shakeapeare/hamlet) compile-time templates. This should work fine as long as you use the new yesod devel and stick to the defaults, but it is still brittle. So we are [solving this issue in GHC](http://www.google.com/url?sa=D&q=http://hackage.haskell.org/trac/ghc/ticket/4900). The Simons have proposed a solution and believe this to be a small and relatively easy to implement change. They are still hoping for us to step up and make it, and we are so busy pushing Yesod along that we are hoping someone else might feel like learning a little bit about GHC. If you are interested in helping out let us know, but we will make sure this gets done one way or another.

Happstack has also come up with some good techniques to reload an application. They can detect file changes with inotify on Linux, whereas we use polling in Yesod. Happstack also uses plugins to reload code. This has always been a fragile approach, but Hapsstack has recently made some improvements on it. Yesod's approach is cross-platform and reliable, but Happstack's approach performs better when it works. We are going to work together with the Happstack maintainers to combine our efforts in this area.


## MongoDB

The Persistent MongoDB backend has always been on tenuous grounds. Today it finally just works. There is even a scaffolding option you can use to generate a site that uses MongoDB as a backend.

MongoDB is a high performance database designed for scalability. SQL is a much more flexible way to store data- you can always join one table to another to perform complex ad-hoc queries- this is particularly useful for creating reports and summarizing your application data. However, reporting or other reasons for ad-hoc queries are not a strong requirement for most applications. If this is the case then one can design a schema in MongoDB where the query patterns are frozen. Instead of joining data structures, one embeds a data structure within another. Embedding is very fast and easier to scale. And you can embed lists or maps, something sorely missing from most SQL databases. This persistent release still has limitations in its support for embedding. You can embed Haskell lists and maps, but not records.

## Recent discussions

### Records discussion

I enjoyed the discussion around the [last blog post](http://www.yesodweb.com/blog/2011/09/limitations-of-haskell). My hope is that we can keep the records discussion alive and solve that issue in the Haskell language. I certainly learned a lot more about the issue. Various [techniques have been proposed for dealing with the existing records situation](http://www.yesodweb.com/wiki/record-hacks). They all start with each record declared in its own file and imported (qualified when necessary) instead of being prefixed. In Yesod (Persistent actually), records represent database information and we prefix them with the table name. However, this was mostly for reasons that no longer matter after the 0.9 release. So we are going to look into declaring records in different files and some of the techniques to make records play more nicely with each other.


### Yesod discussions

Expect some improvements in Yesod's Internationalization for determining which languages are supported. There have also been a fair amount of discussions around forms lately on how to satisfy complex use-cases where fields are dependent upon eachother. Unfortunately there are no elegant solutions to this yet.

### Websockets and Event-Stream

These seem to be on people's mind. And for good reason, because Haskell is the best language to support these highly concurrent technologies. We are looking forward to see what people come up with and eager to help them out.

## Yesod has very few bugs

Now that yesod devel and MongoDB work properly, I feel like Yesod is finally stable now. No more buggy or broken things hanging around. Yesod encompasses a lot of code and a lot of features. The 0.9 release introduced a lot of changes and features, and has already seen a lot of users. With all these changes in a large code-base, we saw only 2 bugs in the framework reported, and neither of them were new to this release. Besides those 2 framework bugs, the rest of the bugs were in the code generated from the scaffolding tool (code generator for new sites). The scaffolding system is inherently fragile, but an indispensable convenience.

Yesod's increasing maturity is helping with stability. The maturing process represents more users exercising our code, but also a maturing of our tooling and development approach. Most of the Yesod code base now has solid testing behind it. We always view the type system and compiler as our first line of defense. However, correctness is usually only encoded in the type system at a low-level - functions work properly and interact well with data and one-another. When we need increased assurance at this level QuickCheck is an amazing tool. But generally we skimp on low-level unit tests, and write more api level integration tests. We find this gives us great value for the amount of effort put into testing. The new cabal test feature is our ally in this effort, and [hspec](http://github.com/trystan/hspec) helps us write well described tests with the minimum amount of cruft.

On the whole, I think the stability Yesod has achieved is amazing! Haskell lets us release some rock solid code.