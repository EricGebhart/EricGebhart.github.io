---
layout: post
title: "Using Pail with a Graph Schema."
description: "Using Graph Schema with Pail and Cascalog."
date: 2014-01-23 22:34:52
category: clojure
tags: [Graph Schema, Pail, Cascalog, Clojure, Big data]
---

In a [previous post](href="http://ericgebhart.com/thrift-pail-cascalog-and-clojure/)
I wrote about using Thrift, Pail and Cascalog. In this post I'll expand
on that, or rather simplify that example with an extension library,
Pail-Graph. More specifically this is about using Pail with a Graph
Schema, which is a little bit more specialized than just using thrift and
Pail. The use of Graph Schema and the likeness of the example Graph Schema
comes from [Nathan Marz' Book on Big Data](http://manning.com/marz/). Like
everything else I've done lately this is all based on David
Cuddeback's clj-pail, clj-thrift, pail-thrift and pail-cascalog
[libraries](https://github.com/dcuddeback/clj-pail). Pail-graph
wraps and extends each of the libraries. [The Pail-Graph
library](http://GitHub.com/EricGebhart/Pail-graph) mostly
simplifies using a graph-schema with pail and Cascalog. Pail-Graph
is available on [Clojars](https://clojars.org/pail-graph). All
of the code in this post is available in the [pail-graph
example](https://github.com/EricGebhart/pail-graph/blob/master/src/example/example.clj)
in the library on github. Just like the last time, 
[clone my repository](https://github.com/EricGebhart/pail-graph) fire up a REPL and
follow along!

## Graph Schema

This is the easy part. I'm using the same graph schema
before, and one that is somewhat similar to what Nathan
Marz describes in his [Big Data book](http://manning.com/marz/). 
You may also want to read this 
[post](http://nathanmarz.com/blog/thrift-graphs-strong-flexible-schemas-on-hadoop.html)
about Graph Schema and thrift. The gist
of this schema is that there is a single Data Unit that is the entire
database. The Data Unit is a Union of possible values, one of which is a
PersonProperty. A PersonProperty is a structure that contains a person Id
and a property. The property is a Union of values which could be simple
values or structures. First name and last name vs Location which is a
structure of any or all of the following; Address, city, county, state,
country and zip code.

The graph schema looks like this.

```java
namespace * people

/* a location structure. */
struct Location {
  1: optional string address
  2: optional string city;
  3: optional string county;
  4: optional string state;
  5: optional string country;
  6: optional string zip;
}

/* the basic union of properties */
union PersonPropertyValue {
  1: string first_name;
  2: string last_name;
  4: Location location;
  5: i16 age;
}

/* A struct to hold the id and the property together. */
struct PersonProperty {
  1: required string id;
  2: required PersonPropertyValue property;
}

/*  an Edge. */
struct FriendshipEdge {
  1: required string id1;
  2: required string id2;
}

/* this is a basic node. This is what the database is. Everyone consists of a bunch of these.
 */
union DataUnit {
  1: PersonProperty property;
  2: FriendshipEdge friendshipedge;
}
```

## Creating Thrift objects

This part is just like the other example. We just need to build some DataUnit's with the build function from clj-thrift.

```clojure
(def du1-1 (thrift/build DataUnit {:property {:id "123"
                                              :property {:first_name "Eric"}}}))

(def du1-2 (thrift/build DataUnit {:property {:id "123"
                                              :property {:last_name "Gebhart"}}}))

(def  du1-3 (thrift/build DataUnit {:property {:id "123"
                                               :property { :location {:address "1 Pack Place"
                                                                      :city "Asheville"
                                                                      :state "NC"}}}}))

(def du2-1 (thrift/build DataUnit {:property {:id "abc"
                                              :property {:first_name "Frederick"}}}))

(def du2-2 (thrift/build DataUnit {:property {:id "abc"
                                              :property {:last_name "Gebhart"}}}))

(def  du2-3 (thrift/build DataUnit {:property {:id "abc"
                                               :property { :location {:address "1 Wall Street"
                                                                      :city "Asheville"
                                                                      :state "NC"}}}}))

(def du3 (thrift/build DataUnit {:friendshipedge {:id1 "123" :id2 "abc"}}))

(def objectlist [du1-1 du1-2 du1-3 du2-1 du2-2 du2-3 du3])
```

Now we have a list of thrift objects. Opening a pail and writing them is easy.

```clojure
(def mypail (pail/find-or-create ( DataUnitPailStructure.) "example_output"))
(pail/write-objects mypail objectlist)
```

Here's what Pail looks like.

<pre>
└─(17:45:%)── tree example_output                                       ──#(Tue,Jan14)─┘
example_output
├── friendshipedge
│   ├── 636155fb-7126-4d78-b977-cc90daee62ed.pailfile
│   └── 8dadaae2-8602-499f-a6f4-339b909712a0.pailfile
├── pail.meta
└── property
    ├── first_name
    │   ├── 636155fb-7126-4d78-b977-cc90daee62ed.pailfile
    │   └── 8dadaae2-8602-499f-a6f4-339b909712a0.pailfile
    ├── last_name
    │   ├── 636155fb-7126-4d78-b977-cc90daee62ed.pailfile
    │   └── 8dadaae2-8602-499f-a6f4-339b909712a0.pailfile
    └── location
        ├── 636155fb-7126-4d78-b977-cc90daee62ed.pailfile
        └── 8dadaae2-8602-499f-a6f4-339b909712a0.pailfile
<pre/>

That's about it for getting data into a pail. What I skipped was setting
up a PailStructure which defines partitioning. In Pail-Graph, unlike
Pail-Thrift, there is the additional work of
defining a Tap Mapper. This is where I make you read my
previous posts if you haven't already. I don't want to
explain Pail Structures and Partitioning again. First is my '<a
[Thrif-pail-cascalog and clojure post](http://ericgebhart.com/thrift-pail-cascalog-and-clojure/)
. Second is my
[Fressian, Pail and Cascalog post](http://ericgebhart.com/fressian-pail-and-cascalog/)
You'll understand and appreciate all of this that much more
if you read them.

Pail-Graph adds a TapMapper setting to the Pail-Structure that was defined
in clj-pail. A tap mapper takes a list of property-paths and processes
them in a way that is corollated to the behavior of the partitioner. The
key is in the property-paths. A property path is a vector of field id's
and names leading to a final property within the Thrift object. In this
case it is DataUnit. Here is how we get the property-paths for a DataUnit.

```clojure
(type/property-paths DataUnit)
=>[({:id 1, :name "property"} {:id 1, :name "id"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 1, :name "first_name"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 2, :name "last_name"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 1, :name "address"})   
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 2, :name "city"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 3, :name "county"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 4, :name "state"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 5, :name "country"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 4, :name "location"} {:id 6, :name "zip"}) 
   ({:id 1, :name "property"} {:id 2, :name "property"} {:id 5, :name "age"}) 
   ({:id 2, :name "friendshipedge"} {:id 1, :name "id1"}) 
   ({:id 2, :name "friendshipedge"} {:id 2, :name "id2"}) 
```

The Tapmapper function only needs to recieve one of these entries and
return a path to a given property as the partitioner would have defined
it. We can get
a list of the taps from a pail connection or PailStructure once the
tapmapper function is defined and assigned to the PailStructure. The
tap map for DataUnit,
with the current pail structure which created the Pail above, looks likes this.

```clojure
(pail/tap-map (DataUnitPailStructure.))
=> {:property ["property"], 
    :first_name ["property" "first_name"], 
    :last_name ["property" "last_name"],
    :location ["property" "location"],
    :age ["property" "age"],
    :friendshipedge ["friendshipedge"]}
```

There are tap mapper functions defined for all 4 partitioners supplied
in the Pail-Thrift and Pail-Graph libraries. There is an additional null
tapmapper which is the default value for
any PailStructure. The tap mapper for the Union-name-property partitioner
looks like this. The most complicated part is creating a reasonable
key name for a given property path. The second function is the one
used in the PailStructure definition and which returns the mapper to
the PailStructure.

```clojure
(defn union-name-property-taps [path]
  "fields ending in [Pp]roperty are partitioned further. ie. :first_name ['property' 'first_name']
   for partitioners where the field name is the directory name."
  (let [propregex #"^.*[Pp]roperty$"
        res (:name (first path))
        subunion (if (and (re-find propregex res) (> (count path) 2)) (nth path 2) nil)
        name (let [prefix (clojure.string/replace res propregex "-")]
               (if subunion
                 (if (= prefix "-")
                   (:name subunion)
                   (clojure.string/join prefix (:name subunion)))
                 res))]
    (conj [(keyword name)]  (vec (if subunion (conj [res] (:name subunion)) [res])))))

(defn union-name-property-tap-mapper
  "returns a union name property tap mapper"
  []
  union-name-property-taps)
```


Now that all of that is done, getting Cascalog taps is easy. We just ask for the property we want with get-tap.

```clojure
(pail/get-tap mypail :first_name)
=> #<PailTap PailTap["PailScheme[['pail_root', 'first_name']->[ALL]]"]["example_output/property/first_name"]>
```

Get-tap takes all the work out of getting taps from a partitioned Pail. We no longer need to remember what our partitioner is doing every time we want to create a tap.
If there is ever any question about the taps available for a Pail we can list them with list-taps.

```clojure
(pail/list-taps mypail)
=> (:property :first_name :last_name :location :age :friendshipedge)
```

Next to getting taps another somewhat painful part of using thrift
with Cascalog is getting the values back out of the taps. For simple
properties like first name, last name and age this is no problem. But
when it comes to more complex properties which are structures of optional
values this is more of a problem. Location is an example of this type
of structure. In my previous post the solution was this function.

```clojure
;location property deconstruction.
(defmapfn locprop
  "Deconstruct a property object, which has an id and a location struct
   Top Data Union -> Property-struct {:id :property} -> Property-union -> location structure"
  [du]
  (into [(thrift/property-value du :id)]
        (map #(thrift/value (thrift/property-union-value du :property) %)
             [:address :city :county :state :country :zip])))
```

This is fine, but it is not generic which means for each structure
like Location, there needs to be another Cascalog operator to pull
it apart. Pail-Graph provides a solution to this in it's field-keys
function. Field-keys gives a list ordered by field id, of any structure
or union. Using field-keys allows the creation of a more generic Cascalog
operator.

```clojure
(defmapfn structprop
  "Deconstruct a property structure object such as location, which has an id and a structure
   this is only good for one layer below the property union.
   Top Data Union -> Property-struct {:id :property} -> Property-union -> target structure"
  [du]
  (into [(thrift/property-value du :id)]
        (let [th-structure (thrift/property-union-value du :property)]
            (map #(thrift/value th-structure %) (thrift/field-keys th-structure)))))
```

Now we have a Cascalog function that can deconstruct any structure that
might come along in the PersonPropertyValue union. Using all of this
together is very easy and nice
compared to the version in the 
[Thrift, Pail, Cascalog and Clojure post](http://ericgebhart.com/thrift-pail-cascalog-and-clojure/)
. Compared to that example, getting
taps using operators to extract the data values is much easier here
with Pail-Graph.

```clojure
(defn get-everything [pail-connection]
  (let [fntap (pail/get-tap pail-connection :first_name)
        lntap (pail/get-tap pail-connection :last_name)
        loctap (pail/get-tap pail-connection :location)]
    (??<- [?first-name ?last-name !address !city !county !state !country !zip]
          (fntap _ ?fn-data)
          (lntap _ ?ln-data)
          (loctap _ ?loc-data)
          (sprop ?fn-data :> ?id ?first-name)
          (sprop ?ln-data :> ?id ?last-name)
          (structprop ?loc-data :> ?id !address !city !county !state !country !zip))))


(get-everything mypail)
=> (["Eric" "Gebhart" "1 Pack Place" "Asheville" nil "NC" nil nil] 
    ["Frederick" "Gebhart" "1 Wall Street" "Asheville" nil "NC" nil nil])
```

Using Pail with a Graph Schema and Cascalog can be a fairly streamlined
experience with just a little introspection of the datatypes being
written to any given pail. Graph Schema provides some structure where
there could be none, and at the same time provides some infrastructure
to make life easier when it comes to actually working with the data. If
you have any idea's on how to improve this further, please fork me. Leave
any comments below.


