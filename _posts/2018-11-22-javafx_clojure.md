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
The prevailing wisdom in software circles holds that I should use Electron to build my application but I choose to take the simpler path of staying within the clojure eco-system. I also want to explore finite state machines and core.async and see if these ideas enable a cleaner GUI codebase.  
Hence the choice of JavaFx, the leading GUI application development framework on the JVM which also has a clojure wrapper in [fn-fx](https://github.com/fn-fx/fn-fx).  
With the end result of creating a real time data redering desktop application in mind, this series of blog posts will explore JavaFx and GUI desktop application development in clojure.  

Even though it is tempting to start right away on my oscilloscope project I will begin with something simple. In this blog post I will create a simple hello world GUI application using java interop, converting it to fn-fx in the next post.  
This what the standard "hello world" example found on the JavaFx official documentation page looks like. There is one addition though - an icon in the title bar which is a picture of the globe.  

![javafx-simple]({{ site.url }}/assets/images/javafx_simple.png)

The full code listing for this post is [here](https://github.com/anuj-seth/javafx-simple).  

All JavaFx applications must extend the javafx.application.Application class and call the launch method of that class. The Application class has two launch methods - one takes the class containing "main" as the first argument while the other does not. From clojure I had to use first one, the second one throwing an error that my class javafx_simple.core was not a subclass of Application.
```
(defn -main
  "A direct translation of a javafx sample program to clojure"
  [& args]
  (javafx.application.Application/launch javafx_simple.core
                                         (into-array String args)))
```

The Application/start method is the entry point for any JavaFx application and holds all the code in this simple example.  
Each JavaFx application must create a Scene and insert nodes into it to hold the GUI elements. Here I create an instance of StackPane as the "root" node to hold our button. Notice how the root node is sent to the constructor of the Scene. The button is added as a child element of the root node and finally all the elements are set on the Stage.
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
