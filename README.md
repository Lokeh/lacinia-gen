# lacinia-gen

`lacinia-gen` lets you generate GraphQL responses using your [lacinia](https://github.com/walmartlabs/lacinia) schema
and GraphQL queries, allowing you to make your tests more rigorous in both Clojure and Clojurescript.

[![Clojars Project](https://img.shields.io/clojars/v/lacinia-gen.svg)](https://clojars.org/lacinia-gen)

## Usage

There are two main ways to use `lacinia-gen`:
 - [Full graph](#full-graph) generates the entire graph from any entry point
 - [Query result](#query-result) generates a response for a particular query

### Full graph

You can create a generator for a full graph from any root by using the `generator` function in
`lacinia-gen.core`. This resolves all reachable nodes in the graph up to the desired recursion depth.

```clojure
(require '[lacinia-gen.core :as lgen])
(require '[clojure.test.check.generators :as g])

(let [schema {:enums {:position {:values [:goalkeeper :defence :attack]}}
              :objects {:team {:fields {:wins {:type 'Int}
                                        :losses {:type 'Int}
                                        :players {:type '(list :player)}}}
                        :player {:fields {:name {:type 'String}
                                          :age {:type 'Int}
                                          :position {:type :position}}}}}]

  (let [gen (lgen/generator schema)]

    (g/sample (gen :position) 5)
    ;; => (:defence :goalkeeper :defence :goalkeeper :goalkeeper)

    (g/sample (gen :player) 5)
    ;; => ({:name "", :age 0, :position :attack}
           {:name "", :age 0, :position :attack}
           {:name "", :age 1, :position :attack}
           {:name "m", :age -2, :position :defence}
           {:name "¤", :age 4, :position :defence})

    (g/sample (gen :team) 5)
    ;; => ({:wins 0, :losses 0, :players ()}
           {:wins 1,
            :losses 1,
            :players ({:name "", :age -1, :position :defence})}
           {:wins 1,
            :losses 2,
            :players
            ({:name "", :age 1, :position :attack}
             {:name "q", :age -2, :position :attack})}
           {:wins -3, :losses 1, :players ()}
           {:wins 0,
            :losses -3,
            :players
            ({:name "ßéÅ", :age 3, :position :attack}
             {:name "", :age -4, :position :defence})})))
```

It supports recursive graphs, so the following works too:

```clojure
(let [schema {:objects {:team {:fields {:name {:type 'String}
                                        :players {:type '(list :player)}}}
                        :player {:fields {:name {:type 'String}
                                          :team {:type :team}}}}}]

    (let [gen (lgen/generator schema)]
      (g/sample (gen :team) 5)))

;; => {:name "ö",
       :players
       ({:name "³2",
         :team
         {:name "ºN",
          :players
          ({:name "Ïâ¦", :team {:name "¤¼J"}}
           {:name "o", :team {:name "æ8"}}
           {:name "/ç", :team {:name "ãL"}}
           {:name "éíª6", :team {:name "v"}})}}
```

If you want to limit the depth to which certain objects recurse, you can do so with the following option:

```clojure
(lgen/generator schema {:depth {:team 0}})
```

This will ensure that `:team` only appears once in the graph; the team will have players, but it prevents the players from having a team.
The default value for all objects is 1, meaning each will recur once (a team will have players which have a team which has players, but no further).
You can set any integer value you wish.

If you want to limit the number of items in lists, you can do so with the following option:

```clojure
(lgen/generator schema {:width {:player 2}})
```

This will ensure that lists of `:player` will have a maximum size of 2.

### Query result

If you have a GraphQL query and wish to generate data for its result, you can use `lacinia-gen.query`.

```clojure
(require '[lacinia-gen.query :as query])

(def schema '{:enums {:position {:values [:goalkeeper :defence :attack]}}
              :objects {:team {:fields {:wins {:type Int}
                                        :losses {:type Int}
                                        :players {:type (list :player)}}}
                        :player {:fields {:name {:type String}
                                          :age {:type Int}
                                          :position {:type :position}}}}
              :queries {:teams {:type (list :team)
                                :resolve :resolve-teams}}})

(let [f (query/generate-fn schema {})]
    (f "query { teams { wins players { name } } }" {}))

;; => {:data
       {:teams
        ({:wins 0, :players ()}
         {:wins 0, :players ({:name ""})}
         {:wins 1, :players ()}
         {:wins 0, :players ({:name "÷ "} {:name "¢"})}
         {:wins 1, :players ()})}}
```

Currently the queries are interpreted by Lacinia and as such require the JVM. This means
`generate-fn` cannot be used from Clojurescript. The two macros `generate-query` and
`generate-query*` may be used in Clojurescript and will evaluate to the generated result
of the query.

```clojure
(ns my.test
  (:require-macros [lacinia-gen.query :refer [generate-data*]]))

(def schema '{:objects {:team {:fields {:wins {:type Int}
                                        :losses {:type Int}}}}
              :queries {:teams {:type (list :team)
                                :resolve :resolve-teams}}})

(def query "query { teams { wins } }")

(def data (generate-data* schema query {} {}))

data
;; => {:data
       {:teams
        ({:wins 0}
         {:wins 0}
         {:wins 0)}
         {:wins -3}
         {:wins 0})}
```

### Custom scalars

If your schema contains [custom scalars](http://lacinia.readthedocs.io/en/latest/custom-scalars.html) you will need to
provide generators for them. You can do so in the following way:

```clojure
(let [schema {:scalars {:Custom {:parse :parse-custom
                                 :serialize :serialize-custom}}
              :objects {:obj-with-custom
                        {:fields
                         {:custom {:type :Custom}
                          :custom-list {:type '(list :Custom)}}}}}]

  (let [gen (lgen/generator schema {:scalars {:Custom gen/int}})]
    (g/sample (gen :obj-with-custom) 10)))

;; => {:custom 23
       :custom-list (-1 4 16)}
```

## Development

[![CircleCI](https://circleci.com/gh/oliyh/lacinia-gen.svg?style=svg)](https://circleci.com/gh/oliyh/lacinia-gen)

## License

Copyright © 2018 oliyh

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
