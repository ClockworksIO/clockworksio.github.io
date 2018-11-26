---
title: Reactive Datalog with Vega
category: 3DF, ClojureScript, Vega, Datalog
language: EN
author: david
---

In this post we will build a streaming Vega visualization backed by
reactive Datalog queries.

<!--abstract-->

*In this post we will build a streaming Vega visualization backed by
reactive Datalog queries.*

## Reactive Datalog?

Yes exactly! We are working on a system, dubbed **3DF** for
*Declarative Differential Dataflow* [1], that has the ability to
compile Datalog queries into Differential Dataflows.

Differential Dataflow [2] is a data-parallel programming framework
designed to efficiently process large volumes of data and to quickly
respond to arbitrary changes in input collections.

A unique property of this framework is its ability to incrementally
[^1] update the state of various operators (think `join`, `group`,
...).  In other words: A Differential Dataflow is a computation that
reacts to incoming changes in a smart and efficient way and propagates
any new information correctly.

The system consists of a server written in Rust that takes commands via WebSocket and internally constructs the appropriate dataflows. We've build a client that can compile Datalog queries into query plans that the backend understands.

There is quite a lot to it and I'll gladly defer to [2] for a lot of information. 

To clarify, here is a small example: We are interested in all people with a certain name and a certain age. As a Datalog query that may look like this:
```clojure
[:find ?name ?age
 :where
 [?e :name ?name]
 [?e :age ?age]
 [(> ?age 18)]]
```
Now we input some new information (In general that may come from some file, a stream, kafka, datomic, ... you name it): 

```clojure
{:name "Mabel" :age 19}
```

3DF will update us promptly (For how see below):

```clojure
[[["Mabel" 19] 1]]
```

It tells us that there is new information in the result set of our query. The tuple consisting of `"Mabel" 19`is the result of our query and the `1`is the difference. That means this information is added. 

Maybe we realized, that we made a typo and we retract the first information and add a new one: `:name "Mabel :age 21"`. 3DF will promptly tell us 

```clojure
[[["Mabel" 19] -1] [["Mabel" 21] 1]]
```
indicating that now there are new facts in the system. The person with name "Mabel" is not 19 but actually 21. This represents a fact, a statement about our world that is true at a given point in time.

Given such differences, Differential Dataflow will only do work proportional to the change itself.

Having 3DF and subsequentially Differential Dataflow keeping track of our changes we can start looking at Clojurescript and Vega.

## Communication

3DF exposes a WebSocket connection over which we can register new
Datalog expressions, manage input parameters, send data, and receive
updates to our queries, as new data arrives in the system.

Any further communication is build around Clojure's amazing async
library. The client code [3] is available for both Clojure &
ClojureScript. When connecting to the server, we get a single
connection handle:

```clojure
(def conn (create-conn "ws://127.0.0.1:6262"))
```

It contains one async channel were all outputs are put and one sink to
which we can write commands.  As we likely will be running more than
one query, we want dedicated subscribers for every one of them. For
that purpose we use pub/sub from the async library.  We partition our
results by the name of the query.

A registration will look like the following, with the former query:

```clojure
(exec! conn (register-query "people-over-18" query))
```

Here `people-over-18` is our globally named relation and `exec!` is a
macro that will serialize the output from `register-query` and write
it into the WebSocket.

## Vega

Alright. Communication is sorted and we have 3DF running. Now a short
look at Vega.

Vega is high-level grammar for visualizations [4]. That means we
describe, in a declarative manner, what we want Vega to
visualize. Here is an example specification[^2] for a bar chart with x
values taken from the `hour` field and y values from `sum_passenger`:

```clojure
{:data     {:name "passenger"}
 :width    1000
 :heigt    1500
 :mark     "bar"
 :encoding {:x {:field "hour"
                :type  "quantitative"
                :title "Hour"}
            :y {:field "sum_passenger"
                :type  "quantitative"
                :title "# passengers"}}}
```

In contrast to normal Vega specs, we do not specify any data. We will
stream it!

### Streaming

Vega offers an `embed` functionality, which can run arbitrary
JavaScript in its callbacks. Well thats exactly the place we will put
our communication logic to get new results.

```clojure
(go-loop [] 
  (when-let [[_ results] (<! source)]
    (.. res -view
      (change data-name
              (.. js/vega changeset
              (insert (vega-insert encoding (:insert diffmap)))
                      (remove (vega-remove encoding (:remove diffmap)))))
                      (run))
  (recur)))
```

As soon as some data arrives in the channel, we take it, process it
and call two Vega methods.  We create a `changeset` via the `change`
method, expressing what changes Vega should visualize. The `insert`
takes a vector of tuples indicating new data to insert and the
`remove` takes a function, which will be called on the existing data
tuples and returns `true` for all that are supposed to be removed.

In the end we call `run` which will execute and render the
changes. More on that can be read here [5].

Now we need to simply call the `embed` method and mount it into the
DOM.

## Running some analytics

Our data set is the publicly available data of NYC cab rides [6].
The dataset contains one single day, these are around 9.000.000 lines of uncompressed csv with a size of 800 MB. We are running 3DF in one thread on my MBP.

Every line describes one cab ride with the following attributes: `VendorID`, `pickup_time`, `passenger_count`, `trip_distance`. The remaining ones are omitted, as we are not using them.
We will simulate streaming this data at 1024 lines per batch.
```clojure
(exec! conn
    (register-source
     [:cab/vendor :cab/hour :cab/passenger :cab/distance]
     {:CsvFile {:path      "DATA_PATH"
                :separator ","
                :schema    [[0 {:Number 0}][1 {:Number 0}] [3 {:Number 0}] [4 {:Number 0}]]}}))
```
Columns from this dataset will always have the namespace `cab/` s.t. we can distinguish them from different sources.
The reading of the whole dataset takes, at a batch size of 1024 lines, around 70 seconds. That is quite long, compared to 25 when we do not batch them with that high granularity. That is because we increment the logical timestamp every 1024 lines, which leads Differential to do a lot more progress tracking work.

The first thing we look at is quite simple: *Number of transported people per hour*

Here is the Datalog query
```clojure
(def passenger
  "Number of transported people per hour"
  '[:find ?hour (sum ?passenger)
    :with ?e
    :where
    [?e :cab/passenger ?passenger]
    [?e :cab/hour ?hour]])
```

The Vega specs for all queries are here: [vega-specs](https://gist.github.com/eoxxs/cca4b9d83b1eece79e4ce4ea22b16509)

You'll only see parts of the whole day. 3DF will update us with a
quite high frequency. Every update consists of values to be removed
and values to be added.

![passengers by hour](/assets/blog/reactive-datalog-with-vega/passengers.gif)

On the x-axis you see the different hours and the y-axis represent the number of transported passengers. You can see how neatly those bars are growing.

## Interactive queries
Well that has been nice and fun. We can stream results of our computations into Vega and get updated visualizations whenever something changes in our queries result set. But this is just the beginning. How about creating some interactive visualization just with 3DF.

Alright, so in order to control anything about an already existing dataflow (our query) we need to use input collections. Here is the query we are looking into now

```clojure
(def distance-distribution
  "Aggregates the number of rides for a given distance"
  '[:find ?distance (count ?e)
    :where
    (analysis/hour ?hour)
    [?e :cab/distance ?distance]
    [?e :cab/hour ?hour]])
```

It is a query that asks for every distance and the number of times a
ride took that distance. I cast the distance float into integers. The
difference to the former query is the first line in the `:where`
clause. We bind the symbol `?hour` to the result of `analysis/hour`
and then join it with the `:cab/hour`. `analysis/hour` is a named
relation, nothing else then an already registered query.

Cool! So we can feed output from existing queries into others, simply
by mentioning them. Let's look at how to create this input collection.
This query simple produces the `?hour` value of all `:filter/hour`
attributes.

```clojure
(exec! conn 
  (create-input 
    {:filter/hour {:db/valueType :Number}}))

(exec! conn 
  (register-query
    schema "analysis/hour"
    '[:find ?hour
      :where 
      [?e :filter/hour ?hour]]))
```

We create a new schema that just consists of the former attribute,
create a new input collection and then we register the query. Now we
can start transacting into that input handle and observe changes to
our visualization.

In Emacs on the right, I transact different hour values and then
retract them in the same transaction.

![interactive queries](/assets/blog/reactive-datalog-with-vega/interactive_2.gif)

I start with the distribution for hour 12, then I move to hour 18 and
then to hour 3. You could also add several ones to see the distance
distribution for combined hours. Every addition into the collection of
the attribute `:filter/hour` leads the `?hour`-join in the
`distance-distribution` query to match all those rows that have the
respective `:cab/hour` value. Retractions work the same but with
negative differences and as such lead to data being removed.

On this level of abstraction it looks like we're polling the database
with different hour filters. But we are not executing the query from
scatch, we simply let 3DF propagate this new information through the
system, re-using previously computed results wherever it can.

## Embrace the change 

Imaging you just did these analytics and your boss stormes into your
office.  He tells you that one of these vendors has retracted their
consent for data usage.  Oh damn.  We need to remove all of it
asap. Damn you GDPR compliance. We just spend all day building this
wonderful dashboard.

3DF to the rescue. We have a system that can propagate changes. So
lets see how we can solve this.

We'll assume that we have a separate database where we store
information about the different vendors, sort of a user
database. Every line in our analytics files contains as first column
an id, uniquely identifying these vendors (in this case there are just
two of them).

In this case the vendor is referenced by its id in our cab dataset. In
our query we would match these id references, to incorporate
information from that vendor database.[^3] Imagine our query would
have been this:

```clojure
(def passenger
  "Number of transported people per hour"
  '[:find ?hour (sum ?passenger)
    :with ?e
    :where
    [?e :cab/passenger ?passenger]
    [?e :cab/hour ?hour]
    [?e :cab/vendor ?id]
    [_  :vendor/id ?id]])
```

Now observe what happens to our analytics computations, at the end of
day presumably after we processed all data, when we remove this one
vendor from the vendor database.

```clojure
(exec! conn (transact db [[:db/retract :vendor/id 1]]))
```

Number of passengers|  Distribution of distances
:-------------------------:|:-------------------------:
![number of passengers](/assets/blog/reactive-datalog-with-vega/gdpr_1.gif) | ![distribution of distances](/assets/blog/reactive-datalog-with-vega/gdpr_2.gif)

You see how all the values drop down as we remove the vendor. This one
retraction led to a change touching more than half of all cab rides
and these changes are propagated through all dataflows.

All computations got updated without us doing anything else, we did
not change the underlying query, filtered the input dataset or
explicitly rerun any computation at all. 3DF efficiently computed the
differences and told us about the changes.

This is amazing.

Imagine we had a lot more computations, feeding the results from the
former queries into some price forecasting, movement analytics and so
on. Every downstream computation, somehow using information that has
changed through this retraction, would be updated accordingly and as
efficiently as possible.

Well this was a lot of fun. I hope you made it through this lengthy
piece. Next time I will run this in a cluster configuration and we
will see how 3DF performs when we scale out.  Until then cheers and
goodbye.

## Resources

- [1] [3DF](https://github.com/comnik/declarative-dataflow)
- [2] [Differential Dataflow](https://github.com/frankmcsherry/differential-dataflow)
- [3] [clj-3DF](https://github.com/comnik/clj-3df)
- [4] [Vega](https://vega.github.io/vega/)
- [5] [Vega-Streaming](https://vega.github.io/vega-lite/tutorials/streaming.html)
- [6] [NYC Cab Rides](http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml)

[^1]:Actually, even more than that Differential Dataflow keeps the differences partially ordered and as a full trace, instead of totally ordered and compacted as most incremental computation frameworks do, but that does not need to concern us here.
[^2]:This is a Vega-lite spec, but all the things I later show are Vega. Vega specs are just quite verbose.
[^3]:We are actually not using the vendor information anywhere else, but let's assume it for now.
