One feature that's been requested for the Yesod book, and which I've wanted to add for a while, is paragraph-level comments like [Real World Haskell](http://book.realworldhaskell.org/). Now that I've migrated the book to an XML format, this feature was relatively easy to add. Please enjoy, and make good use of it! I'll also be adding in Atom feeds for the comments, and maybe even for book updates, so stay tuned.

One little note: the comments only work for the chapters of the book that have been fully translated to XML. Some of the chapters (like [persistent](http://docs.yesodweb.com/book/persistent/)) still contain a lot of content in Markdown, so the comments links do not appear on those paragraphs.

## New Site Design

I'd like to make a request from the community now: I think it's time to add some more style to this website. If anyone has some new site design ideas, please send them over. All part of the preparing-for-one-point-oh thing.

Oh, and good job everyone on the "Please Break Yesod" campaign, I'm getting bug reports and **lots** of good API suggestions. Some people are really putting together some creative packages on top of Yesod, and I look forward to sharing them with you as they mature (the packages, not the people).

## Technical details

In case anyone is interested, some minor details on how I implemented the comment system. I decided to just keep all the comments in memory in a TVar, since they are likely going to not take up much memory and be mostly read-only. I defined a simple datatype, and created some cereal instances to encode/decode with bytestrings.

Each time a comment is added, I update the TVar with the new comment and write it to file, using cautious-file to make sure I don't accidently the whole comments database. I will probably start doing regular backups of that file as well.

As usual, the whole site's code is [available on Github](https://github.com/snoyberg/yesoddocs), so check it out if you are interested.
