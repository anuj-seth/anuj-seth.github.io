---
layout: post
title: "From a function to a macro and back again"
date: 2020-08-02
categories: clojure
tags: clojure macro
style: |
  body {
  font-size: 15px;
  } 
---
[Practical Common Lisp](http://www.gigamonkeys.com/book/) by Peter Seibel has the best programming exercises I have ever seen in an introductory computer book. In the preface of the book Seibel states the goal of dispelling the myth that lisp is not a practical language and by the end of the book he achieves that goal by writing a unit testing framework, a spam filter, a binary data parser and a streaming music service along the way.  
&nbsp;  
In chapter 3 he writes a program to maintain a database of compact discs (CDs). For my younger readers who may not know what CDs are - they were a storage media popular till the early 2000s to share music and data and were eventually killed off by the advent of streaming services. Compact discs may not be a thing anymore but lisp is still relevant and the problems Seibel tackles are fun to do in Clojure.

### The story of the where clause
The CD database is a simple vector of hash maps with :artist, :title, :rating and :ripped? as the keys. The :ripped? attribute tells us if a particular CD in our collection has been converted to MP3 or not, and is the only attribute which is nullable.  
```
[{:artist "Shaan" :title "Tanha Dil" :rating 7 :ripped? true}
 {:artist "Junoon" :title "Azadi" :rating 10 :ripped? nil}
 {:artist "Silk Route" :title "Boondein" :rating 10 :ripped? false}]
```
There are functions to enter data, print the contents of the database to the screen, dump the contents to a file, etc., but the most interesting function is the implementation of the where clause, used like it's namesake in SQL, to filter records.  
```
user> (filter (where :ripped? true :artist "Shaan") 
              [{:artist "Shaan" :title "Tanha Dil" :rating 7 :ripped? true} 
               {:artist "Junoon" :title "Azadi" :rating 10 :ripped? nil} 
               {:artist "Silk Route" :title "Boondein" :rating 10 :ripped? false}])
({:artist "Shaan", :title "Tanha Dil", :rating 7, :ripped? true})
user> (filter (where :ripped? nil) 
              [{:artist "Shaan" :title "Tanha Dil" :rating 7 :ripped? true} 
               {:artist "Junoon" :title "Azadi" :rating 10 :ripped? nil} 
               {:artist "Silk Route" :title "Boondein" :rating 10 :ripped? false}])
({:artist "Junoon", :title "Azadi", :rating 10, :ripped? nil})
```
The first implementation of _where_ is a function that returns a predicate.  
```
(defn where
  [& {:keys [artist title rating ripped?] :or {ripped? :not-set} :as clauses}]
  (fn [record]
    (and (or (nil? artist) (= (:artist record) artist))
         (or (nil? title) (= (:title record) title))
         (or (nil? rating) (= (:rating record) rating))
         (or (= ripped? :not-set) (= (:ripped? record) ripped?)))))
```
This satisfies our requirement but adding more attributes in our CD record e.g., :year-released, :production-label, would mean that we have to modify the _where_ function every time we add an attribute. In addition, this code is specific to our CD database and will not work with a different database e.g., books by Terry Pratchett in our collection.  
If we could generate the above code using the input arguments to the _where_ function that would solve both the above problems.  
&nbsp;  
Whenever lisp and clojure programmers hear the words "generate code" they automatically reach for macros.
```
user> (defmacro where
        [& clauses]
        `(fn [~'record]
           (and ~@(for [[k# v#] (partition 2 clauses)]
                    `(= ~v# (~k# ~'record))))))

user> (macroexpand-1 '(where :ripped? nil :artist "Shaan"))
(clojure.core/fn
 [record]
 (clojure.core/and
  (clojure.core/= nil (:ripped? record))
  (clojure.core/= "Shaan" (:artist record))))
```
The _~'record_ syntax is to make a non-namespace qualified symbol for the argument to the generated function.  
Using an auto gensym directly i.e. record#, to generate the symbol will not work since the inner syntax quoted form will not see auto-gensym symbols from outside of that form.  
We could get the same result by using gensym to generate a symbol and then inject that into the syntax quoted forms.  
```
user> (defmacro where
        [& clauses]
        (let [args (gensym)]
          `(fn [~args]
             (and ~@(for [[k# v#] (partition 2 clauses)]
                      `(= ~v# (~k# ~args)))))))

user> (macroexpand-1 '(where :ripped? nil :artist "Shaan"))
(clojure.core/fn
 [G__12930]
 (clojure.core/and
  (clojure.core/= nil (:ripped? G__12930))
  (clojure.core/= "Shaan" (:artist G__12930))))
```
Seibel stops at this point and we could too.  
But looking carefully at the macro, I realized that we could check each attribute value pair in a function just as easily.  
```
(defn where
  [& clauses]
  (fn [record]
    (every? (fn [[k v]] (= v (record k)))
            (partition 2 clauses))))
```
&nbsp;  
Macros are powerful but never use a macro where a function will do.
