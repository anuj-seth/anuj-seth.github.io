---
layout: post
title: "Things I forget when I am being lazy"
date: 2020-09-18
categories: clojure
tags: clojure laziness
style: |
  body {
  font-size: 15px;
  } 
---
These are some things that have at one time or the other tripped me up while working with lazy sequences.  
The examples are made up, the time spent scratching my head was real.  

### Laziness and mutable state do not play well together
```
(def some-shared-state (atom 0))

(defn manipulate
  []
  (swap! some-shared-state inc)
  (map (fn [n]
         (+ @some-shared-state n))
       [1 1 1]))

(def x (manipulate))

(def y (manipulate))
```
I evaluated the above code on the REPL and expected the first call to the function manipulate to increment the atom and add that to each element of the vector [1 1 1] giving x the value [2 2 2] and the subsequent call increments the atom to 2 and gives y the value [3 3 3].  
```
user=> x
(3 3 3)
user=> y
(3 3 3)
```
This did not make sense until I realized that the sequence produced by map in the manipulate function was realized only when I evaluated x and y on the REPL, while the swap! on the atom had taken place when I called the function.  
&nbsp;  
I am guilty of writing something similar to this code. That time the mutation and the lazy sequence were seperated by a few function calls and it took me much longer to figure out what was happening.  

### doseq/doall/etc will not realize nested lazy sequences
Forcing realization of a lazy sequence below using doall prints the nested vectors on the console.   
```
user=> (def y (doall (map println [[1 2 3] [4 5 6]])))
[1 2 3]
[4 5 6]
#'user/y
```
When I change the above code to print each element of the nested vector nothing is printed on the console.  
```
user=> (def y (doall (map #(map println %) [[1 2 3] [4 5 6]])))
#'user/y
```
doall forces the realization of the outer map not the inner one.  
If all you care about are side effects use doseq directly rather than a combination of doall and a lazy sequence producing function.  

### REPL realizes things
I was playing around on the REPL with some lazy sequences and ran this code  
```
user=> (doall (map (fn [x] (map println x)) [[1 2 3] [4 5 6]]))
1
2
3
4
5
6
((nil nil nil) (nil nil nil))
```
I got the side effect I wanted but did not care for return values.  
Being a conscientious lazy programmer I knew I did not want to hold on to the head of this sequence so I should use dorun instead.  
```
user=> (dorun (map (fn [x] (map println x)) [[1 2 3] [4 5 6]]))
nil
```
That was surprising - what magic was doall doing under the hood that dorun was not capable of performing ?  
I looked at the source code of doall and saw that it called dorun and then returned the sequence.  
I sat looking at that code for five minutes, willing it to divulge it's secrets until I realized that it was the REPL that was forcing the realization of the returned sequence from doall.  
&nbsp;  
That was the moment I knew I had to blog about these and then maybe next time I would remember these lessons.
