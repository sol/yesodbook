## Yesod Release Candidate 3 is out! 

Thanks to those who spotted issues with the scaffolding in the previous release candidates. So far no issues have been reported in Yesod proper, so if we keep the streak going through the weekend we will put out the official release.

Also, we are going to version bump from 0.9.0 to 0.9.1 after the release so that you don't have to blow away your .ghc folder when the release comes out. So now you have no excuse to avoid upgrading right now.


To get the latest RC, add this to your ~/.cabal/config:

    remote-repo: yesod-yackage:http://yackage.yesodweb.com

And run:

    cabal update && cabal install 'yesod >= 0.9.0'


# More Information on upgrading your site

First, see the [previous upgrading post](http://www.yesodweb.com/blog/2011/08/yesod-0-9-release-candidate), which tackles most of the upgrade issues. I will first mention a couple more changes I had to do, that will be obvious when you compile:

* import Text.Blaze.Renderer.String (renderHtml) -> import Text.Blaze.Renderer.Text (renderHtml)
* tabs are not allowed in hamlet

## Extra Hamlet Upgrade Info

A big change not mentioned in specific detail is changing the use of [hamlet||]. You should probably just skip this paragraph and read the [new documentation](http://www.yesodweb.com/show/map/118), which covers how to use Yesod 0.9. When you upgrade your site, you may have to use the terms ihamlet, whamlet, or even shamlet. Which one of these funny names should you use? It is pretty simple, the first letter stands for something. `ihamlet` isn't an Apple device that makes pork, it just allows internationalization interpolation in hamlet.

QuasiQuoter Prefix letter         Interpolation
----------- --------------------- -------------
shamlet     s = simple            #{} 
hamlet      none                  #{} @{} 
ihamlet     i = internationalized #{} @{} ^{} _{} 
whamlet     w = widgets           #{} @{} ^{} _{}

And you need to prefix using these templates with toWidget (or if you were using addHtml or something similar, remove that and use toWidget). This all seems like extra work, but we are hoping it will pay off with understandable error messages.

## Scaffolding Changes

One of the reasons the scaffolding had a couple issues in the RC is because of all the great stuff that has been added to it. We are going to work on taking some of the code out of the scaffold and into the library, but much of this does belong in the scaffold to be configured by the user. These changes should effect just a few key common files instead of your entire site, but they are also a more error-prone and manual process. You don't have to do all of this conversion now if you don't need the features. But I highly recommend getting the new logger working properly, even if you don't need all the new configurable settings.

I recommend generating a new scaffold with the same foundation type, and copying over the new functionality. Luckily we know the compiler will guide you with your changes.

### Settings.hs, and cabal file

* copy over config/settings.yml, and change the settings appropriately (to match what is in your Settings.hs file)
* copy over Settings.hs, but you have to manually inspect against your config/Settings.hs and make sure you don't lose any needed customizations.
* copy over main.hs (previously it was named after your project and located in the config dir), and change your cabal file so the executable points to main.hs. Likely you don't have anything you care about that gets overwritten (but double check).
* while in your cabal file, add dependencies for cmd-args, data-object, data-object-yaml, yesod-core, shakespeare-text, and unix (if you are on a unix based system).
* move config/StaticFiles.hs to Settings/StaticFile.hs (create the Settings directory first) and then you can remove config from the hs-source-dirs section of your cabal file. This can make life easier for some IDEs.

### Controller.hs -> Application.hs

* rename Controller.hs to Application.hs

The best approach may be to copy over the Application.hs file from a new scaffolded site. Here are instructions for making indifidual changes:

* approot = Settings.appRoot . settings
* add to Application.hs:

~~~~~~~~{.haskell}
      import Yesod.Logger (makeLogger, flushLogger, Logger, logLazyText, logString)
      import Network.Wai.Middleware.Debug (debugHandle)
      -- only if on unix
      import qualified System.Posix.Signals as Signal
      import Control.Concurrent (forkIO, killThread)
      import Control.Concurrent.MVar (newEmptyMVar, putMVar, takeMVar)
~~~~~~~~

* copy the new withFOUNDATION handlers from the new scaffolding site you generated.

### foundation file -> Foundation.hs

* rename your foundation type file (the one with `instance Yesod` to Foundation.hs. This isn't necessary, but will allow you to communicate better with others about your site.

* add these to your foundation type

~~~~~~~~{.haskell}
      settings :: Settings.AppConfig
      getLogger :: Logger
~~~~~~~~~~

* add to your imports

~~~~~~~~{.haskell}
   import Yesod.Logger (Logger, logLazyText)
~~~~~~~~~~

* add to your Yesod instance

~~~~~~~~{.haskell}
   messageLogger y loc level msg =
         formatLogMessage loc level msg >>= logLazyText (getLogger y)
~~~~~~~~~~

When this is all working, you should see all requests logged, and logging messages should be thread-safe, and you should be able to configure your application as you see fit. We will go over some more aspects of these features in the release announcement.

