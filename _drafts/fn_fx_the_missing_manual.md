---
layout: post
title: "fn-fx:The missing manual"
date: 2019-01-07
categories: clojure
tags: clojure fn-fx gui
style: |
  body {
  font-size: 15px;
  } 
---
_This is the first part in a series on using [fn-fx](https://github.com/fn-fx/fn-fx) for Desktop GUI application development in clojure._  
&nbsp;   
&nbsp;  
fn-fx is a clojure wrapper over JavaFx, described by wikipedia as "a software platform for creating and delivering desktop applications, as well as rich Internet applications (RIAs) that can run across a wide variety of devices".  
Calling fn-fx a wrapper over JavaFx is doing it a disservice. fn-fx provides a declarative and functional interface over JavaFx - data is used to express the GUI elements and any changes to the data trigger changes in the GUI elements.  
Let's start this series with our version of the "hello world" of desktop applications - a button with text and the behaviour associated with the button press.  
<p align="center">
<img src="{{ site.url }}/assets/images/javafx_simple.png">
</p>
Pressing the button changes the text to "Said 'Hello World'" and disables the button for five seconds. After five seconds the button reverts to it's original state and text. Let's go one step beyond the introductory JavaFx tutorial and set an image as the icon on the title bar - notice the globe on the left end of the title bar.  
The full code for this tutorial is [here](https://github.com/anuj-seth/fn-fx-simple).  
&nbsp;  
## The data
Data holds centre stage in any clojure program and our fn-fx application is no different. Our UI consists of a button that can be either enabled or disabled state and has text printed on it corresponding to the current state. We use a clojure map to hold the state of the button and the text associated with each state. The value associated with the key :button-state changes in response to user actions and can be either :enabled or :disabled. The key :button-text only associates the button state with the text to be displayed on the button and never changes.  
```
{:button-state :enabled,
 :button-text {:enabled "Say 'Hello World'"
               :disabled "Said 'Hello World'"}}
```
## The button
We use the defui macro to generate user components. The defui macro
## The main class
We start off by creating a class that will act as the entry point for our application. The sole job of this class is to call a function that does the actual work of initializing our application.
```
(ns fn-fx-simple.main
  (:require [fn-fx-simple.core :as core])
  (:gen-class))

(defn -main
  [& args]
  (core/start-javafx))
```
All JavaFx programs must extend the javafx.application.Application class and define a start method that acts as the entry point. Calling the launch method of the Application class creates the JavaFx application thread, which handles GUI construction and updates.
```
(ns fn-fx-simple.core
  (:gen-class :extends
              javafx.application.Application))

(defn start-javafx
  [& args]
  (javafx.application.Application/launch fn_fx_simple.core
                                         (into-array String args)))

```
&nbsp;  
## The stage




