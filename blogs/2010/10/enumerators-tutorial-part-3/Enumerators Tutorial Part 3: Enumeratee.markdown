
This content is now part of the [Yesod book](http://www.yesodweb.com/book/enumerator). It is recommended to read there, since the content is more up-to-date.

This is part 3 of a series of tutorials on the [enumerator package](http://hackage.haskell.org/package/enumerator). As usual, the code for this part of the tutorial is available as [a github gist](http://gist.github.com/615786).

## Generalizing getNumberEnum

In [part 2](http://docs.yesodweb.com/blog/enumerators-tutorial-part-2/) of this series, we created a getNumberEnum function with a type signature:

    getNumberEnum :: MonadIO m => Enumerator Int m b

If you don't remember, this means getNumberEnum produces a stream of <code>Int</code>s. In particular, our getNumberEnum function read lines from stdin, converted them to ints and fed them into an iteratee. It stopped reading lines when it saw a "q".

But this functionality seems like it could be useful outside the realm of Ints. We may like to deal with the original Strings, for example, or Bools, or a bunch of other things. We could easily define a more generalized function which simply doesn't do the String to Int conversion:

    lineEnum :: MonadIO m => Enumerator String m b
    lineEnum (Continue k) = do
        x <- liftIO getLine
        if x == "q"
            then continue k
            else k (Chunks [x]) >>== lineEnum
    lineEnum step = returnI step

Cool, let's plug this into our sumIter function (I've renamed the sum6 function from the previous two parts):

    lineEnum $$ sumIter

Actually, that doesn't type check: lineEnum produces <code>String</code>s, and sumIter takes <code>Int</code>s. We need to modify one of them somehow.

    sumIterString :: Monad m => Iteratee String m Int
    sumIterString = Iteratee $ do
        innerStep <- runIteratee sumIter
        return $ go innerStep
      where
        go :: Monad m => Step Int m Int -> Step String m Int
        go (Yield res _) = Yield res EOF
        go (Error err) = Error err
        go (Continue k) = Continue $ \strings -> Iteratee $ do
            let ints = fmap read strings :: Stream Int
            step <- runIteratee $ k ints
            return $ go step

What we've done here is wrap around the original iteratee. As usual, we first need to unwrap the Iteratee constructor and the monad to get at the heart of the Step value. Once we have that innerStep value, we pass it to the go function, which simply transforms that values in the Stream value from Strings to Ints.

## Even more general

Of course, it would be nice if we could apply this transformation to *any* iteratee. To start with, let's just pass the inner iteratee and the mapping function as parameters.

    mapIter :: Monad m => (aOut -> aIn) -> Iteratee aIn m b -> Iteratee aOut m b
    mapIter f innerIter = Iteratee $ do
        innerStep <- runIteratee innerIter
        return $ go innerStep
      where
        go (Yield res _) = Yield res EOF
        go (Error err) = Error err
        go (Continue k) = Continue $ \strings -> Iteratee $ do
            let ints = fmap f strings
            step <- runIteratee $ k ints
            return $ go step

We could call this like:

    run_ (lineEnum $$ mapIter read sumIter) >>= print

Nothing much to see here, it's basically identical to the previous version. What's funny is that enumerator comes built in with a <code>map</code> function to do just this, but it has a significantly different type signature:

    map :: Monad m => (ao -> ai) -> Enumeratee ao ai m b

since:

    type Enumeratee aOut aIn m b = Step aIn m b -> Iteratee aOut m (Step aIn m b)

that's equivalent to:

    map :: Monad m => (aOut -> aIn) -> Step aIn m b -> Iteratee aOut m (Step aIn m b)

What's with all this extra complication in type signature? Well, it's not necessary for map itself, but it *is* necessary for a whole bunch of other similar functions. But let's focus on this map for a second so we don't get lost: the first argument is the same old mapping function we had before. The second argument is a Step value. This isn't really so surprising: in our mapIter, we took an Iteratee with the same parameters, and we internally just unwrapped it to a Step.

But what's happening with that return value? Remembering the meanings for all these datatypes, it's an Iteratee which will be fed a stream of <code>aOut</code>s and return a Step (aka, a new iteratee, right?). This kind of makes intuitive sense: we've introduced a middle man which accepts input from one source and transforms a Step to a newer state.

But now perhaps the trickiest part of the whole thing: how do we actually *use* this map function? It turns out that an Enumeratee is close enough in type signature to an Enumerator that we can just do:

    map read $$ sumIter

But the type signature on *that* turns out to be a little bit weird:

    Iteratee String m (Step Int m Int)

Remembering that an Iteratee is just a wrapped up Step, what we've got *here* is an iteratee that takes Strings and returns an Iteratee, which in turn takes Ints and produces an Int. Having this fancy result allows us to do one of our great tricks with iteratees: plug in data from multiple sources. For example, we could plug some Strings into this whole ugly thing, run it, get a *new* iteratee which takes Ints, feed *that* some Ints and get an Int result.

(If all that went over your head, don't worry. I won't be talking about that kind of stuff any more.)

But often times, we *don't* need all of that power. We just want to stick our enumeratee onto our iteratee and get a new iteratee. In our case, we want to attach our map onto the sumIter to produce a new iteratee that takes Strings and returns Ints. In order to do that, we need a function like this:

    unnest :: Monad m => Iteratee String m (Step Int m Int) -> Iteratee String m Int
    unnest outer = do -- using the Monad instance of Iteratee
        inner <- outer -- inner :: Step Int m Int
        go inner
      where
        go (Error e) = throwError e
        go (Yield x _) = yield x EOF
        go (Continue k) = k EOF >>== go

We can then run our unholy mess with:

    run_ (lineEnum $$ unnest $ map read $$ sumIter) >>= print

And actually, the unnest function is available in Data.Enumerator, and it's called joinI. So we should really write:

    run_ (lineEnum $$ joinI $ map read $$ sumIter) >>= print

## Skipping

Let's write a slightly more interesting enumeratee: this one skips every other input value.

    skip :: Monad m => Enumeratee a a m b
    skip (Continue k) = do
        x <- head
        _ <- head -- the one we're skipping
        case x of
            Nothing -> return $ Continue k
            Just y -> do
                newStep <- lift $ runIteratee $ k $ Chunks [y]
                skip newStep
    skip step = return step

What's interesting about the approach here is how similar it looks to an Enumerator. We're doing a lot of the same things: checking if the Step value is a Continue; if it's not, then simply return it. Then we capitalize on the Iteratee Monad instance, using the head function to pop two values out of the stream. If there's no more data, we return the original Continue value: just like with an Enumerator, we don't give an EOF so that we can feed more data into the iteratee later. If there is data, we pass it off to the iteratee, get our new step value and then loop.

And what's cool about enumeratees is we can chain these all together:

    run_ (lineEnum $$ joinI $ skip $$ joinI $ map read $$ sumIter) >>= print

Here, we read lines, skip every other input, convert the Strings to Ints and sum them.

## Real life examples: http-enumerator package

I started working on these tutorials as I was working on the [http-enumerator](http://hackage.haskell.org/package/http-enumerator) package. I think the usage of enumeratees there is a great explanation of the benefits they can offer in real life. There are three different ways the response body can be broken up:

* Chunked encoding. In this case, the web server gives a hex string specifying the length of the next chunk and then that chunk. At the end, it sends a 0 to indicate the end of that response.

* Content length. Here, the web server sends a header before any of the body is sent specifying the total length of the body.

* Nothing at all. In this case, the response body lasts until an end-of-file.

In addition, the body may or may not be GZIP compressed. We end up with the following enumeratees, each with type signature <code>Enumeratee ByteString ByteString m b</code>: chunkedEncoding, contentLength and ungzip. We then get to do something akin to:

    let parseBody x =
            if ("transfer-encoding", "chunked") `elem` responseHeaders
                then joinI $ chunkedEncoding $$ x
                else case mlen of
                        Just len -> joinI $ contentLength len $$ x
                        Nothing -> x -- no enumeratee applied at all
    let decompress x =
            if ("content-encoding", "gzip") `elem` responseHeaders
                then joinI $ ungzip $$ x
                else x
    run_ $ socketEnumerator $$ parseBody $ decompress $ bodyIteratee

We create a chain: the data from the server is fed into the parseBody function. In the case of chunked encoding, the data is processed appropriately and then headers are filtered out. If we are dealing with content length, then only the specified number of bytes are read. And in the case of neither of those, parseBody is a no-op.

Whatever the case may be, the raw response body is then fed into decompress. If the body is GZIPed, then ungzip inflates it, otherwise decompress is a no-op. Finally, the parsed and inflated data is fed into the user-supplied bodyIteratee function. The user remains blissfully unaware of any steps the data took to get to him/her.

## Exercises

* Write an enumeratee which takes hex chars (eg, "DEADBEEF") to Word8s. Its type signature should be <code>Enumeratee Char Word8 m b</code>.

* Write the opposite enumeratee, eg <code>Enumeratee Word8 Char m b</code>.

* Create a quickcheck property that ensures that these two functions work correctly.

## Conclusion

* Enumeratees are the pipes connecting enumerators to iteratees.

* The strange type signature of an Enumeratee hides a lot of possible power. Especially notice how similar their type signatures are to Enumerators.

* You can merge an Enumeratee into an Iteratee with <code>joinI $ enumeratee $$ iteratee</code>.

* Don't forget that you can use the Monad instance of Iteratee when creating your own enumeratees.

* You can always compose multiple enumeratees together, such as in http-enumerator.

This concludes the three parts of the tutorial that I'd planned. If people had particular questions or topics they wanted me to cover, just leave a comment or send me an email.
