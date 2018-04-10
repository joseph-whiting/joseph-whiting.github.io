---
---

## What do I want to test?
As part of writing a Scala Graphql client library my plan is to generate Scala code (from API schemas) which will allow typesafe and easy sending of queries and usage of the responses. It should provide a cool Domain Specific Language (DSL) that offers these compile-time checks. In my initial implementation there are seperate classes for reading the schema into the internal model and for generating the DSL code from that model. This post is about how best to test the code generation step.

There are several ways to generate Scala code programatically, they're nicely summarised and discussed [here](http://yefremov.net/blog/scala-code-generation/). The basic gist is that an Abstract Syntax Tree (AST) is a structured representation of Scala source code, which the Scala compiler to then type-check and compile. So your options are
1. Generate source code with string formatting &rarr; let the compiler generate the AST (little bit fiddly, but simple)
2. Generate AST with [treehugger](http://eed3si9n.com/treehugger/) &rarr; generate source from that &rarr; let the compiler generate the AST again from that (bit safer and cleaner, bit more complicated)
3. (curve ball) don't generate source code at all, make something that intercepts the AST on its way to be compiled and changes it (a Scala macro - bit of a steep learning curve, complicated and the Scala macro world seems to be changing a lot)

I'll have to decide which of those to use, but that's not what this post is about! What do I want to test here? Similarly there are three options as I see them:
* Test the source code that is generated using regexp (can test options 1 and 2, but will probably lead to brittle tests when I change anything about the generated code, even just formatting, and I can imagine this being a lot of work - I don't want to write the Scala compiler in regexp)
* Test a produced AST, either by parsing source code or before an AST gets converted to source code (can test all 3 options with a little rejigging, won't break on formatting changes but probably a little bit fiddly and will break if the generated DSL changes in any non-superficial way)
* Test by writing code using the DSL (can test all 3 implementations similarly well, easy to read and write, only breaks if the generated DSL changes in a way that will bother its consumers, will provide an up-to-date documentation of how to use the DSL)

To me the third is the obvious choice. Its good aspects are mostly down to the fact that it is testing the externals of the class/function and not the internals. In particular, the logic about how the generated source code will actually provide a DSL will be expressed in the generator class. If I bake that logic into my tests by testing the source code or AST on a low level then I won't be testing that the DSL actually works properly. The fact that the generated DSL code is looking like it will have to make heavy use of lots of generics and implicits seals the deal - I really want to know that stuff works.
It's possible that this generator class/function is too large and needs unit testing on a lower level in smaller functions, but if that happens I think these tests would still serve as some very effective integration tests (if I'm wrong about this then I'd love to be told so - get in touch!). 

It's pretty neat that this shows that the process of Test Driven Development worked its wonders here. By not having chosen an implementation yet I made myself take a while to think about a flexible testing method, and ended up with a really good one. One could've arrived here after writing an implementation anyway by being clever, but why bother when being stupid is so easy?

## Exactly how do I test that?

If it still seems pretty murky what I mean by this testing approach then perhaps some examples will help.

My goal for the DSL is that for a schema like 
{% highlight graphql %}
type Character {
  name: String
  age: Int
}
{% endhighlight %}
it should be possible to write Scala code something like
{% highlight scala %}
val response = client.send(query {
  character {
    name
  }
})
response.map(_.characters.map(_.name))
{% endhighlight %}
to get a future wrapping a list of names. It should not be possible to write ```response.map(_.characters.map(_.age))``` (by which I mean it should fail to compile). So what I want is something to test compilation, surely an assertion in ScalaTest called [assertCompiles](http://www.scalatest.org/user_guide/using_assertions) will be my hero?
Unfortnately not. When I eagerly wrote a test that called that assertion on the result of my code generation the compiler was quick to complain that ```assetCompiles``` only takes string literals because underneath it's a macro. At compile time it tries to convert the AST from a string literal to a pass if the code compiles or a fail if it doesn't, which then shows up at runtime in the tests. So it was back to the drawing board (by which I mean Google and StackOverflow).

It turns out that what I wanted was Scala's reflection library, which (amongst other things) will allow you to compile strings at runtime.
