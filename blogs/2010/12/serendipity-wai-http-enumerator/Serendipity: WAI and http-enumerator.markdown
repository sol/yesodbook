It's always an amazing feeling when you realize that two problems you are working on have the same solution. In this case, I was separately trying to figure out a better interface for the http-enumerator library, and migrate WAI over to a combination of blaze-builder and the enumerator package.

The problem with http-enumerator is that it's *not* an enumerator: it would make sense for the http function to in fact create an enumerator that feeds data into a user's iteratee. However, there's a bit of a snag: we have some extra information in the way of the HTTP status code and response headers. So currently, the type signature of the http function is:

    http :: MonadIO m => (Int -> Headers -> Iteratee ByteString m a) -> Request -> m a

In other words, you give it a function that produces an Iteratee, and it will run that Iteratee for you. This approach makes it impossible to send more data to the iteratee after the http function is complete, and makes it more difficult to use standard tools like enumeratees. I called a stalemate on this issue about 2 months ago.

More recently, I was working on the issue of rendering content within an enumerator. After some emails with Simon Meier, John Millikan and Gregory Collins, I decided to not only use blaze-builder-enumerator for generating XML content (my original goal), but to try it out for the WAI as well. The most obvious approach was to define a response as:

    data Response = Response Status ResponseHeaders (Enumerator Builder IO ())

Well, things are actually a little more complicated than this, since we have support for sending files and using lazy bytestrings as well, but that's not the point at hand. This seemed to be a pretty good fit, and I decided that it would be great to demonstrate using the two libraries together with a proxying web server: it will take any request, forward it to a different website, and return the result to you.

Unfortunately, we have a big mismatch between the two APIs. There are in fact two major problems:

* http-enumerator does not supply an enumerator at all, which is exactly what the WAI wants.

* The WAI expects to know the status code and response headers before the enumerator is run. However, in the case of a proxy, we won't know those until *after* the enumerator has started.

It turns out that both of these issues have to do with passing extra information in the presence of enumerators. I explored a number of different approaches to get around the problem, but it turns out that my original type signature for the http function was *almost* right on the money. Let's remind ourselves of what an enumerator is:

    type Enumerator a m b = Step a m b -> Iteratee a m b

It's a function that transforms steps into iteratees. But an iteratee is really just a wrapper around a step in a monad. So after a little comprimising with myself, I came to this new type signature:

    http :: MonadIO m => Request -> (Int -> Headers -> Iteratee ByteString m a) -> Iteratee ByteString m a

I've put the Request at the beginning, but that's not the important part. This function now returns an iteratee instead of running the computation entirely. This allows it to be resumed later on by another enumerator. This in turn gave me the idea that our response in WAI could be modeled as:

    (Status -> ResponseHeaders -> Iteratee Builder IO a) -> Iteratee Builder IO a

Unfortunately, this time around returning the iteratee is *not* a good idea: it prevents the usage of enumeratees which change the stream type. That will virtually *always* be the case, since we need to use blaze-builder-enumerator to render the Builders into ByteStrings. So instead, applications need to run the iteratee themselves:

    (Status -> ResponseHeaders -> Iteratee Builder IO a) -> IO a

There's something tricky going on here: that type variable "a" makes sure that the application actually runs the iteratee, because otherwise it will have no way of returning a value of type IO a. Well, short of using bottom or exceptions, but that's a different issue.

Since the type signatures for these two things were becoming increasingly similar, I decided to bridge the gap a little more. The WAI datatype Status contains both the numeric status code and the textual status message, while the ResponseHeaders datatype uses a case-insensitive bytestring for the keys. http-enumerator now uses both of those, giving us a final type signature of:

    http :: MonadIO m
         => Request
         -> (Status -> ResponseHeaders -> Iteratee S.ByteString m a)
         -> Iteratee S.ByteString m a

So the three important differences between these two types is:

1) http uses ByteStrings, since the content is meant to be read. WAI responses use Builders, since the content is meant to be written.

2) http returns an iteratee so the computation can be resumed by another enumerator. WAI responses return a computed value to avoid issues with the stream type.

3) http lives in any arbitrary MonadIO, while WAI responses only live in the IO monad. There's no way for a WAI handler to automatically unwrap arbitrary monads.

All of these changes are available in respective github repos: [http-enumerator](https://github.com/snoyberg/http-enumerator/tree/enum-not-iter), [wai](https://github.com/snoyberg/wai/tree/ver0.3) and [wai-extra](https://github.com/snoyberg/wai-extra/tree/wai0.3) (for the SimpleServer). I'm far from sold on these API changes, but I did think the concept interesting enough to warrant discussing them now. I'm very interested in work others have done with enumerators and extra data.

Oh, and with these changes, it was a piece of cake to write that proxy application. I used the [Yesod Wiki](http://wiki.yesodweb.com/) since it uses relative URLs for links, though any site should be fine.

    {-# LANGUAGE OverloadedStrings #-}
    import qualified Network.HTTP.Enumerator as H
    import qualified Network.Wai as W
    import Network.Wai.Handler.SimpleServer (run)
    import qualified Data.ByteString.Char8 as S8
    import qualified Data.ByteString.Lazy.Char8 ()
    import Data.Enumerator (($$), joinI, run_)
    import Blaze.ByteString.Builder (fromByteString)
    import qualified Data.Enumerator as E

    main :: IO ()
    main = run 3000 app

    app :: W.Application
    app W.Request { W.pathInfo = path } =
        case H.parseUrl $ "http://wiki.yesodweb.com" ++ S8.unpack path of
            Nothing -> return notFound
            Just hreq -> return $ W.ResponseEnumerator $ run_ . H.http hreq . go
      where
        go f s h = joinI $ E.map fromByteString $$ f s $ filter safe h
        safe (x, _) = not $ x `elem` ["Content-Encoding", "Transfer-Encoding"]

    notFound :: W.Response
    notFound = W.ResponseLBS W.status404 [("Content-Type", "text/plain")] "Not found"

Starting with the less-interesting stuff, main is just a standard call to SimpleServer, notFound is a basic 404 response, safe strips out any response headers that would alter how a client would understand the response delivery itself, and parseUrl simply converts a String into a http-enumerator Request.

The interesting stuff is the go function and the line that returns the ResponseEnumerator. The go function converts an iteratee that takes a stream of Builders into an iteratee that takes a stream of ByteStrings. It also needs to address the extra "metadata" of the status and response headers.

Once the conversion is done, it's easy to plug that iteratee into the http function to produce a final iteratee, and use the run_ function from enumerator to run the iteratee.

By the way, this example was originally going to involve demonstrating XML parsing and rendering as well, but I think I'll save that one for another blog post.
