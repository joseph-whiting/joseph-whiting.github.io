
---
---
##Motivation
I recently started work on a hobby project where I wanted to use [ScalaJS and React](https://github.com/japgolly/scalajs-react). I decided to use [GraphQL](https://graphql.org/) to interface with the backend because of its flexibility. The Apollo organisation do a great set of client libraries for working with GraphQL, including [one for ScalaJS and React](https://github.com/apollographql/react-apollo-scalajs). Unfortunately that was designed for a different ScalaJS/React library than the one I wanted to use. It also didn't do much apart from convert GraphQL queries to Scala types: very handy, but with the power of Scala why not be more abitious?
So my idea is to try and build a more flexible and fully featured code generation tool for consuming GraphQL APIs. I'm learning a lot as I go along, so decided to write down some of those things. For now though: roughly what do I think I might be able to achieve?
##DSLs
Scala has some features that have allowed people to write some awesome Domain Specific Languages(DSLs). They're basically APIs to libraries that are easy to read and write, and convey meaning really well. Some great examples are [ScalaTest matchers](http://www.scalatest.org/user_guide/using_matchers) and [Akka Http routing](https://doc.akka.io/docs/akka-http/current/routing-dsl/overview.html). My idea is to write an sbt plugin that generates a DSL from a GraphQL schema. It would be really cool if for GraphQL API schemas like
{% highlight graphql %}
Character {
name: String,
age: Integer
}
{% endhighlight %}
we could write Scala code like
{% highlight scala %}
query = query {
  character {
    name
  }
}
val response = client.send(query)
{% endhighlight %}
ie. write Scala code which looks just like a GraphQL query.
On top of the type safety provided by what queries we're allowed to make, it would also be really cool to then type the response accordingly (wrapped in a [Future](https://docs.scala-lang.org/overviews/core/futures.html), naturally) so that
`response.map(_.name)` compiles but `response.map(_.age)` doesn't.
I'd also like the generated code to take care of serialisation, deserialisation and making the requests.

I've started by writing the function to parse GraphQL Schema Definition Language (SDL) files into an internal model. This was an easy place to start, especially since (in the spirit of getting a minimal viable end to end product working) I've just written unit tests for a basic parsing schema and made them parse. For the parsing I've just used Scala's regexp pattern matching, and a little bit of recursion.

When I've finished writing a basic DSL generator I'll write down some of what I've learnt from that, which so far is loads!
