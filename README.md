# joinery

A library to enable traversal of graph-like data sources using Clojure(Script)
map protocols.

## Quickstart

`joinery` ships with built-in support for table-like databases, i.e. `{table {id entity}}`.

```clojure
(require '[cjsauer.joinery :refer [joined-map]])

;; Assume you have a local normalized data source.
;; joinery includes support for "table-like" sources out of the box:
(def db
  {:person/id {1 {:person/name "Calvin"
                  :person/friends [[:person/id 2]]
                  :person/pet [:pet/id 1]}
               2 {:person/name "Derek"
                  :person/friends [[:person/id 1]]}}
   :pet/id    {1 {:pet/name "Malcolm"
                  :pet/species :dog
                  :pet/owner [:person/id 1]}}})

;; Create a joined-map interface to the db
(def jm (joined-map db))

;; Use it like a normal map, and observe "joins" resolved on-demand
(get-in jm [:person/id 1 :person/pet])
;;=> #:pet{:name "Malcolm", :owner [:person/id 1], :species :dog}

;; Cardinality-many joins are resolved recursively
(get-in jm [:person/id 1 :person/friends])
;; => [#:person{:name "Derek", :friends [[:person/id 1]]}]

;; Cycles are possible
(get-in jm [:person/id 1 :person/pet :pet/owner])
;; => #:person{:name "Calvin", :friends [[:person/id 2]], :pet [:pet/id 1]}

;; Most of the expected map functions are implemented
;;   assoc, dissoc, find, seq, reduce, etc...
(def jm' (assoc-in jm [:person/id 1 :person/best-friend] [:pet/id 1]))
(get-in jm' [:person/id 1])
;; => #:person{:name "Calvin", :best-friend [:pet/id 1], ...}

;; Let's follow the newly added edge
(get-in jm' [:person/id 1 :person/best-friend])
;; => #:pet{:name "Malcolm", :owner [:person/id 1], :species :dog}
```

## Advanced usage

The `joined-map` constructor accepts a couple more helpful options that provide
additional means of customization. Firstly, by default, `joined-map` will use
the provided `db` as the starting point of traversal. One can optionally provide
a "starting entity" that will become the "current" value of the joined map:

```clojure
(def db
  {:person/id {1 {:person/name "Calvin"
                  :person/friends [[:person/id 2]]
                  :person/pet [:pet/id 1]}
               2 {:person/name "Derek"
                  :person/friends [[:person/id 1]]}}
   :pet/id    {1 {:pet/name "Malcolm"
                  :pet/species :dog
                  :pet/owner [:person/id 1]}}})

;; Use the db as the backing source, but start at a different entity
(def jm (joined-map db {:ui.selected/user [:person/id 1]}))
jm
;; => #:ui.selected{:user [:person/id 1]}

;; Joins are resolved just as before
(:ui.selected/user jm)
;; => #:person{:friends [[:person/id 2]], :name "Calvin", :pet [:pet/id 1]}
```

In addition, one can customize the functionality of joins by implementing the
`Joinery` protocol. Here is the default table-join implementation that ships
with `joinery`:

```clojure
(deftype TableIdentJoinery []
  Joinery
  (is-join? [_ v] (and (vector? v)
                    (= 2 (count v))
                    (keyword? (first v))))
  (join [_ table v] (get-in table v)))

;; Create a joined-map using a specific Joinery implementation:
(joined-map db entity (TableIdentJoinery.))
```

We can see above that the protocol is quite simple. We need to provide two things:
how to identify what values should be treated as "links", and, given we've reached
one of these links, how do we "resolve" it. With these two functions, we can obtain
a map-like interface to a myriad of different normalized sources.

## Development

Run the project's tests:

    $ clojure -T:build test

Run the project's CI pipeline and build a JAR:

    $ clojure -T:build ci

This will produce an updated `pom.xml` file with synchronized dependencies
inside the `META-INF` directory inside `target/classes` and the JAR in `target`.
You can update the version (and SCM tag) information in generated `pom.xml` by
updating `build.clj`.

Install it locally (requires the `ci` task be run first):

    $ clojure -T:build install

Deploy it to Clojars -- needs `CLOJARS_USERNAME` and `CLOJARS_PASSWORD` environment
variables (requires the `ci` task be run first):

    $ clojure -T:build deploy

## License

Copyright © 2021 Calvin Sauer

_EPLv1.0 is just the default for projects generated by `clj-new`: you are not_
_required to open source this project, nor are you required to use EPLv1.0!_
_Feel free to remove or change the `LICENSE` file and remove or update this_
_section of the `README.md` file!_

Distributed under the Eclipse Public License version 1.0.
