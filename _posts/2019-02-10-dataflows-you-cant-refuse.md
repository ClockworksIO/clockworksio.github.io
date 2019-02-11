---
title: Dataflows you can't refuse 
tags: 3DF Clojure
language: EN
author: malte
---

In this article, we'll explore what bringing a declarative, pull-based web after tomorrow to life actually means.

<!--abstract-->

*This article was originally published on [Malte's personal blog](https://www.maltesandstede.com/clojure/3df/2019/02/10/dataflows-you-cant-refuse.html).*

*I guess it's time I write about [Declarative Differential Dataflows (3DF)](https://github.com/comnik/clj-3df), a project we have been working on [academically](https://www.systems.ethz.ch) and [professionally](https://www.clockworks.io) for quite some time. 3DF builds upon [Differential Dataflows](https://github.com/TimelyDataflow/differential-dataflow), which in turn is based on [Timely Dataflows](https://github.com/TimelyDataflow/timely-dataflow), both created by [Frank McSherry](http://www.frankmcsherry.org).*

*Timely takes expressive, distributed stream processing to the next level. Differential takes Timely to the next level by making iterative computation in a distributed dataflow setting possible. 3DF sits on top of all that good stuff and philosophizes about simpler times where client-server communication is declarative and the [web after tomorrow](http://tonsky.me/blog/the-web-after-tomorrow/) is within reach.*

*In this article, we'll explore what bringing a declarative, pull-based web after tomorrow to life actually means. Spoiler alert: It enables you to obtain a realtime, highly performant, and consistent view of a complex system that supports incremental and iterative updates, automatically notifies you of any change, is composable, and connects to multiple data sources.*

*If that sounds pretty cool to you (I think it really does), make sure to check out the linked repositories! Oh, and of course, read on...*

## Intro

...Chicago, the 1920s. Time of prohibition, flourishing black markets, Lucky Luciano, Al Capone, Frank Costello.

Assume you're part of [Chicago Outfit](https://en.wikipedia.org/wiki/Chicago_Outfit), an Italian-American organized crime syndicate based in Chicago. Actually, assume you're Al Capone — dream big and all that stuff. You're expanding your bootlegging business, and you're also using 3DF as a new way to manage earnings and analyze thug loyalty.

You're also a Clojure aficionado, so you're using clj-3DF (currently only available directly from the [clj-3DF github repo](https://github.com/comnik/clj-3df)):

*If you want to skip everything, check out [chicago's github repo](http://github.com/li1/chicago). I won't be angry, just a little disappointed.*

```clojure
;; deps.edn

{:deps
 {comnik/clj-3df {:git/url "https://github.com/comnik/clj-3df"
                  :sha "52753c7d3c05f1144518e4d2c1d835452877ddc8"}}}
```

You've set up a small fact-oriented database (e.g. [Datomic](https://www.datomic.com)) where your dependable spies report essential information about your gang. You register its schema in the management frontend:

```clojure
;; core.clj

(ns chicago.core
  (:require [clj-3df.core :as df :use [exec!]]
            [chicago.diff-formatter :as formatter]))

(def schema
  {:thug/name      {:db/valueType :String}
   :thug/boss      {:db/valueType :Eid}
   :thug/earnings  {:db/valueType :Number}
   :thug/rat?      {:db/valueType :Bool}
   :thug/territory {:db/valueType :Eid}
   :territory/name {:db/valueType :String}})

(def db (df/create-db schema))
```

Every thug has a name, some income, and belongs to a territory (`:thug/territory` points to a territory `:db/id`). A thug may have a boss. If they have none, they directly report to you. A thug might also be a rat (more on rodents later).

Note that each schema attribute could originate from a different source, so we could easily aggregate information from databases, durable event logs, or even files in a single consistent view.

This is really all there is to managing a successful bootlegging business! Easy, right?

We can now connect to 3DF. Clone 3DF (I built against commit [9123455](https://github.com/comnik/declarative-dataflow/tree/91234554cd8097f4c970dc238004626398f2a4c1)) and run the server using `cargo run --bin server`. Then, connect to it and register your schema:

```clojure
(do
  (def conn (df/create-publication
              "ws://127.0.0.1:6262"
              (comp clojure.pprint/pprint formatter/format-diffs)))
  (exec! conn (df/create-db-inputs db)))
```

`create-publication` allows you to register queries and business rules (more on that in a bit). Also, I'm passing a custom middleware `(comp ...)` that uses a small formatter I wrote to display query results (check out the [gist](https://gist.github.com/li1/aa75a18fa6eab6c05fbbc983b8026701)). If you're fine with verbose `println` logging, just replace `(df/create-publication ...)` with `(df/create-debug-publication "ws://127.0.0.1:6262")`. In that case, your REPL results will look slightly different from the ones in this article.

Let's transact some initial data.

```clojure
(def initial-data
  [{:db/id 1 :territory/name "west"}
   {:db/id          3
    :thug/name      "alfredo"
    :thug/earnings  1000
    :thug/territory 1}
   {:db/id          4
    :thug/name      "bernado"
    :thug/boss      3
    :thug/earnings  500
    :thug/territory 1}
   {:db/id          5
    :thug/name      "carlo"
    :thug/boss      4
    :thug/earnings  120
    :thug/territory 1}
   {:db/id          6
    :thug/name      "cristiano"
    :thug/boss      4
    :thug/earnings  100
    :thug/territory 1}

   {:db/id 2 :territory/name "east"}
   {:db/id          7
    :thug/name      "aurora"
    :thug/earnings  900
    :thug/territory 2}
   {:db/id          8
    :thug/name      "berta"
    :thug/boss      7
    :thug/earnings  450
    :thug/territory 2}
   {:db/id          9
    :thug/name      "carla"
    :thug/boss      8
    :thug/earnings  125
    :thug/territory 2}
   {:db/id          10
    :thug/name      "corinna"
    :thug/boss      8
    :thug/earnings  125
    :thug/territory 2}])

(exec! conn (df/transact db initial-data))
```

The syntax should be relatively self-explanatory, it follows [datomic](https://docs.datomic.com/on-prem/transactions.html) convention. Note that while we're transacting this data ourselves here, in a real setting, this would probably happen somewhere else in the system. E.g., some mobster could transact his newest earnings into a Postgres instance that is connected as a source to 3DF. `df/transact` is perfectly fine here, just be aware that we're more flexible than that.

## Territory Overview

Now that we've connected to 3DF and transacted some thug info, we can start analyzing. To get the ball rolling, let's display our current territory earnings (if you can't read datalog yet, now would be a great time to [learn it](http://www.learndatalogtoday.org)):

```clojure
(exec! conn (df/register-query db "chic/territory-earnings"
              '[:find ?territory (sum ?earnings)
                :with ?t
                :where
                [?terr :territory/name ?territory]
                [?t :thug/territory ?terr]
                [?t :thug/earnings ?earnings]]))
```

Note our use of the `:with ?t` clause[^1]. 

[^1]: `:with` clauses prevent identical `[attribute value]` pairs with originally different `:db/id`s (in our case: carla's and corinna's earnings) in aggregates being lost due to set semantics (for more info, see [the datomic docs](https://docs.datomic.com/cloud/query/query-data-reference.html#with)). We're in the process of migrating to bag semantics by default, so this might not be necessary anymore, but we're playing it safe here (mobsters are really risk averse).

Also note we're using namespaced global query names here. This hints at the power of 3DF: If not only you (i.e. Al Capone) is using the management interface, but many other users — perhaps with different permissions — as well, every user could register his own unique queries (or, if allowed, reuse an existing query). 3DF can handle the load.

As soon as you have registered the query, you should see the current earnings grouped by territory in the REPL:

```clojure
["chic/territory-earnings"
 ([("east" 1600) 2 1]
  [("west" 1720) 2 1])]
```

Now we know what our territories are earning. As promised, any query you register will automatically update itself and notify its listener as soon as something changes. No more push-based REST endpoints!

What about the numbers at the end? The first one tells us about the current timestamp, the second whether we're looking at an addition (`1`) or retraction (`-1`). When some thug earnings change later on, we'll get notified with a new result:

```clojure
["chic/territory-earnings"
 ([("east" 1600) 4 -1]
  [("east" 2025) 4 1])]
```

At `t=4`, east territory's old earnings value `1600` is retracted and the new `2025` added. We're looking at diffs here (think event log, blockchain, datomic)!

## Subordinate Earnings

Let's proceed to a more complex query. We would like to show the total earnings of every thug, i.e. their own income plus their subordinates' income, their subordinates' subordinates' income, etc. — semantically, something along the lines of a "department income".

For this, we'll first construct a query that returns the sum of all direct and indirect subordinates' earnings for a thug:

```clojure
(def rules
  '[;; read `->` as "boss of" relation: boss -> subordinate
    ;; let A -> B -> C
    ;; then (subordinate? C B) == true
    ;; and  (subordinate? A B) == false
    [(subordinate? ?sub ?boss)
     [?sub :thug/boss ?boss]]

    [(subordinate? ?sub ?boss)
     [?sub :thug/boss ?middleman]
     (subordinate? ?middleman ?boss)]])

(exec! conn (df/register-query db "chic/sub-earnings"
              '[:find ?b ?boss (sum ?earnings)
                :with ?t
                :where
                [?b :thug/name ?boss]
                [?t :thug/earnings ?earnings]
                (subordinate? ?t ?b)]
              rules))
```

What's new (read cool) about this query is its use of query rules. Rules can be appended to a `register-query` call and can then be used from the associated query. They can be recursive. Multiple rules with the same name are traversed until one of them matches. In our case, we can use this to implement indirect (read transitive) subordinates.

Now that we've registered `chic/sub-earnings`, let's reuse it; we add it to the individual thug's earnings in `chic/thug-total-earnings`:

```clojure
(exec! conn (df/register-query db "chic/thug-total-earnings"
              '[:find ?t ?thug ?earnings
                :where
                [?t :thug/name ?thug]
                [?t :thug/earnings ?thug-earnings]
                [chic/sub-earnings ?t ?thug ?sub-earnings]
                [(add ?thug-earnings ?sub-earnings) ?earnings]]))
```

There we go:
```clojure
["chic/thug-total-earnings"
 ([(3 "alfredo" 1720) 2 1]
  [(4 "bernado" 720) 2 1]
  [(7 "aurora" 1600) 2 1]
  [(8 "berta" 700) 2 1])]
```

Not so fast! This is a trap, not unlike the one Sonny ran into in The Godfather! This query won't work for the lowest level thugs who don't have any subordinates.

But why? Remember that datalog is a declarative logic programming language. It solves constraint problems. If it can't find a solution, the query evaluates to `false`. For lowest level thugs, no `subordinate?` rules match. `chic/sub-earnings` evaluates to `false` (multiple clauses are connected by logical `and`s). `thug-total-earnings`' `chic/sub-earnings` clause evaluates to `false`. Its `add` clause evaluates to `false`. `thug-total-earnings` evaluates to `false`. We don't receive a result. Everything is `false`. Somebody will sleep with the fishes for that.

 The easiest solution would be to provide default values for `?sub-earnings`, but this isn't implemented yet (I promise it's on our roadmap). We'll have to live with a workaround:

```clojure
(exec! conn (df/register-query db "chic/thug-total-earnings"
              '[:find ?t ?thug ?earnings
                :where
                (or (and [?t :thug/name ?thug]
                         [?t :thug/earnings ?earnings]
                         (not [?s :thug/boss ?t]))
                    (and [?t :thug/name ?thug]
                         [?t :thug/earnings ?thug-earnings]
                         [chic/sub-earnings ?t ?thug ?sub-earnings]
                         [(add ?thug-earnings ?sub-earnings) ?earnings]))]))
```

`or` in queries isn't exclusive, so to prevent double results for thugs that have subordinates, we have to make sure they're exclusive with `(not [?s :thug/boss ?t])`. By the way, have you noticed how I've been showing off transforms such as `add` and `subtract`, logical clauses `not` and `or`, and negation? Pretty cool, eh?

With this new query in place, `chic/thug-total-earnings` does what it's supposed to do! We now not only see a consolidated view of the territory controls, but also which mobsters are in control of particularly lucrative subgroups:

```clojure
["chic/thug-total-earnings"
 ([(3 "alfredo" 1720) 2 1]
  [(4 "bernado" 720) 2 1]
  [(5 "carlo" 120) 2 1]
  [(6 "cristiano" 100) 2 1]
  [(7 "aurora" 1600) 2 1]
  [(8 "berta" 700) 2 1]
  [(9 "carla" 125) 2 1]
  [(10 "corinna" 125) 2 1])]
```

*Bonus points:* The attentive reader might have noticed that we maybe could have rewritten this query without sub-earnings at all, using a recursive thug-total-earnings query:

```
The proof is left as an exercise for the reader.
```

Yes, you're right. But I really wanted to show off rules and query composition, and we're gonna reuse our rules later on. Sue me! Seriously, try me, I've bribed every copper in town.

## Business Rules

In this section, I don't want to praise business. Instead, we are going to use logical rules (≠ query rules) to implement complex behavior that might not be easily possible just with datalog queries. Business rules give you more power than pure datalog queries, but you sacrifice declarativeness and a bit of safety.

I promised you that we'd talk about rodents. Now is the time.

It happens to the best of us: Some sleazy snitch earns enough money to feel brave enough to quit the mob but not enough to respect our honorable business practices. So he talks to the cops and next thing you know he's an informant: a rat! Rats do two things: First, we can't count on their earnings anymore. They want to remain somewhat unnoticed, so all their subordinates will probably continue earning money, but one shouldn't rely on the rat's income. Secondly, they are infectious. Direct bosses will realize there's something going on. Some of them might become rats themselves to escape potential prosecution. If you're unlucky, this brings down your whole organization.

Naturally, we want to simulate this.

First, let's get an overview of all thugs that we believe to be rats:

```clojure
(exec! conn (df/register-query db "chic/rats"
                               '[:find ?t ?thug
                                 :where
                                 [?t :thug/rat? true]
                                 [?t :thug/name ?thug]]))
```

Secondly, we need to model thugs abandoning ship (a very rat thing to do). Through some very complicated formula I won't explain here we've arrived at the critical area of temptation for most mobsters: total earnings between 500 and 1000.

Let's formulate a query that returns these thugs: 

```clojure
'[:find ?t ?thug ?earnings
  :where
  [chic/thug-total-earnings ?t ?thug ?total-earnings]
  [?t :thug/earnings ?earnings]
  (not [chic/rats ?t ?thug])
  [(< ?total-earnings 1000)]
  [(> ?total-earnings 500)]]
```

Next, we'll write a function that potentially turns thugs into rats:

```clojure
(defn turn-to-rat [desc prob diffs]
  (doseq [[[rat name earnings] _time diff] diffs]
    (let [is-rat (< (rand) prob)]
      (when (pos? diff)
        (println "is" name "a" desc "?" is-rat)
        (when is-rat
          (exec! conn 
            (df/transact db 
              [[:db/add rat :thug/rat? true]
               [:db/retract rat :thug/earnings earnings]
               [:db/add rat :thug/earnings 1]])))))))
```

As arguments, we pass in a short description for debugging, a probability with which the mobster turns, and the diffs (these are the results we receive from our trigger query).

In case we think the thug is a rat, we transact that information into our database. Also, we set his earnings to 0. For that, we first have to retract the old value, otherwise we'll suddenly be working with alternate realities (does that sound cool? Then watch our [upcoming talk](http://bobkonf.de/2019/goebel-sandstede.html)). Currently, we can't retract an attribute without knowing its current value (again, I promise this is on our TODO list). We also don't handle the `(not is-rat)` case — once a rat, always a rat.

Putting it all together, we arrive at the following business rule:

```clojure
(df/business-rule conn db "chic/might-rat"
                  '[:find ?t ?thug ?earnings
                    :where
                    [chic/thug-total-earnings ?t ?thug ?total-earnings]
                    [?t :thug/earnings ?earnings]
                    (not (chic/rats ?t ?thug))
                    [(< ?total-earnings 1000)]
                    [(> ?total-earnings 500)]]
                  (partial turn-to-rat "rat" 0.5))
```

Now this is pretty cool. As soon as a mobster has a total income of 500 to 1000, she might turn into a rat!

We've taken care of the first part of rat protocol, so there is only one last thing missing: modelling panicking bosses, or, as I like to call them: squeakers. We can reuse `turn-to-rat` here and simply provide a different query:

```clojure
(df/business-rule conn db "chic/might-squeak"
                  '[:find ?b ?boss ?earnings
                    :where
                    [?b :thug/name ?boss]
                    [?b :thug/earnings ?earnings]
                    (not (chic/rats ?b ?boss))
                    [?s :thug/name ?sub]
                    (subordinate? ?s ?b)
                    [chic/rats ?s ?sub]]
                  (partial turn-to-rat "squeaker" 0.8))
```

When running this, some of our beloved bernados and bertas might abandon ship and pull others with them. This can cascade and might bring down our whole beautiful close-to-legal empire.

A run could look like this:
```clojure
["chic/territory-earnings" ([("east" 1600) 2 1] [("west" 1720) 2 1])]
["chic/sub-earnings"
 ([(3 "alfredo" 720) 2 1]
  [(4 "bernado" 220) 2 1]
  [(7 "aurora" 700) 2 1]
  [(8 "berta" 250) 2 1])]
["chic/thug-total-earnings"
 ([(3 "alfredo" 1720) 2 1]
  [(4 "bernado" 720) 2 1]
  [(5 "carlo" 120) 2 1]
  [(6 "cristiano" 100) 2 1]
  [(7 "aurora" 1600) 2 1]
  [(8 "berta" 700) 2 1]
  [(9 "carla" 125) 2 1]
  [(10 "corinna" 125) 2 1])]

["chic/might-rat" ([(4 "bernado" 500) 2 1] [(8 "berta" 450) 2 1])]
is bernado a rat ? true
is berta a rat ? true

["chic/rats" ([(4 "bernado") 9 1])]
["chic/territory-earnings" ([("west" 1221) 9 1] [("west" 1720) 9 -1])]
["chic/sub-earnings" ([(3 "alfredo" 221) 9 1] [(3 "alfredo" 720) 9 -1])]

["chic/might-squeak" ([(3 "alfredo" 1000) 9 1])]
is alfredo a squeaker ? true 
["chic/thug-total-earnings"
 ([(3 "alfredo" 1221) 9 1]
  [(3 "alfredo" 1720) 9 -1]
  [(4 "bernado" 221) 9 1]
  [(4 "bernado" 720) 9 -1])]
["chic/might-rat" ([(4 "bernado" 500) 9 -1])]

["chic/rats" ([(8 "berta") 10 1])]
["chic/territory-earnings" ([("east" 1151) 10 1] [("east" 1600) 10 -1])]
["chic/sub-earnings" ([(7 "aurora" 251) 10 1] [(7 "aurora" 700) 10 -1])]

["chic/might-squeak" ([(7 "aurora" 900) 10 1])]
is aurora a squeaker ? true
["chic/thug-total-earnings"
 ([(7 "aurora" 1151) 10 1]
  [(7 "aurora" 1600) 10 -1]
  [(8 "berta" 251) 10 1]
  [(8 "berta" 700) 10 -1])]
["chic/might-rat" ([(8 "berta" 450) 10 -1])]

["chic/rats" ([(3 "alfredo") 11 1])]
["chic/thug-total-earnings"
 ([(3 "alfredo" 222) 11 1] [(3 "alfredo" 1221) 11 -1])]
["chic/territory-earnings" ([("west" 222) 11 1] [("west" 1221) 11 -1])]
["chic/might-squeak" ([(3 "alfredo" 1000) 11 -1])]

["chic/rats" ([(7 "aurora") 12 1])]
["chic/thug-total-earnings"
 ([(7 "aurora" 252) 12 1] [(7 "aurora" 1151) 12 -1])]
["chic/territory-earnings" ([("east" 252) 12 1] [("east" 1151) 12 -1])]
["chic/might-squeak" ([(7 "aurora" 900) 12 -1])]
```

Try playing around with it. E.g., you could simulate what happens when you tempt corinna:

```clojure
(exec! conn (df/transact db [[:db/retract 10 :thug/earnings 125]
                             [:db/add 10 :thug/earnings 550]]))
```

She's a nasty rat! Or perhaps not! It depends. I see a lot of potential for some fancy Monte Carlo simulations using multi-dimensional time concepts. Something to talk about [in greater detail](http://bobkonf.de/2019/goebel-sandstede.html).

## The End

We just discussed 3DF, schemas, multiple sources, (nested) queries, (recursive) query rules, business rules, probabilities, and also some minor limitations of 3DF. All in all, I think this has been a productive day for the mob. Al Capone would be proud of us. Well, I guess you are Al Capone in this article. So... you should be proud!

**PS:** I'm gonna be frank with you: Al Capone did not use 3DF back in the 1920s. Even if he had, the [Saint Valentine's Day Massacre](https://en.wikipedia.org/wiki/Saint_Valentine%27s_Day_Massacre) probably couldn't have been prevented using differential dataflows. But there's one thing I'm sure of (and Al Capone would have agreed): Running a crime syndicate has never been as fun as it is with 3DF and Clojure.

**PPS:** As mentioned before, you can find [chicago's source code](http://github.com/li1/chicago) on github. If you have questions, suggestions, business proposals, or just want to say hi, feel free to open a github issue or send [me](mailto:malte@clockworks.io) / [us](mailto:contact@clockworks.io) an email.
