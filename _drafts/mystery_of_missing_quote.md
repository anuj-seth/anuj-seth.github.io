---
layout: post
title: "A missing quote raises symbols"
date: 2020-08-20
categories: clojure
tags: clojure macro
style: |
  body {
  font-size: 15px;
  } 
---
[Practical Common Lisp](http://www.gigamonkeys.com/book/) by Peter Seibel 
What is a symbol ?
(defmacro deftest
  [name & body]
  (print (instance? clojure.lang.Symbol name))
  `(defn ~name
     []
     (print (type ~name))
     (binding [*test-name* '~name]
       ~@body)))

When clojure reader sees the symbol that ~name resolves to defn handles it.
in the binding if i do not quote ~name it gets resolved to the function

