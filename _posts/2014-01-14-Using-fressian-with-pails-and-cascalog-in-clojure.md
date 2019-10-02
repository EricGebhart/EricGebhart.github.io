---
layout: post
title: "Fression, Pail and Cascalog"
description: "Using Fressian with Pails and Cascalog in Clojure."
Date: 2014-01-14 20:58:33
category: clojure
tags: [Fressian, Pail, Cascalog, Clojure, Big data]
---
In my [Previous post](http://ericgebhart.com/thrift-pail-cascalog-and-clojure/) I wrote
about using Thrift, Pail and Cascalog. In this post I'll replace thrift
and graph schema with Fressian and native data types. It turns out that
Fressian, Pail, and Cascalog go together like peanut butter and jelly. As
before this is based on David Cuddeback's clj-pail and pail-cascalog
libraries. Instead of pail-thrift I have a new <a
[Pail-Fressian library](http://GitHub.com/EricGebhart/Pail-Fressian)
which handles the details of using Fressian with Pail. <a
[Pail-Fressian on clojars](https://clojars.org/pail-fressian) 
All of the code in this post is available in an [example repository]
(https://github.com/EricGebhart/Pail-Fressian/blob/master/src/example/example.clj)
Clone my repository, fire up a REPL and follow along!

Leaving out thrift greatly simplifies everything.In fact, if you haven't
read my previous post</a>, you should go do that so you can fully appreciate the simplicity
of using Fressian instead of Thrift. You can learn more about Fressian
by watching [Stuart Halloway's presentation on Fressian.](http://www.youtube.com/watch?v=JArZqMqsaB0)
Fressian is not a serializer, but does a really good job at it.

---

## Look ma! No Schema!

We don't need a schema for this. Although I did make some simple
types to make things easier. How you do it is totally up to you. This
is an incredibly flexible system. These types are roughly modeled
after the thrift objects I used before. But they are actually simpler
while retaining the same flexibility as the graph schema unions and
structures. There is a PersonProperty that holds an id and a property,
and there are three properties, FirstName, LastName and Location.

```clojure
(defn person-property [id property]
  ^{:type ::PersonProperty}
  {:id id :property property})

(defn first-name [name]
  ^{:type ::FirstName}
  {:first_name name})

(defn last-name [name]
  ^{:type ::LastName}
  {:last_name name})

(defn location [{:keys [address city county state country zip]}]
  ^{:type ::Location}
  {:address address :city city :county county :state state :country country :zip zip})

(defn friendshipedge [id1 id2]
  ^{:type ::friendshipedge}
  {:id1 id1 :id2 id2})
```

### Creating some data
The code to create some of these is very straight forward. Since we have
fressian, there's no need to build any special objects. We just put the
data together how we want it.

```clojure
(def du1-1 (person-property "123" (first-name "Eric")))
(def du1-2 (person-property "123" (last-name "Gebhart")))
(def du1-3 (person-property "123" (location {:address "1 Pack Place"
                                                   :city "Asheville"
                                                   :state "NC"})))
(def du2-1 (person-property "123" (first-name "Frederick")))
(def du2-2 (person-property "123" (last-name "Gebhart")))
(def du2-3 (person-property "123" (location {:address "1 Wall Street"
                                                   :city "Asheville"
                                                   :state "NC"})))
(def du3 ( friendshipedge "123" "abc"))

(def objectlist [du1-1 du1-2 du1-3 du2-1 du2-2 du2-3 du3])
```

These data objects look as simple as you would expect.
```clojure
du1-1
=>{:id "123", :property {:first_name "Eric"}}
du1-2
=>{:id "123", :property {:last_name "Gebhart"}}
du1-3
=>{:id "123", :property {:address "1 Pack Place", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}}
```

---

## The Pail Partitioner

The Pail partitioner is also fairly straight forward. The partitioner
has no problems looking around at these objects, and we've given them
types which means we can control pretty much anything we want just based
on the type name. This partitioner uses the type name as the directory
name. If the type name ends in [Pp]roperty, it tries to get the type of
the :property field and that becomes a second level directory. Here's
the make-partition function from the partitioner. You will want to make
a partitioner to fit your data, but this partitioner might be a good
place to start.

```clojure
 (p/make-partition
    [this object]
    (let [res (vector (name (type object)))]
      (if (re-find #"^.*[Pp]roperty$" (first res))
        (let [prop (name (type (:property object)))]
          (conj res prop))
        res)))
```

Now that we have some data and a partitioner, We need a Pail Structure. it
looks just like the others. it's just got Fressian written all over
it instead of thrift. This is a gen-class so remember to recompile,
and restart your REPL when you change anything. Thankfully there's not
much to change.

```clojure
(ns pail-fressian.fressian-pail-structure
  (:require [clj-pail.structure :refer [gen-structure]]
            [pail-fressian.serializer :as s]
            [pail-fressian.partitioner :as p])
  (:gen-class))

(gen-structure pail-fressian.FressianPailStructure
               :serializer (s/fressian-serializer)
               :partitioner (p/fressian-property-partitioner))
```


---

## Create a Pail

Now we need a pail so we can write some data. This is the same as the thrift example.

```clojure
(def mypail (find-or-create (fressianPailStructure.) "example_pail"))
(write-objects mypail objectlist)
```

Wow, that was easy. Here's how the pail looks.

```
example_pail
├── PersonProperty
│   ├── FirstName
│   │   └── be3242ba-2922-427a-9d72-109b6c5ed9fb.pailfile
│   ├── LastName
│   │   └── be3242ba-2922-427a-9d72-109b6c5ed9fb.pailfile
│   └── Location
│       └── be3242ba-2922-427a-9d72-109b6c5ed9fb.pailfile
├── friendshipedge
│   └── be3242ba-2922-427a-9d72-109b6c5ed9fb.pailfile
└── pail.meta
```

---

## Cascalog

Now let's get some data back out. We can get a basic tap right at PersonProperty and take a look at what we have.

```clojure
(defn prop-tap [pail-connection] (pcas/pail->tap pail-connection :attributes [["PersonProperty"]]))

(defn raw-query [pail-connection]
  (let [ptap (prop-tap pail-connection)]
    (??<- [?data] (ptap _ ?data))))

(raw-query mypail)

=>([{:id "123", :property {:address "1 Pack Place", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}}] 
[{:id "123", :property {:address "1 Wall Street", :city "Asheville", :county nil, :state "NC", :country nil, :zip nil}}] 
[{:id "123", :property {:first_name "Eric"}}] [{:id "123", :property {:first_name "Frederick"}}] 
[{:id "123", :property {:last_name "Gebhart"}}] [{:id "123", :property {:last_name "Gebhart"}}])
```

These look a lot different from the raw thrift objects we got back in
the thrift example. Because they are native clojure data types they are
a lot easier on the eyes.
We only need one function to deconstruct these three data types, and it's an easy one. Because defmapfn's are functions we can try it out without cascalog.

```clojure
(defmapfn sprop [du]
    "Deconstruct a property object"
    (into [(:id du)] (vals (:property du))))

(sprop du1-1)
=>["123" "Eric"]
(sprop du1-2)
=>["123" "Gebhart"]
(sprop du1-3)
=>["123" "1 Pack Place" "Asheville" nil "NC" nil nil]
```

Let's put this all together into a query! We need some taps for our pail partitions and some queries to use them. 

```clojure
(defn fn-tap [pail-connection] (pcas/pail->tap pail-connection :attributes [["PersonProperty" "FirstName"]]))
(defn ln-tap [pail-connection] (pcas/pail->tap pail-connection :attributes [["PersonProperty" "LastName"]]))
(defn loc-tap [pail-connection] (pcas/pail->tap pail-connection :attributes [["PersonProperty" "Location"]]))


(defn get-everything [pail-connection]
  (let [fntap (fn-tap pail-connection)
        lntap (ln-tap pail-connection)
        loctap (loc-tap pail-connection)]
    (??<- [?first-name ?last-name !address !city !county !state !country !zip]
          (fntap _ ?fn-data)
          (lntap _ ?ln-data)
          (loctap _ ?loc-data)
          (sprop ?fn-data :> ?id ?first-name)
          (sprop ?ln-data :> ?id ?last-name)
          (sprop ?loc-data :> ?id !address !city !county !state !country !zip))))

```


And here it goes.

```clojure
(get-everything mypail)
=>(["Eric" "Gebhart" "1 Pack Place" "Asheville" nil "NC" nil nil] 
["Frederick" "Gebhart" "1 Pack Place" "Asheville" nil "NC" nil nil]
```

Using Fressian instead of thrift makes almost everything easier. Even though Fressian is not a serializer, it makes a great serializer and it works beautifully with Pail. The simplicity
of the data objects in this example verses the thrift and graph example
make everything easier from beginning to end. Fressian, Pail and Cascalog
make a very flexible and powerful system.


