---
layout: post
title: "Fresh data with Prismatic Schema, Fressian, Pail and Cascalog."
description: "Using Prismatic Schema with Fressian, Pail and Cascalog."
Date: 2014-03-06 23:13:53
category: clojure
tags: [Fressian, Pail, Cascalog, Clojure, Big data]
---

This is the fourth post in a series on using Pail and Cascalog in Clojure. 
 
_Since this post was written, Prismatic schema has changed and no longer
supports unions, and is therefore not useful in the way that it was. 

In addition, Clojure now has Clojure.spec which would be
a much nicer way to do this._
 

In the [first post](http://ericgebhart.com/thrift-pail-cascalog-and-clojure/) I
wrote about using Thrift, Pail and Cascalog. My initial goals were
to explore Nathan Mar's Lambda Architecture from clojure. In the
[second post](http://ericgebhart.com/fressian-pail-and-cascalog/) I
used Fressian instead of Thrift to serialize clojure data to and from
a pail. The simplicity of using Fressian with native clojure data types
was really nice.
In the last post,<a [Using pail with a graph schema](http://ericgebhart.com/using-pail-with-a-graph-schema/) 
I expanded on the use of graph schema, Thrift and pail by using a tap
mapping abstraction on top of pail and graph schema which makes it easy
to create taps from vertically partitioned pails.

This post is about doing the same thing but with 
[Prismatic Schema](https://github.com/Prismatic/schema)
instead of Graph Schema, clojure
data types, rather than thrift objects, and fressian as a serializer
instead of thrift. Replacing Graph Schema and Thrift with Prismatic
Schema and clojure data types simplifies everything and gives a
few benefits over using thrift objects. In the process of creating
this example the tap mapping abstraction has found it's way into <a
[clj-pail-tap](https://github.com/EricGebhart/clj-pail-tap) 
at GitHub, an extension library for
David Cuddeback's [clj-pail library](https://github.com/dcuddeback/clj-pail)
The end result is
[The Pail-Schema library](http://GitHub.com/EricGebhart/Pail-schema"
which is much
simpler than the thrift based libraries. For the most part pail-schema
simply combines [clj-pail-tap](https://github.com/EricGebhart/clj-pail-tap"
and 
[pail-fressian](https://github.com/EricGebhart/pail-fressian)
so that it is then a simple matter of
creating a pail-structure which uses them.

All of the code in this post is available in the 
[pail-schema-example](https://github.com/EricGebhart/pail-schema-example)
repository. Just like
the last time, clone my repository, fire up a REPL and follow along!

---
## Prismatic Schema 

Using Prismatic Schema to define and enforce the data shape in a database does not seem to be exactly what Prismatic had in mind when they created it. However it does work very well
as a Graph Schema replacement. Instead of just throwing exceptions when data does not fit the schema, Prismatic Schema actually prints mostly reasonable messages that will tell you
why your data doesn't fit the schema. In addition to that, with Schema 2.0, Prismatic schema can now do coercion of your data. These two things alone make Prismatic Schema a strong
competitor to Graph schema. Add in that everything including the schema and the data is native clojure code, and things are really looking good. There are a lot of advantages, and I'm having a hard time finding any disadvantages. Writing code is really simple when apples are just apples.

	*Better validation messages
	*Coercion
	*Schema is clojure code
	*Data is clojure data

As with the other examples I've recreated the same Schema here, using <a
[Prismatic Schema](https://github.com/prismatic/schema" 
. It defines a Data Unit which is
the thing all database entities are made of. A Data Unit can be a Person
Property or a Friendship Edge. A Person Property can either first-name,
last-name, age or location. Location is a map which contains :address,
:city, :county, :state, :country and :zip any of which can be nil.

```clojure
(ns pail-schema-example.people
  (:require [schema.core :as s]))

; a location structure.
(def Location
  "A schema for a location"
    {:address (s/maybe s/Str)
     :city    (s/maybe s/Str)
     :county  (s/maybe s/Str)
     :state   (s/maybe s/Str)
     :country (s/maybe s/Str)
     :zip     (s/maybe s/Str)})
                                        ; the basic union of properties
(def PersonProperties
  "A Union of possible properties"
     (s/either
       {:first-name s/Str}
       {:last-name s/Str}
       {:location Location}
       {:age s/Int}))

(def PersonProperty
  "A Person property "
    { :id s/Str  ; make this s/Uuid
      :property PersonProperties})

; an Edge.
(def FriendshipEdge
  "A schema for a friendship edge connector."
   {(s/required-key :id1) s/Str
    (s/required-key :id2) s/Str})

(def DataUnit
  "The basic DataUnit for the database."
    (s/either
      {(s/required-key :person-property) PersonProperty}
      {(s/required-key :friendshipedge) FriendshipEdge}))
```

This schema maps almost exactly to the graph schemas used in the previous
examples. As before this schema consists of a single Data Unit which is
a union of properties. Each property for a person becomes a single Data
Unit. Simple properties like first name, last name and age are defined
inline as a part of the Person Properties union. Although they could
also be defined separately in the same way that the Location property
is defined.


## Constructors

In addition to creating the schema, it is also helpful to create
some constructors that will make creating a Data Unit a simple
task. These constructors are similar to those used in the <a
[Pail-Fressian example.](http://ericgebhart.com/fressian-pail-and-cascalog/)
In this example I also retained the type
hints although they provide no real value since they do not persist
through coercion or when read back from a pail.

The very first function, 'master-schema' provides a way for any functions
to get a handle to the Data Unit schema. Everything else is just to help
make it easy to create the various parts of a Data Unit.

```clojure
(defn master-schema
  "Return the master schema"
  [] DataUnit)

(defn person-property [id property]
  ^{:type ::PersonProperty}
  {:id id :property property})

(defn first-name [name]
  ^{:type ::FirstName}
  {:first-name name})

(defn last-name [name]
  ^{:type ::LastName}
  {:last-name name})

(defn age [name]
  ^{:type ::Age}
  {:age name})

(defn location [{:keys [address city county state country zip]}]
  ^{:type ::Location}
  {:location {:address address :city city :county county :state state :country country :zip zip}})

(defn friendshipedge [id1 id2]
  ^{:type ::friendshipEdge}
   {:id1 id1 :id2 id2})

(defn dataunit [key property]
  ^{:type ::DataUnit}
  {key property})

;helpers to build Data Units from a Person Property or Friendship edge.
(defn create-person-property [id property]
  (dataunit :person-property (person-property id property)))

(defn create-friendshipedge [id1 id2]
  (dataunit :friendshipedge (friendshipedge id1 id2)))

```

---
## Creating some Data Units

Creating our DataUnits is as straight forward as can be, and what we end up with plain old clojure data.

```clojure
(def du1-1 (p/create-person-property "123" (p/first-name "Eric")))
(def du1-2 (p/create-person-property "123" (p/last-name "Gebhart")))
(def du1-3 (p/create-person-property "123" (p/location {:address "1 Pack Place"
                                                                       :city "Asheville"
                                                                       :state "NC"})))
(def du1-4 (p/create-person-property "123" (p/age "40")))
(def du1-5 (p/create-person-property "123" (p/age 50)))

(def du2-1 (p/create-person-property "456" (p/first-name "Frederick")))
(def du2-2 (p/create-person-property "456" (p/last-name "Gebhart")))
(def du2-3 (p/create-person-property "456" (p/location {:address "1 Wall Street"
                                                                       :city "Asheville"
                                                                       :state "NC"})))
(def du3 (p/create-friendshipedge "123" "456"))

(def objectlist [du1-1 du1-2 du1-3 du1-5 du2-1 du2-2 du2-3 du3]

```

---
## Validation
All we've done so far is create some basic clojure data maps. Now we
can validate them with our schema. This is all we need to validate the
du1-1 data unit. On success we get the original du1-1 as a return.

```clojure
(s/validate (p/master-schema) du1-1)
```

Here is a not so friendly way to validate everything in our list. If
anything fails, you won't really know which one it is.

```clojure
(map #(s/validate %1 %2) (repeat (p/master-schema)) objectlist)
```

---
## Coercion

Of all the Data Unit's defined, there is only one that is invalid,
du1-4 has age as a string rather than an integer. The error message is
only slightly better than the exception we would get from thrift.

```clojure
(s/validate (p/master-schema) du1-4)
-> ExceptionInfo Value does not match schema: (not (every? (check % a-clojure.lang.PersistentArrayMap) schemas))  schema.core/validate (core.clj:165)  
```

But we can coerce du1-4 into the shape we want. Prismatic schema currently
provides two coercers, and you can also write your own. The two provided
are 'json-coercer-matcher' and 'string-coercer-matcher'. One thing to
be aware of, if you are counting on type hints, meta does not survive
coercion. Nor does it survive the roundtrip from a pail. You can see
that by examining the types before and after a coercion or returning
from a query.

From the code: 

<ul>
<li> Json-coercer-matcher <p>
"A matcher that coerces keywords and keyword enums from strings, and longs and doubles
     from numbers on the JVM (without losing precision)"</p></li>
<li> String-coercer-matcher <p>
"A matcher that coerces keywords, keyword enums, s/Num and s/Int,
     and long and doubles (JVM only) from strings."
</p></li>
</ul>

Prismatic's coercer takes a schema and coercion matcher and returns a
coercion function. Here are two simple wrappers for using both coercers
with our Data Unit schema.

```clojure
(def jcoerce-dataunit
  (coerce/coercer (p/master-schema) coerce/json-coercion-matcher))

(def coerce-dataunit
  (coerce/coercer (p/master-schema) coerce/string-coercion-matcher))
```

To get a good version of the du1-4 Data Unit, all we have to do is call
the coercer on it. Then we can add it to the object list for insertion
to the database.

```clojure
;get a good version of du1-4
(def du1-4c (coerce-dataunit du1-4))

(def objectlist (conj objectlist du1-4c))
```

---
## Defining a Pail

Defining a pail is a little bit different now since clj-pail-tap is
adding some extra functionality over the old Pail Structure definition
found in clj-pail. There is now the option of a Schema, rather than
a data Type, and there is also Tap Mapper and Property path generator
entries. Otherwise it is still rather straight forward. We need to specify
the Fressian serializer, and a partitioner that knows how to look at
native clojure data rather than disecting Thrift objects. Overall this
part is not so different than before.

```clojure
(ns pail-schema-example.data-unit-pail-structure
  (:require [clj-pail-tap.structure :refer [gen-structure]]
            [pail-fressian.serializer :as s]
            [pail-schema.partitioner :as p]
            [pail-schema.core :as pc]
            [pail-schema.tapmapper :as t]
            [pail-schema-example.people :as people])
  (:gen-class))

(gen-structure pail-schema-example.DataUnitPailStructure
               :schema (people/master-schema)
               :serializer  (s/fressian-serializer)
               :partitioner (p/property-name-partitioner (people/master-schema))
               :tapmapper   (t/property-name-tap-mapper)
               :property-path-generator pc/property-paths)
```

---
#### The Partitioner
We've already seen people/master-schema and the fressian-serializer is the one we get from pail-fressian. The rest is code we'll need. Pail-Schema provides a fairly generic partitioner, tapmapper and property-path generator. I experimented with type/meta information, and named schemas, all of which seem like they might make things simpler and more flexible but in the end were not that helpful or persistent. 

The easiest thing is still the way that things work with thrift. Look
at the property names/keys in the data and use those to create
partitions. This also means that the tap mapper can use the schema to
do it's work and everything will be consistent. The partitioner doesn't
look too much different from the two level property name partitioners in
the other posts. It looks for anything ending in [Pp]roperty and looks
for :property inside of that for a second level directory.

```clojure
p/VerticalPartitioner
  (p/make-partition
    [this object]
    (let [res (vector (name (first (keys object))))]
          (if (re-find #"^.*[Pp]roperty$" (first res))
            (let [subunion (:property ((first (keys object)) object))]
              (conj res (name (first (keys subunion)))))
            res)))
```

---
#### The Tap Mapper
The Tap Mapper code is supposed to take the output of the property
path generator and return a map of property paths where the compounded
property name is the key. It is totally up to you how to construct the
keys, but the results should match the paths that the partitioner creates.

```clojure
(defn property-name-taps [path]
  "fields ending in [Pp]roperty are partitioned further. ie. :first_name ['property' 'first_name']
   for partitioners where the field name is the directory name."
  (let [propregex #"^.*[Pp]roperty$"
        res (name (first path))
        subunion (if (and (re-find propregex res) (> (count path) 2)) (nth path 2) nil)
        pname (let [prefix (clojure.string/replace res propregex "-")]
               (if subunion
                 (if (= prefix "-")
                   (name subunion)
                   (clojure.string/join prefix (name subunion)))
                 res))]
    (conj [(keyword pname)]  (vec (if subunion (conj [res] (name subunion)) [res])))))

(defn property-name-tap-mapper
  "returns a union name property tap mapper"
  []
  property-name-taps)
```

It's quite possible that code could be shorter. But it does what it
should. Now the Tap Mapper needs to be fed with a list of property
paths. With thrift it was easy to traverse the java objects and
return a list of property paths. With Prismatic Schema it's about the
same. Pail-Schema provides this functionality with the property-paths
function. Property-paths is a descent parser which could use some
fleshing out. It currently does not support named schemas, and there is
the possiblity of other problems as well. It does currently support this
simple schema, which is good enough for now.

That finishes up our Pail Structure, now we just need to use it. Remember
your Pail Structure is a gen-class so it needs to be :aot compiled. To
play with the pail structure we just need an instance. Then we can ask
it all sorts of things. To see everything it can do take a look at the 
[defrecord.](https://github.com/EricGebhart/clj-pail-tap/blob/master/src/clj_pail_tap/structure.clj)
One of the easier things we can do is ask the partitioner for the partition
target of one of our data units.  We can also get the tap mapper function.

```clojure
pail-schema-example.example> (def ps (DataUnitPailStructure.))
#'pail-schema-example.example/ps
pail-schema-example.example> (.getTarget ps du1-1)
("person-property" "first-name")
pail-schema-example.example> (.getTarget ps du1-2)
("person-property" "last-name")
pail-schema-example.example> (.getTarget ps du1-3)
("person-property" "location")
pail-schema-example.example> (.getTapMapper ps)
#<tapmapper$property_name_taps pail_schema.tapmapper$property_name_taps@26f557bf>
```

---
#### Tap Maps
Part of the core functionality of clj-pail-tap is to also provide easy
ways to get to the tap maps. These functions will use a pail structure
or a pail connection, and work with whatever you have set up wether it
is a thrift type or a prismatic schema. So in addition to the functions
in the Pail structure we can also do things like this.

```clojure
(pail/list-taps ps)
-> (:person-property :first-name :last-name :location :age :friendshipedge)
(pail/tap-map ps)
-> {:person-property ["person-property"], :first-name ["person-property" "first-name"], :last-name ["person-property" "last-name"], :location ["person-property" "location"], :age ["person-property" "age"], :friendshipedge ["friendshipedge"]}
```

---
## Using the Pail

Our Pail Structure seems to be working fine. But we haven't written
anything to the pail yet. As with the previous examples this part is
pretty simple.

```clojure
(def mypail (pail/find-or-create (DataUnitPailStructure.) "example_output"))
(pail/write-objects pc objectlist)
```

The pail now has some data and the entire pail looks like this.

<pre>
─(16:41:%)── tree example_output
example_output
├── friendshipedge
│   └── 23c58dc8-def8-4613-a11e-8101cddf4432.pailfile
├── pail.meta
└── person-property
    ├── age
    │   └── 23c58dc8-def8-4613-a11e-8101cddf4432.pailfile
    ├── first-name
    │   └── 23c58dc8-def8-4613-a11e-8101cddf4432.pailfile
    ├── last-name
    │   └── 23c58dc8-def8-4613-a11e-8101cddf4432.pailfile
    └── location
        └── 23c58dc8-def8-4613-a11e-8101cddf4432.pailfile
</pre>


---
#### The simplist Cascalog Query
This is where it starts to get fun. With thrift, we always had a thrift
object to deconstruct. With these data objects there is no real need. We
can look at them and use them as they are. Deconstructing the data for
cascalog is simpler, and more flexible than with thrift data. We can do
a raw query with no deconstruction and still see what we've got.

```clojure
(??<- [?data] ((pail/pail->tap mypail) _ ?data))
-> ([{:person-property {:id "123", :property {:location {:address "1 Pack Place", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}}}}] [{:person-property {:id "456", :property {:location {:address "1 Wall Street", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}}}}] [{:person-property {:id "123", :property {:first-name "Eric"}}}] [{:person-property {:id "456", :property {:first-name "Frederick"}}}] [{:person-property {:id "123", :property {:last-name "Gebhart"}}}] [{:person-property {:id "456", :property {:last-name "Gebhart"}}}] [{:person-property {:id "123", :property {:age 50}}}] [{:person-property {:property {:age 40}, :id "123"}}] [{:friendshipedge {:id1 "123", :id2 "456"}}])
```


---
#### partial deconstruction
It could be that what we want back from cascalog is not so deconstructed
at all. Maybe all we want is to deconstruct it far enough to be joined
by the query. In this case that means getting two values, id and Person
Property, whatever it is. The defmap for that is really simple, and it
works for every kind of Person Property we have.

```clojure
(defmapfn pprop [{:keys [person-property]}]
    "Deconstruct a Person property"
    (into [(:id person-property)] (:property person-property)))
```

Using this defmap with location gives us location as the map it was
original created as. This is potentially a much more useful format
downstream.

```clojure
(defn loc-prop-query [pail-connection]
  (let [ptap (pail/get-tap pail-connection :location)]
    (??<- [?id ?property]
          (ptap _ ?data)
          (pprop ?data :> ?id ?property))))

(loc-prop-query mypail)
> (["123" [:location {:address "1 Pack Place", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}]] 
   ["456" [:location {:address "1 Wall Street", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}]])
```

---
## Getting Pail Taps
Notice that getting taps is very easy with the tap mapper doing all the
work underneath. It's also possible to leverage the tap-mapper in other
ways. We could create a tap that includes all person properties but
not friendship edges. That person property defmap would be very unhappy
if it encountered a friendship edge. In this particular case there is
an easy way to make sure we only get person properties, (pail/get-tap
mypail :person-property) would do the trick. But if the schema were
more complex that might not get it. It is possible to create a tap
from multiple paths. Leveraging the tap mapper we can select the paths
explicitly by keyword. The following function uses the base pail->tap
function to create a custom tap which explicitly lists first-name,
last-name, location and age.

```clojure
(defn prop-tap
  "Get a tap for all the person property partitions in one."
  [pail-connection]
  (pail/pail->tap pail-connection
                  :attributes (map #(%1 %2)
                                   [:first-name :last-name :location :age]
                                   (repeat (pail/tap-map mypail)))))

(defn personprop-query [pail-connection]
  (let [ptap (prop-tap pail-connection)]
    (??<- [?id ?property]
          (ptap _ ?data)
          (pprop ?data :> ?id ?property))))


(personprop-query mypail)
-> (["123" [:location {:address "1 Pack Place", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}]] 
    ["456" [:location {:address "1 Wall Street", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}]] 
    ["123" [:first-name "Eric"]] 
    ["456" [:first-name "Frederick"]] 
    ["123" [:last-name "Gebhart"]] 
    ["456" [:last-name "Gebhart"]] 
    ["123" [:age 50]] 
    ["123" [:age 40]])
```

---
## full deconstruction
Of course it's also very easy to deconstruct the data units when we query
them. There are really only two flavors of person property, Location
and the other simple properties. The defmap functions for both of them
are much simpler than those created for thrift and it definitely seems
like these could be improved upon.

```clojure
(defmapfn sprop [{:keys [person-property]}]
    "Deconstruct a property object"
    (into [(:id person-property)] (vals (:property person-property))))

(defmapfn locprop [{:keys [person-property]}]
    "Deconstruct a location property object"
    (into [(:id person-property)] (vals (:location (:property person-property)))))
``` 

Now we can query for and deconstruct everything. Here's what that looks like.

```clojure
(defn get-everything [pail-connection]
  (let [fntap (pail/get-tap pail-connection :first-name)
        lntap (pail/get-tap pail-connection :last-name)
        loctap (pail/get-tap pail-connection :location)]
    (??<- [?first-name ?last-name !address !city !county !state !country !zip]
          (fntap _ ?fn-data)
          (lntap _ ?ln-data)
          (loctap _ ?loc-data)
          (sprop ?fn-data :> ?id ?first-name)
          (sprop ?ln-data :> ?id ?last-name)
          (locprop ?loc-data :> ?id !address !city !county !state !country !zip))))

(get-everything mypail)
-> (["Eric" "Gebhart" "1 Pack Place" "Asheville" nil "NC" nil nil] 
    ["Frederick" "Gebhart" "1 Wall Street" "Asheville" nil "NC" nil nil])
```

<br/>
## Conclusion
I've been working on various forms of using Pail within Clojure for
several months now. Using the Fressian Pail with no schema but just
constructors was really nice, but seemed a bit loose in some ways. Maybe
that is ok, but I definitely prefer having some sort of schema to keep
the shape of the data consistent. Prismatic schema does that by offering
validation and coercion, both of which are nicer than what Graph Schema
and Thrift offers. Keeping all the data in native data formats has a
transparency that is refreshing. Using Prismatic Schema with Pail-Fressian
and clj-pail-tap has a simplicity and power that is hard to dismiss. I'll
be continuing to use this framework and see where it leads.
