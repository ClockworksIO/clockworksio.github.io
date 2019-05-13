---
title: Differential GraphQL
tags: 3DF GraphQL
language: EN
author:
  - niko
  - malte
---

@TODO

<!--abstract-->

We've [previously]({% post_url 2018-09-13-incremental-datalog %})
looked at [3DF, a streaming Datalog
engine](https://github.com/comnik/declarative-dataflow) built on
[Differential
Dataflow](https://github.com/TimelyDataflow/differential-dataflow). Instead
of polling a database, 3DF clients can simply subscribe to Datalog
queries and will receive updates whenever their result set
changes. Thanks to Differential Dataflow, queries — even nested or
recursive ones — are efficently maintained, and don't have to be
recomputed from scratch on every new input. This makes it pretty easy
to obtain near real-time, consistent views on highly dynamic,
graph-structured data.

[GraphQL](https://graphql.org/learn/) is another query language for
simple access to relational data. While somewhat less powerful,
GraphQL hits a sweet spot somewhere within the intersection of
*useful*, *en vogue*, and *enough funding for a logo and a
website*. GraphQL was designed to communicate data interests between
client and server — a use case that could benefit from a reactive
engine.

## GraphQL Recap

An archetypical GraphQL expression is

``` graphql
{hero {name age}}
```

which might elicit a response such as

``` json
{
  "hero": [
    {
      "name": "Ser Brienne of Tarth",
      "age": 32
    },
    {
      "name": "Tormund Giantsbane"
    }
  ]
}
```

Note how `{name age}` are understood to form a disjunction (we do not
know Tormund's age, still he is part of the result set), whereas in
Datalog we mostly deal in conjunctions. The whole cabal dangles off of
a relation called `hero`, serving as the starting point of our graph
exploration. This most basic aspect of GraphQL semantics is readily
formalized: *for a set of initial entities, grab a bunch of attributes
and put them into maps*.

From that we can start to navigate along relations, asking for example
about the foes that each hero has `bested` in battle.

``` graphql
{
  hero {
    name
    age
    # Navigating relations.
    bested { name }
  }
}
```

``` json
{
  "hero": [
    {
      "name": "Ser Brienne of Tarth",
      "age": 32,
      "bested": [
        {"name": "The Hound"},
      ]
    },
    {
      "name": "Tormund Giantsbane",
      "bested": [
        {"name": "The Giant(?)"},
      ]
    }
  ]
}
```

While disjunctions are frankly rather boring in isolation, querying
relations in GraphQL implies a conjunctive term: we of course do not
want to see any being that has ever been bested, but rather only those
specifically bested by either Brienne or Tormund.

GraphQL has several other noteworthy capabilities, we will however
only consider on more extension for now: the use of
[arguments](https://graphql.org/learn/queries/#arguments) to perform
selections. For example, the following query would only produce
results for Brienne:

``` graphql
{
  # Selection on parents.
  hero(name: "Ser Brienne of Tarth") {
    age
    bested { name }
  }
}
```

It is less clear what we expect to happen for selections on nested
relations.

``` graphql
{
  hero {
    name
    age
    # Selection on children.
    bested(name: "The Hound") { bestedAt }
  }
}
```

The above query might produce results for *all* heroes, some (most?)
of which might not have bested The Hound. In this reading, a selective
argument would only constrain the pointy end of a relation without
affecting higher level entities. Alternatively, we could interpret
this query to ask for *only* heroes that have bested The Hound. For
now, we will stick with the former interpretation (we have Datalog for
the latter, after all). Later on we will see how to implement the
alternative semantics as well.

## Mapping GraphQL onto 3DF

Our strategy for implementing the above subset of GraphQL in 3DF is
rather straightforward. Differential Dataflow allows us to think in
relational terms — without even worrying much about streaming
execution.

We will treat each nesting level independently, as illustrated
below. For each level, we must determine the set of relevant entity
ids according to the semantics we've identified in the previous
section. We then join the relevant entity ids against each requested
attribute individually, concatenatenating all of the result sets as we
go.

![GraphQL Implementation Strategy](/assets/blog/differential-graphql/strategy.png)

In order to determine relevant entities, we look at the path leading
to a nesting level and turn it into a set of clauses that 3DF's query
engine can understand. This is illustrated for the path `[hero,
bested]` below.

![Path to Clauses](/assets/blog/differential-graphql/path_to_clause.png)

In order to identify which values came from which attribute in the
resulting pile of tuples, we will also interleave tuples with path
information, e.g. transforming output tuples of the form `[?root ?hero
?foe ?name]` into `[?root :hero ?hero :bested ?foe :name ?name]`. 

This process is easily extended to cover selective arguments (which
merely pose additional constraints on the set of relevant entities),
as shown below.

![Argument to Clauses](/assets/blog/differential-graphql/argument_to_clause.png)

Here we can now decide between the two semantics for nested arguments,
as touched on in the beginning. If we wanted to constrain parents by
their children's arguments, we would construct the set of clauses for
the deepest nesting level and use that on all parent levels as
well. However at that point, why not give [Datalog]({% post_url
2018-09-13-incremental-datalog %}) a try.
