---
layout: post
title: "JavaFx and Clojure"
date: 2018-11-22
category: clojure
tags: clojure javafx gui
style: |
  body {
  font-size: 15px;
  }
---
I want to make an application to plot real time current/voltage readings - a homegrown oscilloscope with the data rendered on my computer.  
With that end result in mind, this series of blog posts will explore GUI desktop application development on the JVM.  
The prevailing wisdom in software circles holds that I should use Electron to build my application but I choose to take the simpler path of staying within the clojure eco-system. I also want to explore finite state machines and core.async and see if these ideas enable a cleaner GUI codebase.  
Hence the choice of JavaFx, the leading GUI application development framework on the JVM which also has a clojure wrapper in [fn-fx](https://github.com/fn-fx/fn-fx).  
Even though it is tempting to start right away on my oscilloscope project I will take some time to explore GUI development as I do not have experience in the area. In this blog post I will create a simple hello world GUI application using java interop, converting it to fn-fx in the next post. The application has only one button, clicking on which prints a message to the console.  
The full code listing for this post is [here](https://github.com/anuj-seth/javafx-simple).  

All JavaFx applications must extend the javafx.application.Application class and call the launch method of that class. Even though there are two launch methods - one that takes the application class and another that does not - in my clojure code only the first one worked, the second one throwing an error that my class javafx_simple.core was not a subclass of Application.
```
(defn -main
  "A direct translation of a javafx sample program to clojure"
  [& args]
  (javafx.application.Application/launch javafx_simple.core
                                         (into-array String args)))
```

The Application/start method is the entry point for any JavaFx application and holds all our code in this simple example.  
```
(defn -start
  [app stage]
  (let [btn-event-handler (proxy [EventHandler] []
                            (handle [event] (println "Hello World!")))
        btn (doto (Button.)
              (.setText "Say 'Hello World'")
              (.setOnAction btn-event-handler))
        root (StackPane.)
        scene (Scene. root 300 250)]
    (do
      (.add (.getChildren root) btn)
      (doto stage
        (-> .getIcons
            (.add (Image. (io/input-stream
                           (io/resource "icon.png")))))
        (.setTitle "Hello World !")
        (.setScene scene)
        (.show)))))
```
This is the standard JavaFx "hello world" example found on the official documentation page. There is one addition though, an icon in the title bar which is a picture of the globe.
