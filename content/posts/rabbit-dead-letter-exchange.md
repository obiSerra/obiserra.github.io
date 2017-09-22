---
title: "Dead Letter Exchange in Clojure"
date: 2017-09-21T21:28:13+02:00
draft: false
---
## The Problem

In one of the project that I'm working on, we have several microservices that talk each other via RabbitMq messages.
When the microservice A gets a message from the service B, it creates a new db record; when A gets a message from service C, it will look for an existing record and updates it. It is possible though that the message from C gets to A before the message from B, discarding the update or throwing an exception. 

This kind of scenario is quite common, because every microservice is indipendent from each other and RabbitMq messages are asychronous.

To handle this kind of problem using RabbitMq we can setup a Dead Letter Exchange that will be used to wait and retry x number of time with the delivery and the processing of the message. 

## How it works

We will setup a Dead Letter Exchange where RabbitMq will automatically move all messages from a queue that has been "dead lettered". This exchange will have a time to live that will ensure that after x milliseconds, the message will be moved again in original exchange with the original routing key. After a certain amount of attempts a custom handler will stop marking the message as "dead lettered" and publish it to a an error queue.

## The Code

### Original exchange and queue

``` clojure

(let [durable true exclusive false auto-delete false]
    (le/declare ch ex "topic" {:durable durable})
    (lq/declare ch queue-name {:durable durable 
                               :exclusive exclusive 
                               :auto-delete auto-delete 
                               :routing-key routing-key 
                               :arguments {"x-dead-letter-exchange" 
                                           "deadLetterX"}})
    (lq/bind ch queue-name ex {:routing-key routing-key}))

```

Please note that you need to pass as arguments `x-dead-letter-exchange` and the name of the exchange you want to use, in our example "deadLetterX"

### Dead letter exchange and retry queue

``` clojure
(let [durable true exclusive false auto-delete false]
 (le/declare ch "deadLetterX" "topic" {:durable durable})
 (lq/declare ch "retry_queue" {:durable durable 
                               :exclusive exclusive 
                               :auto-delete auto-delete 
                               :routing-key "#"
                               :arguments {"x-dead-letter-exchange" ex 
                                           "x-message-ttl" 30000)}})
    (lq/bind ch "retry_queue" "deadLetterX" {:routing-key "#"}))

```

When we are creating the dead letter exchange, we want to pas as arguments `x-dead-letter-exchange` with the original exchange name; in addition we are also using `x-message-ttl` with a value of 30000 milliseconds to ensure that after 30 seconds, the message is moved to the original queue again.
By setting a `routing-key` equal to "#", the `retry_queue` will catch all messages in the dead letter exchange.


### Dead lettering a message

When you catch an exception throwed by the message handler you want to check how many times the message has been re-queued on the original queue.

``` clojure

(if (< (get-in metadata [:headers "x-death" 0 "count"] 0) 10)
    (lb/reject ch (:delivery-tag metadata {}) false)
    (handle-failure payload)
)

```

Inside the metadata headers we can find a counter that RabbitMq increses each time a message get rejected; after a fixed amout of retry, we will stop to move the message to his original queue and pass the payload to an `handler-failure`.

## All the code together 

~~~clojure

(defn bind-dead-letter
  "Bind a listener to a exchange"
  [connect-data routing-key ex queue-name handler-fn]
  (try
    (let [conn  (if (nil? connect-data) 
                  (rmq/connect) 
                  (rmq/connect connect-data))
          ch (lch/open conn)
          _handler (fn [ch metadata ^bytes payload]
                     (try
                       (let [response (-> (String. payload "UTF-8")
                                          (json/read-str :key-fn keyword)
                                          handler-fn
                                          )]
                         (lb/ack ch (:delivery-tag metadata {}))
                         response
                         )
                       (catch Exception e
                         (if (< (get-in metadata [:headers "x-death" 0 "count"] 0) 10)
                             (lb/reject ch (:delivery-tag metadata {}) false)
                             (handle-failure payload)
                         ))))]
                         
                         
    (let [durable true exclusive false auto-delete false]
        (le/declare ch ex "topic" {:durable durable})
        (lq/declare ch queue-name {:durable durable 
                                   :exclusive exclusive 
                                   :auto-delete auto-delete 
                                   :routing-key routing-key 
                                   :arguments {"x-dead-letter-exchange" "deadLetterX"}})
                                   
        (lq/bind ch queue-name ex {:routing-key routing-key})

        (le/declare ch "deadLetterX" "topic" {:durable durable})
        (lq/declare ch "retry_queue" {:durable durable 
                                      :exclusive exclusive 
                                      :auto-delete auto-delete 
                                      :routing-key "#"
                                      :arguments {"x-dead-letter-exchange" ex 
                                                  "x-message-ttl" 30000)}})
        (lq/bind ch "retry_queue" "deadLetterX" {:routing-key "#"}))


      (let [consumer-tag (lc/subscribe ch queue-name _handler)]
        (fn []
          (do
            (log/info "cancel consumer")
            (lb/cancel ch consumer-tag)
            (log/info "close channel")
            (rmq/close ch)
            (log/info "close connection")
            (rmq/close conn)
            )
          )
        )
      )
    (catch Exception e
      (log/error e)
      )))
      
~~~

## Conclusion

By using RabbitMq's Dead Letter Exchange we can deal with race contions and other kind of errors and failures in a simple and reliable way.
