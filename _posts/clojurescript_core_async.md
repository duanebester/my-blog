title: ClojureScript Core.Async

## ClojureScript Core.Async
### *Featuring WebSockets*

I got it in my mind to build **[Cloj](http://github.com/duanebester/cloj)** a lightweight web application written entirely in [Clojure](http://clojure.org), and built on-top of [http-kit](http://http-kit.org/). Things have been smooth sailing until getting into ClojureScript, I hadn't realized there was so much of a difference to Clojure. 

#### Taking a step back
The First thing I did was implement Ajax with ClojureScript, getting a Json response back, and then updating the dom with some help from [dommy](https://github.com/Prismatic/dommy). I highly recommend dommy, you get ClojureScript side templating and DOM manipulation. Better than just DOM manipulation with domina. I also used [cljs-ajax](https://github.com/yogthos/cljs-ajax) which I'll eventually do away with, it's a great place to see how to wrap the google clojure ajax stuff.

##### Client Ajax ClojureScript:

	(ns cloj.cljs.script
	(:require [ajax.core :refer [GET POST]]
		[dommy.utils :as utils]
      	[dommy.core :as dommy]
		[cloj.js.util :refer [log]]
	(:use-macros
    	[dommy.macros :only [node sel sel1]])
	(:require-macros
		[cljs.core.async.macros :refer [go]]))
	
	(defn receive [event]
	    (let [resp (js->clj event)]
        (dommy/append! (sel1 :#foo) [:li (get-in resp ["test"])])
        (set! (.-scrollTop (sel1 :#foo)) (.-scrollHeight (sel1 :#foo)))))

	(defn error-handler [event]
	    (log (str "Something went wrong: " event)))
 
	(defn ping-server [e]
	    (GET "/api" {:handler receive :error-handler error-handler})
	    (.preventDefault e))
 
	(defn ^:export init []
	    (set! (.-onclick (sel1 :#ping)) ping-server))

On init, I bind the __ping-server__ function to the button's (whose id is __ping__) on-click event. The __ping-server__ function then makes an Ajax request to _localhost:1337/api_ and returns it's result to either the __receive__ function, or the __error-hanlder__ function. In the __receive__ function, I use the handy __js->clj__ to convert from json to a clojure object. I can then extract values using (get-in response ["key"]).

The [:li (some-string-here)] becomes, thanks to dommy, an html node;
	<li>some-string-here</li>

All of the __scrollTop__ and __scrollHeight__ stuff is to make the div I'm appending the content too, always scroll to the lastest appended item.

#### Introducing WebSockets

This is mostly taken from [ddellacosta's project](https://github.com/ddellacosta/cljs-core-async-chat) and was essential in finding what I wanted, and then adding it to my project. But, I promise, I still learned some things…

It's always useful for me when I can see what others have in their header;

	(ns cloj.cljs.sockets
	(:require
		[cljs.core.async :refer [chan <! >! put!]]
		[cljs.reader :as reader]
    	[dommy.utils :as utils]
    	[dommy.core :as dommy])
	(:use-macros
    	[dommy.macros :only [node sel sel1]])
	(:require-macros
    	[cljs.core.async.macros :refer [go]]))

First, create send and receive functions;

	(def send (chan))
	(def receive (chan))
	
Setting up the WebSocket;

	(def ws-url "ws://localhost:1337/async")
	(def ws (new js/WebSocket ws-url))
	
Now, for an event channel, where __el__ is the dom element, and __type__ is the event type;

	(defn event-chan
		[c el type]
		(let [writer #(put! c %)]
    	(dommy/listen! el type writer)
    	{:chan c
     	 :unsubscribe #(dommy/unlisten! el type writer)}))

Now we set up functions that will handle the sending and receiving. Here we send a String "Test" to the server whenever our __make-sender__ function is called. Notice we pass an element with id __websocket__ to an __event-chan__ function, with type of __:click__:

	(defn make-sender []
		(event-chan send (sel1 :#websocket) :click)
		(go
		(while true
     		(let [evt  (<! send)]
       			(when (= (.-type evt) "click")
         		(.send ws "Test"))))))
         		
You can also do slick things like this:

	(.send ws {:name "Name" :last "Last"})

Different event types can send different things, of course.

We create our receiver function, let's call it __make-receiver__;

	(defn make-receiver []
		(set! (.-onmessage ws) (fn [msg] (put! receive msg)))
		(add-message))

Notice this calls an __add-message__ function, which takes the message, grabs the raw data, reads it in, and then appends to the html element with id __foo__:

	(defn add-message []
		(go
		(while true
     	(let [msg            (<! receive)
              raw-data       (.-data msg)
              data           (reader/read-string raw-data)]
       		 (dommy/append! (sel1 :#foo) [:li (str data)])
       		 (set! (.-scrollTop (sel1 :#foo)) (.-scrollHeight (sel1 :#foo)))))))
       		 
Finally, for when the page loads, we set up the functions;

	(defn init! []
		(make-sender)
		(make-receiver))

	(defn ^:export init []
		(set! (.-onload js/window) init!))
		
##### Server side handling

Gotta love http-kit;

	(:use [compojure.route :only [files resources not-found]]
		  [compojure.handler :only [site]] ; 
          [compojure.core :only [defroutes GET POST DELETE ANY context]]
          [org.httpkit.server])

Using compojure for routing;

	(defroutes all-routes
		(GET "/async" [] async-handler)
		(resources "/js/" {:root "js"}))

It calls __async-handler__ which just echos back;

	(defn async-handler [request]
    	(with-channel request channel
    	(info "/async") ;; log.info 
    	(on-close channel (fn [status] (println "channel closed: " status)))
    	(on-receive channel (fn [data] ;; echo it back
                          (send! channel data)
                          (info "Sent WebSocket Data.")))))
                          
### [Cloj Source](http://github.com/duanebester/cloj)

[title: ClojureScript Core.Async]: /