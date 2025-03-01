A number of months ago, I got a feature request from Ben Boeckel to add reverse dependency listings to [packdeps](http://packdeps.haskellers.com/). I will admit that, for some reason, I didn't really understand the request at the time, and really had no idea how I would implement it.

Then I got lucky: in the same week, Ben asked me if I had made any progress on this feature, and Simon Hengel announced [HackageOneFive](http://haskell.1045720.n5.nabble.com/ANN-HackageOneFive-Reverse-dependency-lookup-for-all-packages-on-Hackage-td3367669.html). After some follow up with Ben, and some advice from Simon, I was finally able to implement the [reverse dependencies packdeps](http://packdeps.haskellers.com/reverse).

Just to head off the inevitable question: no, this is not the exact same thing as Hackage 2.0. The main goal of reverse packdeps is to tell you which packages rely on an *old* version of your package. In Ben's case, this was so that when making packages for Fedora, it would be easy to find out what libraries needed to be updated. Hopefully package maintainers will also find this useful to know if their new versions are catching on.

If anyone has ideas for improvement, let me know.
