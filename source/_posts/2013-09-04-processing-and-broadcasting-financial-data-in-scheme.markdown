---
layout: post
title: "Processing &amp; Broadcasting Financial Data in Scheme"
date: 2013-09-04 21:44
comments: true
categories: [scheme, lisp, server, client] 
---
Any software developer who worked in financial industry will tell you that there are 
few key requirements to programming applications for real time market. Applications should be as fast as possible and 
they should be as easily modifiable as possible. First requirement is essential since getting and processing information
takes time and sending processed information takes even additional precious time, and in financial world time equals money.
Second requirement is determined by constantly changing business rules imposed on data processing. 

![(eq? 'money 'power)](http://i.imgur.com/OErpvNu.png)

Correctly choosing programming language for such applications is key to success. Since first requirement already suggests using languages 
that produce native binary executables we might start thinking to use popular languages such as **C**, **C++**. However developing networked applications
in **C** or **C++** takes more time than in languages such as **Java** or **C#**, and applications developed aren't as easy modifiable as it may seem, so these languages
don't comply with our second requirement. What alternatives do we have? Since it's financial world and we want our applications to be absolutely correct, we might 
want to choose functional programming language and benefit of their advantages. So we have a choice between popular functional programming languages 
**Haskell**, **OCaml** and **Scheme**. **OCaml** is well known to 
be used by a big financial company [Janestreet](http://janestreet.com/) and is a really good choice, it has **ML** like syntax which is really easy to get used to. 
**Haskell** is extremely popular however it's laziness implies some overcomplicated
programming and some performance penalties we don't want to have. **Scheme** is very simple language with **Lisp**-like syntax that is very easy to start programming with and it's programs 
are very easily modifiable. **Scheme** has many implementations that have it's own libraries and it's own pros and cons. **Scheme**'s only downside is that it's not statically typed so it doesn't catches some obvious
compile-time errors, but as long as our application works fine everything is ok! Further on let's consider that we've chosen **Scheme** as our language. Other languages have it's own benefits but for our
needs it's the most suited choice, since we want to start developing as fast as possible. Since we want to produce native binary executables we'll limit our choice
on [Chicken Scheme](http://www.call-cc.org/) which produces very efficient binary applications and is very popular.

So let's consider our hypothetical but very common situation where we want to quickly develop an application that gets a stream of some data (let's imagine it's a real time financial market data) 
that we can get by connecting to some arbitrary tcp host:port. The data that we'll get will be parsed, processed and will be broadcasted to all clients that will connect to our small application 
using tcp protocol. So we need to develop some small tcp client and broadcasting server black box.

![black box](http://i.imgur.com/T4gMsTT.png)

[Chicken Scheme](http://www.call-cc.org/) has a system called [Eggs](http://wiki.call-cc.org/chicken-projects/egg-index-4.html) which is a common way to distribute community extensions.
We'll be using two extensions to ease our task of developing our application. One is called [synch](http://wiki.call-cc.org/eggref/4/synch) which is a set of some useful functions to
work with concurrent code in [Chicken Scheme](http://www.call-cc.org/). The other one we'll be using is called [mailbox-threads](http://wiki.call-cc.org/eggref/4/mailbox-threads) which 
is a convenient **Erlang**-like message passing framework. So let's install those extension by typing following in our console:

{% codeblock lang:bash %}
chicken-install synch
chicken-install mailbox-threads
{% endcodeblock %}

So let's start writing our application. Create a file called *myapp.scm*. First thing we need to do is load our extensions.
Also we'll be using some functions from [srfi-18](http://srfi.schemers.org/srfi-18/srfi-18.html) and [srfi-1](http://srfi.schemers.org/srfi-1/srfi-1.html) 
for multi threading and some convenient list data manipulations.  

{% codeblock lang:scheme %}
(require-extension tcp posix synch mailbox-threads)
(require-extension (only srfi-18 make-mutex make-thread thread-start!))
(require-extension (only srfi-1 delete))
{% endcodeblock %}


First we need a way to get our data and process it. We'll be using functions *tcp-connect* and *read-line* to connect to our host and read data line by line.
Also we'll define some arbitrary processing function that will not do any processing and just broadcast data. I don't want to talk about processing in this post but rather focus 
on getting data and distributing it, so in the end we'll have just a simple tcp duplicator example in **Scheme**. But you can do any kind of data processing by modifying 
*process-data* function.

{% codeblock lang:scheme %}
(define connect-to-host "host")
(define connect-to-port 1234)

(define (start-data-feeder)
    (handle-exceptions ex 
            (begin (print "error occurred while connecting, trying to reconnect") 
            (sleep 1))
        (let ((in (tcp-connect connect-to-host connect-to-port)))
            (get-data in)))
    (start-data-feeder))

(define (get-data in)
    (process-data (read-line in))
    (get-data in))

(define (process-data data)
    (broadcast data))

{% endcodeblock %}

So we have our data gathering and processing ready, now we need to start a tcp server and broadcast data to all clients.
We'll be using *mailboxes* list that will contain each client's mailbox and we'll have a separate mutex to make sure operations 
on *mailboxes* list are thread-safe.

{% codeblock lang:scheme %}
(define mtx (make-mutex))
(define mailboxes '())

(define (broadcast data)
        (synch mtx (map (lambda (mailbox) (thread-send mailbox data)) mailboxes)))
{% endcodeblock %}

So let's write our tcp broadcasting server. Each connected client will have a his mailbox added to *mailboxes* list. 
When client will disconnect, his mailbox will be deleted from *mailboxes* list. We'll use *tcp-listen* and *tcp-accept* functions to 
listen on port and accept connections on listener. Each client will be processed by separate thread.

{% codeblock lang:scheme %}
(define broadcast-on-port 4321)

(define (start-broadcasting)
    (accept-client (tcp-listen broadcast-on-port)))

(define (accept-client listener)
    (let-values ([(in out) (tcp-accept listener)])
        (thread-start! (make-thread
            (lambda ()
                (handle-exceptions ex 
                        (synch mtx (set! mailboxes (delete (current-thread) mailboxes)))
                    (synch mtx (set! mailboxes (cons (current-thread) mailboxes)))
                    (process-client in out))
                (close-output-port out)
                (close-input-port in)))))
    (accept-client listener))

(define (process-client in out)
    (write-line (thread-receive) out)
    (process-client in out))

{% endcodeblock %}

Now only thing left is to start our broadcasting server in separate thread and start feeding it data

{% codeblock lang:scheme %}
(thread-start! (make-thread start-broadcasting))
(start-data-feeder)
{% endcodeblock %}

Pretty neat and simple, isn't it :)

