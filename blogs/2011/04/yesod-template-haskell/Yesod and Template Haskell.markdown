There's a common phrase that I hear with regard to Yesod: it uses too much Template Haskell/Quasi Quoting. I just want to give a quick explanation of *why* this is the case.

I think the number one distinguishing feature of Yesod is type safety. To my knowledge, no other web framework on the planet has the combination of type safe URLs, type-checked HTML/CSS/Javascript and a type-safe persistence layer. However, this requires a lot of work to get right.

For instance, take HTML. Another possible approach is something like blaze-html, where you define your entire HTML file as Haskell code. This is a nice approach, and I use it on a few projects. But it's not a template language. In order to get type safety from a template language, you need some form a code generation. So for that, we use Template Haskell.

Another is type-safe URLs. If you want this incredibly useful feature, you need to define your route datatype, your render function, and your dispatch function. And you need to make sure that you keep the render/dispatch functions coordinated. That's a tedious, error-prone task. You can do it, but Template Haskell saves you a lot of pain.

So it's true that Yesod uses Template Haskell. But every time we do it, we have a very good reason to do so, and it lets the programmer keep type safety without the error prone boiler-plate that generally accompanies it. You could try to bolt these features onto a different core that doesn't support them out of the box, but it will likely be a painful process.
