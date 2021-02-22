---
layout: post
title: "Thrift, Pail, Cascalog and Clojure"
description: "Using Thrift, Pail and Cascalog in Clojure."
category:
tags: [Thrift, Pail, Cascalog, Clojure, Graph Schema, Nathan Marz, big data]
---

Thrift, Pail, Cascalog and Clojure have been consuming my work for the last 2 months. It isn't that hard. It's just that there isn't much in the way of documentation.  In this post I'll show how to get all of these things working together. It's actually pretty easy and it's very slick once it's working. Many thanks to
[David Cuddeback](https://github.com/dcuddeback?tab=repositories) and his [clj-thrift, ](https://github.com/dcuddeback/clj-thrift) [clj-pail,](https://github.com/dcuddeback/clj-pail) [pail-thrift,](https://github.com/dcuddeback/pail-thrift) and [pail-cascalog](https://github.com/dcuddeback/pail-cascalog) libraries.

At some point in time I read this [post](https://nathanmarz.com/blog/thrift-graphs-strong-flexible-schemas-on-hadoop.html) about Thrift and Graph Schemas by Nathan Marz. Then I started reading his book [Big Data] (http://manning.com/marz/).
Nathan outlines the use of Pail for managing data. Pail simplifies things dramatically over the pains I've seen with other tools, or worse, written from scratch in java. Pail is no different than many other Big data technologies, it is Java. Thankfully we have Clojure to fit everything together seamlessly. Cascalog can handle the map reduce, Graph, Thrift and Pail can manage the data. It was looking like a fun project.

So lets get all of this working in Clojure. Its like many other things, its not all that hard once you know how. If you want to follow along this [project is available on my GitHub.](https://github.com/EricGebhart/thrift-pail-cascalog-example).

<hr/>

<h2>The Graph Schema</h2>

The graph schema is where it all starts. Using a Union for properties allows for the schema to adapt and change without impacting code, yet still enforcing a consistent shape to the data.  In this example we have a Union called Person Property Value. This is where all properties go in this simple example. Location is the only complex property, it is a struct with several optional values. At the intermediat level here is a structure called person property, This serves to connect an id with a property.  Finally we have a DataUnit Union. This is the Thrift object that we will be storing. Everything in the database is a Data Unit.


```java

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

<strong>Now to compile this thing and create some java code.</strong>

There is a plugin for leiningen called <a href="https://github.com/xsc/lein-thriftc" title="leiningen thriftc" target="_blank">thriftc</a> that can automate this. Take a look at this <a href="https://github.com/EricGebhart/thrift-pail-cascalog-example/blob/master/project.clj" title="project.clj at Github" target="_blank">project's project.clj</a>.  If you use thriftc then leiningen will take care of it all.
<pre>lein clean; lein compile</pre>

Underneath it all this is pretty much all there is to it.
    thrift -v --gen java -out ./ people.thrift

<hr/>

<h2>Thrift</h2>

Now we just need to create some thrift objects.  That turns out to be pretty easy.

```clojure
    (def du1-1 (thrift/build DataUnit {:property {:id "123"
                                                    :property {:first_name "Eric"}}}))

    (def du1-2 (thrift/build DataUnit {:property {:id "123"
                                                    :property {:last_name "Gebhart"}}}))

    (def  du1-3 (thrift/build DataUnit {:property {:id "123"
                                                    :property { :location {:address "1 Pack Place"
                                                                            :city "Asheville"
                                                                            :state "NC"}}}}))
                                                                            
```


After a couple of these it becomes clear how this maps directly to the schema. It's also pretty obvious that it would be easy to automate.

Now that we have some thrift objects, Thrift and pail can work together to serialize these into the database. But it might be good to try looking inside these things. This is pretty easy with clj-thrift.

Getting the current value of the Data Unit union gives us a Person Property structure which has an id and a personPropertyValue named property.

```clojure
    (tu/current-value du1-1)
    =>#<PersonProperty PersonProperty(id:123, property:<PersonPropertyValue first_name:Eric>)>
```

We just need to go one level deeper to get to the id or the property. To get the value from a structure we need to give thrift/value a key.

```clojure
    (thrift/value (tu/current-value du1-1) :id)
    => "123"
```

Asking for the property gives us the PersonPropertyValue union.

```clojure
    (thrift/value (tu/current-value du1-1) :property)
    => #<PersonPropertyValue <PersonPropertyValue first_name:Eric>>
```


To get the value of the PersonPropertyValue union we ask for the current value.

```clojure
    (tu/current-value (thrift/value (tu/current-value du1-1) :property))
    => "Eric"
```

if we try the exact same code with a location data unit we see that yet another level is needed to get all the key values from the structure.

```clojure
    (tu/current-value (thrift/value (tu/current-value du1-3) :property))
    =>#<Location Location(address:1 Pack Place, city:Asheville, state:NC)>
```

Now we know enough to wrap these ideas up into functions. One function for the top level structure values like id and property in the PersonProperty
structure or id1 and id2 in the friendshipedge structure.
Another function for union values contained in a a secondary union (PersonPropertyValue) like 'name', and finally a special function to extract the
values from the lowest level structure such as 'city' in the location structure.

```clojure
    (defn property-value [du name]
      "get the named field value from the current structure in the top level union.
       Union -> struct : - value."
      (thrift/value (tu/current-value du) name))

    (defn property-union-value
      "get a value from a union, inside a struct inside a union.
       name is the property name inside the struct.
       Union -> Struct : -> Union - value."
      [du name]
      (tu/current-value (thrift/value (tu/current-value du) name)))

    (defn locprop
      "Deconstruct a location property object, which has an id and a location struct"
      [du]
      (into [(property-value du :id)]
            (map #(thrift/value (property-union-value du :property) %)
                 [:address :city :county :state :country :zip])))
```


Here is how they all work.

```clojure
    (property-value du1-3 :id)
    => "123"
    (property-union-value du1-1 :property)
    => "Eric"
    (locprop du1-3)
    => ["123" "1 Pack Place" "Asheville" nil "NC" nil nil]
```

<hr/>

<h2>Pail</h2>

Pail turns out to be not all that hard to use. But it does take some setup.  First we need a Pail Structure. This tells pail how to behave. It gives pail the serializer which we get from thrift. The Pail Structure also gives pail our partitioner which tells pail how to partition the data. Here is our Pail Structure.

```clojure
    (ns thrift-pail-cascalog-example.data-unit-pail-structure
    (:require [clj-pail.structure :refer [gen-structure]]
                [pail-thrift.serializer :as s]
                [pail-thrift.partitioner :as p]
                [thrift-pail-cascalog-example.partitioner :as p2])
        (:import [people DataUnit])
    (:gen-class))

    (gen-structure thrift-pail-cascalog-example.DataUnitPailStructure
                :type DataUnit
                :serializer (s/thrift-serializer DataUnit)
                :partitioner (p/union-partitioner DataUnit))
```

The important bits are the type, serializer and partitioner.  For now we are using the Union-partitioner that came with Pail-Thrift. The only other special piece of this is that this file must be precompiled,  So in your project.clj you'll need something like this.

```clojure
    :aot [thrift-pail-cascalog-example.data-unit-pail-structure]
```

You'll want to do a <strong>'lein clean; lein compile'</strong> anytime you change this.

The partitioner is fairly straightforward as well. It has two primary methods make-partition and validate. Make-partition returns a vector of folder names, validate simply checks the first entry in that same list for validity and returns a vector of <pre><code>[true (rest dirs)]</code></pre> if it's ok. false otherwise.  This partitioner only looks at the top level Union and returns the field id for the current field as the directory name. Validate checks to see if the number given is in the list of field ids.

```clojure
    (defrecord ^{:doc "A pail partitioner for Thrift unions. It requires a type, which must be a subtype
                  of `TUnion`. The partitioner will partition based on the union's set field so that
                  all union values with the same field will be placed in the same partition."}
    UnionPartitioner
    [type]

    p/VerticalPartitioner
    (p/make-partition
        [this object]
        (vector (union/current-field-id object)))

    (p/validate
        [this dirs]
        [(try
        (contains? (type/field-ids type)
                    (Integer/valueOf (first dirs)))
        (catch NumberFormatException e
            false))
        (rest dirs)]))

    (defn union-partitioner
    "Returns a `UnionPartitioner`. `type` should be a subclass of `TUnion`. Note that it should be the
    class, and not an instance."
    [type]
    (UnionPartitioner. type))
```


Using this pail is pretty simple. First we create a Pail Spec from our DataUnitPailStructure, then we create the pail from the PailSpec.

```clojure
    (def mypail (-> (DataUnitPailStructure.)
                (pl/spec)
                (pl/create "example_output"  :fail-on-exists false)))
```


Now we write to it. We've only got seven objects so this will do.

```clojure
    (defn write-them [pail-connection]
    (with-open [writer (.openWrite pail-connection)]
        (doto writer
            (.writeObject du1-1)
            (.writeObject du1-2)
            (.writeObject du1-3)
            (.writeObject du2-1)
            (.writeObject du2-2)
            (.writeObject du2-3)
            (.writeObject du3))))

    (write-them mypail)
```

If we look in our example_output directory we will see that we have 2 directories.  1 and 2.
looking at the schema will show that 1 is the field id for property and 2 is the field id for friendship edges.
<pre>
--- tree example_output
example_output
├── 1
│   └── 74fe8e95-cadb-47da-801d-3ff898edfc12.pailfile
├── 2
│   └── 74fe8e95-cadb-47da-801d-3ff898edfc12.pailfile
└── pail.meta</pre>

That's great but it's not what we need or want. There are many other [Reasons to vertically partition data.](https://www.google.com/search?q=reasons+to+vertically+partition+hadoop+data&oq=reasons+to+vertically+partition+hadoop+data)
In addition to all of those reasons vertical partitioning also makes using Cascalog with this data much easier. And while we're at it lets change the directory names to the field names so we can see what is going on. Nathan argues well for using field id's and it makes sense to divorce the field names from the database structure. But when it comes to understanding how this stuff works names are good.

<hr/>

<h4>Back to the partitioner</h4>
We need a partitioner that will not only give back names, but also drill deeper into the object if the name contains 'property'. If we do just that much we  will have a much more powerful generic partitioner.

Here's our new partitioner, you can see that all we do is look for the name and go one level deeper if the name is 'property' and that 2nd tier structure has a :property field.

```clojure
    (defrecord ^{:doc "A 2 level pail partitioner for Thrift unions. It requires a type,
                 which must be a subtype of TUnion. The partitioner will partition
                 based on the union's set field name so that all union values with
                 the same field will be placed in the same partition. If a field's
                 name is property or ends in property or Property the partitioner
                 will also partition by the union found in the :property field of
                 that structure.

                    Union PropertyValue {
                        1: name;
                        2: lastname;
                    }

                    Struct PersonProperty {
                        1: string id;
                        2: PropertyValue property;    /* -- this second level property field is
                                                            required. it is the target subunion */
                    }

                    Union DataUnit {
                        1: PersonProperty Property;
                        2: string Things;
                    }

                  Partitioning DataUnit will result in /property/name, /property/lastname,
                  and /Things as the partitions."}

    UnionNamePropertyPartitioner
    [type]

    p/VerticalPartitioner
    (p/make-partition
        [this object]
        (let [res (vector (union/current-field-name object))]
        (if (re-find #"^.*[Pp]roperty$" (first res))
            (let [subunion (thrift/value (union/current-value object) :property)]
            (conj res (union/current-field-name subunion)))
            res)))

    (p/validate
        [this dirs]
        [(try
        (contains? (type/field-names type)
                    (first dirs))
        (catch Exception e
            false))
        (rest dirs)])
    )

    (defn union-name-property-partitioner
    [type]
    (UnionNamePropertyPartitioner. type))

```

Now all we need to do is change our Pail Structure to point at this new partitioner.

```clojure
    :partitioner (p2/union-name-property-partitioner DataUnit)
```


Don't forget to recompile.
<pre><strong> ---lein clean; lein compile.</strong></pre>

Reconnect to our pail and write our objects.

```clojure
    (def mypail (-> (DataUnitPailStructure.)
                    (pl/spec)
                    (pl/create "example_output"  :fail-on-exists false)))

    (write-them mypail)
```

Now let's check our output file tree.

<pre>
── tree example_output
example_output
├── friendshipedge
│   └── dd5208a2-1cf1-4185-930f-2cb0ecc1e837.pailfile
├── pail.meta
└── property
    ├── first_name
    │   └── dd5208a2-1cf1-4185-930f-2cb0ecc1e837.pailfile
    ├── last_name
    │   └── dd5208a2-1cf1-4185-930f-2cb0ecc1e837.pailfile
    └── location
        └── dd5208a2-1cf1-4185-930f-2cb0ecc1e837.pailfile
</pre>

Now that's more like it!  This behavior is important for many reasons not the least of which is ease of use with Cascalog.

<hr/>

<h2>Cascalog</h2>
When using Cascalog the first thing you need is a generator.  There are three flavors of generators but what we want is a Tap. To be specific, a Pail Tap.  These are easy to get because we just use our pail-connection. All we have to do is connect to a pail using the PailStructure and path or open an existing pail to create a connection.

Heres a basic pail tap.

```clojure
    (pcas/pail->tap pail-connection)
```

The problem with this tap is that it will bring back all the thrift objects it can find without any idea of what is in them. It could be handled but it's very messy and complicated. What we really want to do is leverage the vertical partitioning that is built into our data with Pail.  This turns out to be very easy.  One of the arguments to create a pail tap can be a vector of paths where each path is a vector. That tap then only brings back objects from those paths. Here is a way to create all the taps we need.

```clojure
    (defn first-name-tap [pail-connection]
        (pcas/pail->tap pail-connection :attributes [["property" "first_name"]] ))

    (defn last-name-tap [pail-connection]
        (pcas/pail->tap pail-connection :attributes [["property" "last_name"]] ))

    (defn location-tap [pail-connection]
        (pcas/pail->tap pail-connection :attributes [["property" "location"]] ))

    (defn friendedge-tap [pail-connection]
        (pcas/pail->tap pail-connection :attributes [["friendshipedge"]] ))
```


Let's try out the first name tap and see what happens.

```clojure
    (??<- [?data] (first-name-tap _ ?data))
    => (\[\#<DataUnit <DataUnit property:PersonProperty(id:123, property:<PersonPropertyValue first_name:Eric>)>>\]
        \[\#<DataUnit <DataUnit property:PersonProperty(id:abc, property:<PersonPropertyValue first_name:Frederick>)>>\])
```

That worked pretty well but everything is still inside a thrift object which isn't very useful. We need an operator to deconstruct it.  All we need is a defmapfn.  These have been renamed in [Cascalog 2] (https://groups.google.com/forum/#!msg/cascalog-user/F8EkFM7HiE0/c0HVP-zk5OUJ), everything that was def???op is now def???fn.  Additionally these guys are now real functions so we can use them like functions to test them out.  We need two operators, and we can use the functions we created before.

```clojure
    (defmapfn sprop [du]
        "Deconstruct a simple property object that only has an id and one value which
        is a union named 'property'."
        [(property-value du :id) (property-union-value du :property) ])

    ;location property deconstruction.
    (defmapfn locprop
    "Deconstruct a location property object, which has an id and a location struct"
    [du]
    (into [(property-value du :id)]
        (map #(thrift/value (property-union-value du :property) %)
             [:address :city :county :state :country :zip])))
```

These work as advertised.

```clojure
    (sprop du1-1)
    =>["123" "Eric"]
    (sprop du1-2)
    => ["123" "Gebhart"]
    (locprop du1-3)
    => ["123" "1 Pack Place" "Asheville" nil "NC" nil nil]
```

Now let's wrap up a full query using our taps and operators in a function.

```clojure
    (defn get-everything [pail-connection]
        (let [fntap (first-name-tap pail-connection)
            lntap (last-name-tap pail-connection)
            loctap (location-tap pail-connection)]
        (??<- [?first-name ?last-name !address !city !county !state !country !zip]
            (fntap _ ?fn-data)
            (lntap _ ?ln-data)
            (loctap _ ?loc-data)
            (sprop ?fn-data :> ?id ?first-name)
            (sprop ?ln-data :> ?id ?last-name)
            (structprop ?loc-data :> ?id !address !city !county !state !country !zip))))
```

Here it goes.

```clojure
    (get-everything mypail)
    =>(["Eric" "Gebhart" "1 Pack Place" "Asheville" nil "NC" nil nil]
       ["Frederick" "Gebhart" "1 Wall Street" "Asheville" nil "NC" nil nil])
```

Now that's cool.

<hr/>

<h2>Thrift, Pail, Cascalog and Clojure, all together!</h2>

Now that these are all working together it's time to explore.  All of this code is available and ready to run on my [thrift pail cascalog example project at my github.](https://github.com/EricGebhart/thrift-pail-cascalog-example).

Some things to consider are using field id's instead of field names for your vertical partitioning. It will mean that you can change a property name without impacting your existing data. Just don't change the field numbers in your schema.  Cascalog taps could easily be generated from a thrift data type. A generic defmapfn that could deconstruct any structure could also be created.  So wether the data is stored in directories of field id's or names makes no difference to us. With just a little bit of code all of this could disappear under the covers.
