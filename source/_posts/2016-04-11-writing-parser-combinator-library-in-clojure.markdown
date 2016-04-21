---
layout: post
title: "Writing Parser Combinator Library in Clojure"
date: 2016-04-11 23:23:41 +0400
comments: true
categories: [parser,parser combinator,clojure,lisp] 
---

TL;DR Writing parser combinator library in Clojure from scratch

Recently I've had an fortunate opportunity to participate in a commercial project that was heavy on parsing and I've decided to use [Clojure](http://clojure.org/) as my main tool for doing it.
As you might all know **Clojure** is not new to the game and is quite mature functional programming language with a hidden power of [Lisp](https://en.wikipedia.org/wiki/Lisp_%28programming_language%29). One of the distinct features of **Clojure** which I personally adore is nrepl and [Interactive Programming](https://en.wikipedia.org/wiki/Interactive_programming) incremental development style which it cultivates. This is by no means something new, [Common Lisp](https://en.wikipedia.org/wiki/Common_Lisp) has it via *slime/swank*, some of [Scheme](https://en.wikipedia.org/wiki/Scheme_%28programming_language%29) implementations have *geiser* support and almost all Smalltalk variants could be considered as full interactive programming environments, one notable example being [Pharo](http://pharo.org/). 
There are some more specialized interactive programming environments like [Processing](https://processing.org/) and [SuperCollider](https://supercollider.github.io/) which deal with either sound or visual or even both but today we'll be talking about parsing and all things related instead.
Before I'll start explaining what we are actually trying to build I would like to introduce you to a really nice open source library that I've successfully used in a commercial project I've mentioned before, called [kern](https://github.com/blancas/kern). But as all things good there is always some value in trying to build your own even better, at least for me it was an insightful journey. So yeah as title says we'll be building parser combinator library in **Clojure** just to learn a thing or two but solely just for fun. The full source for impatient ones is in my [ccp](https://github.com/troydm/ccp/) repo.
So let's get started then.

![Violin Loli](http://i.imgur.com/JG55prH.png)

<!--more-->

I'm no good with long explanations and to keep things simple I'll start with two references that I've heavily used while building *ccp*. One is being *kern* as you've guessed it and the second one is [parsec](https://wiki.haskell.org/Parsec). Both are excellent parser combinator libraries and are good references for self-study. Before we've delve into what is parser combinator I would like to talk about a simple concept of a *parser*. Now what is a *parser*? Something that parses a thing and returns result as you've guessed it, right?! Wait, isn't that a function of a -> b. Remarkable similarity, indeed. 
A function takes some arbitrary data *a*, parsers it and returns result *b*. Wait, what if it can't parse data and return result? Then we'll return an error instead of parsing result. In *Haskell* our type definition for parser would be something like this

{% codeblock lang:haskell %}
data Parser a b = Parser (a -> Either String b)
{% endcodeblock %}

As some you of you may have noticed our *Parser* is actually a *ReaderT a Either String b* monad transformer in disguise. As we are parsing *a* we've also need to constantly mutate it as it's being parsed so probably we are mistaken and we actually need an *StateT a Either String b* monad transformer instead, but wait, 
aren't we going to build everything in **Clojure**? Yeah, right so we don't need any strict type definitions. 
We'll just use simple function that takes data as input and returns parsing result and will mutate state of input data as we go. And for errors will have some special *deftype* **ParserError**. Any parser that can't parse data will return **ParserError** with detailed error message description.

{% codeblock lang:clojure %}
(deftype ParserError [ps state msg])

{% endcodeblock %}

What is this *ps* and *state*? Patience my friend I'll explain it later. First let's talk about data we are going to parse. What should it be? Well it can be literally anything but we need 
some contract in order to parse it bit by bit right? So let's introduce a *protocol* called **ParseStream**. Any type that implements our **ParseStream** protocol is good enough to be an input data for our parser. We can go back and forth element by element traversing this data stream using *next-elem*, *prev-elem* and *current-elem* methods. 
Our **ParseStream** implementors need to track state and mutate it as we go over the data with parsers. This is actually a data + state concept. As we are writing everything in Clojure, we'll just use *deftype* with mutable state variable and data. I know about dangers of using mutable variables but as we are only giving limited access to our type via strict protocol there is no way our parser can harm our inner **ParseStream** state in any way, so we are kinda safe. In order to be able to produce a **ParseStream** out of any data we'll introduce another protocol called **ParseStreamFactory**. 
Any type/object that implements this protocol will be convertible into a valid **ParseStream**. As any **ParseStream** tracks state it can reset itself to initial position using *reset* method, and record and restore it's state using *get-state* and *restore-state* methods. We can also check if it's state is initial or ending using *start-state?* and *end-state?* methods. 
Actually our **ParseStream** is just some good old [Iterator](https://en.wikipedia.org/wiki/Iterator) pattern from Object-Orient world. 
We are combining Functional and Object-Oriented programming using **Clojure** in order to produce the most elegant solution to the given problem at hand. This method of programming is quite powerful, best of both worlds indeed. Now you know what *ps* and *state* in **ParserError** is, it's **ParseStream** data and it's *state* when the **ParserError** occurred. Also we would like to have a human readable description of parser error so our **ParseStream** will have *describe-error* method too. We'll also define some helper methods to work with **ParserError**.


{% codeblock lang:clojure %}
(defprotocol ParseStreamFactory
  "A parse stream factory protocol"
    (parse-stream [d]))

(defprotocol ParseStreamCharSequenceFactory
  "A parse stream char sequence factory protocol"
    (char-sequence [ps]))

(defprotocol ParseStream
  "A parse stream protocol used by parser"
    (reset [ps])
    (current-elem [ps])
    (prev-elem [ps])
    (next-elem [ps])
    (get-state [ps])
    (restore-state [ps st])
    (state-changed? [ps st])
    (start-state? [ps])
    (end-state? [ps])
    (describe-error [ps pe]))

(deftype ParserError [ps state msg]
  Object
    (toString [self] (describe-error ps self)))

(defn parser-error [ps msg] (ParserError. ps (get-state ps) msg))
(defn parser-error-stream [pe] (.ps pe))
(defn parser-error-state [pe] (.state pe))
(defn parser-error-message [pe] (.msg pe))
(defn parser-error? [pe] (= (type pe) ParserError))

{% endcodeblock %}

Now let's define String implementation of **ParseStream**. Our **StringParseStream** will go character by character incrementing/decrementing indexed position *p*.
The code is pretty straightforward. We just save state of parse stream into index position starting at *-1* using *unsynchronized-mutable* variable *p*.
We don't need any thread-safety as our parse stream is guaranteed to be used only by one thread. Also in order to describe error we need to correctly inform our parsing library user about exact location in String where that error occurred. For this we use private function *line-col* which counts lines and columns and returns exact line and column number of that index position.

{% codeblock lang:clojure %}

(defn- line-col
 "find out line/column at position p in String s"
 [s p]
 (let [len (.length s)]
  (loop [p p l 1 c 1]
   (if (<= p 0)
    `(~l ~c)
    (if (>= p len)
     `(~l ~(+ p c))
     (if (= (.charAt s p) \newline)
      (recur (- p 1) (+ l 1) 1)
      (recur (- p 1) l (+ c 1))))))))

(deftype StringParseStream [^:unsynchronized-mutable p s]
 ParseStream
 (reset [ps] (set! p -1))
 (current-elem [ps] (if (or (start-state? ps) (end-state? ps)) nil (.charAt s p)))
 (prev-elem [ps] (if (not (start-state? ps))
                  (set! p (- p 1)))
  (current-elem ps))
 (next-elem [ps] (if (not (end-state? ps))
                  (set! p (+ p 1)))
  (current-elem ps))
 (get-state [ps] p)
 (restore-state [ps st] (if (and (>= st -1) (<= st (.length s))) (set! p st)) p)
 (state-changed? [ps st] (not= st p))
 (start-state? [ps] (= -1 p))
 (end-state? [ps] (= p (.length s)))
 (describe-error [ps pe]
  (let [[l c] (line-col s (parser-error-state pe))]
   (str "expecting " (parser-error-message pe) " on line " l " col " c))))

{% endcodeblock %}

Now let's implement the same but for any arbitrary **Clojure** sequence. Note how *describe-error* method differs compared to **StringParseStream** implementation. 
Rest of the code is almost exactly the same except that we are using sequence related functions instead of *String* related ones.

{% codeblock lang:clojure %}

(deftype SequenceParseStream [^:unsynchronized-mutable p s]
 ParseStream
 (reset [ps] (set! p -1))
 (current-elem [ps] (if (or (start-state? ps) (end-state? ps)) nil (nth s p)))
 (prev-elem [ps] (if (not (start-state? ps))
                  (set! p (- p 1)))
  (current-elem ps))
 (next-elem [ps] (if (not (end-state? ps))
                  (set! p (+ p 1)))
  (current-elem ps))
 (get-state [ps] p)
 (restore-state [ps st] (if (and (>= st -1) (<= st (count s))) (set! p st)) p)
 (state-changed? [ps st] (not= st p))
 (start-state? [ps] (= -1 p))
 (end-state? [ps] (= p (count s)))
 (describe-error [ps pe]
  (let [p (parser-error-state pe)]
   (str "expecting " (parser-error-message pe) " on index " p " in sequence, but got: " (nth s p)))))

{% endcodeblock %}

Now in order for most types to be convertible into *ParseStream* we need to extend them to implement **ParseStreamFactory** protocol.

{% codeblock lang:clojure %}

(extend-type String
    ParseStreamFactory
        (parse-stream [d] (StringParseStream. -1 d)))

(extend-type java.util.List
    ParseStreamFactory
        (parse-stream [d] (SequenceParseStream. -1 d)))

(extend-type clojure.lang.PersistentVector
    ParseStreamFactory
        (parse-stream [d] (SequenceParseStream. -1 d)))

(extend-type clojure.lang.PersistentVector$TransientVector
    ParseStreamFactory
        (parse-stream [d] (SequenceParseStream. -1 d)))

(extend-type clojure.lang.PersistentList
    ParseStreamFactory
        (parse-stream [d] (SequenceParseStream. -1 d)))

(extend-type clojure.lang.PersistentList$EmptyList
    ParseStreamFactory
        (parse-stream [d] (SequenceParseStream. -1 d)))

{% endcodeblock %}

Before we'll start writing our sample parsers I would like to introduce an macro that will help us in minimizing our code needed for writing almost any parser.
This macro is called *parsed* and is simply a let + if combination that gives us ability to define a variable *e* initialized to some parsed value *p*, check if this value is 
parser error or not and depending on that return either *t* or *f* expression.

{% codeblock lang:clojure %}

(defmacro parsed [e p t f]
  `(let [~e ~p]
       (if (parser-error? ~e) ~f ~t)))

{% endcodeblock %}

Let's define our first parser generating function called *satisfy*. 
Given a predicate function *pd* and *m* message it will give back a parser that will get next element from **ParseStream**, check if it satisfies the predicate.
If it does it we'll return it, otherwise it will return **ParserError** with message *m*. 
Note how we would like to support two kinds of predicate functions, with 1 argument and with
2 arguments second being actual **ParseStream** itself. To distiguish those two we'll use another private function called *n-args*.

{% codeblock lang:clojure %}

(defn- n-args
 "return number of arguments function can take"
 [f]
 (-> f class .getDeclaredMethods first .getParameterTypes alength))

(defn satisfy
 "satisfy predicate pd, if not return failure with m"
 [pd m]
 (if (= (n-args pd) 2)
  (fn [ps] (let [e (next-elem ps)]
            (if (pd e ps)
             e
             (parser-error ps m))))
  (fn [ps] (let [e (next-elem ps)]
            (if (pd e)
             e
             (parser-error ps m))))))

{% endcodeblock %}

Note how *satisfy* is both suitable for String streams as well as sequence streams. Before we'll define some simple String stream oriented parsers
let's define one more crucial function that is necessary to define our parsers. What this function does is that it takes a parser *p*, 
saves **ParseStream** *ps*'s state into *st* variable, applies *p* to *ps*, checks if result is **ParserError**, and if it is, it restores *ps*'s state
to value it had before *p*'s application. Now simply put this function allows us to convert any parser that might fail into a parser that doesn't affects
**ParseStream** in case of failure and restores it's state to the way it was before parser was applied to it. 
We can also define *restore* parser transformer that will restore stream every time.
This parser transformers are actually called a parser combinator. So what is parser combinator? It's a function that takes as an input
one or many parser functions and returns a parser that is result of combining those parsers. By combining we actually mean that parser combinator introduces an behaviour 
over already defined parser behaviour. This allows us to stack/combine parsers like a lego pieces and it's actually a very very powerful concept. 
In essence this concept is actually the real corner stone of functional programming and it's conceptually heart of power. 

{% codeblock lang:clojure %}

(defn rewind
 "try parsing p and restore stream state on failure"
 [p]
 (fn [ps] (let [st (get-state ps) e (p ps)]
           (if (parser-error? e)
            (do (restore-state ps st) e)
            e))))

(defn restore
 "try parsing p and restore stream state afterwards"
 [p]
 (fn [ps] (let [st (get-state ps) e (p ps)]
           (do (restore-state ps st) e))))


{% endcodeblock %}

Note that we aren't limited to parser(s) as input for a parser combinator as we can have any kind of value that drives resulting behaviour of our parser.
In that essence the function *satisfy* that we've defined before is also an parser combinator. Okey zen reached, let's move on already!
Now that we know what parser combinators actually are let's define some simple parser combinator called *chr* that can give us parser for any specific character.
We also define *satisfy-char* parser combinator that is actually used to be sure that value we are trying to parse is of character type.
It's not strictly necessary for *chr* but we'll define it anyway.

{% codeblock lang:clojure %}

(defn satisfy-char
  "satisfy predicate pd for character, if not return failure with m"
    [pd m]
      (satisfy #(and (char? %) (pd %)) m))

(defn chr
  "expect character c"
    [c]
      (rewind (satisfy-char #(= % c) (str (pr-str c) " character"))))

{% endcodeblock %}

Now using those parser combinators we can define some commonly used parsers

{% codeblock lang:clojure %}

(def digit
  "expect digit character"
    (rewind (satisfy-char #(Character/isDigit %) "digit")))

(def ws
  "expect whitespace character"
    (rewind (satisfy-char #(Character/isWhitespace %) "whitespace")))

(def alpha
  "expect alphanum character"
    (rewind (satisfy-char #(or (Character/isLetter %) (Character/isDigit %)) "alphanum")))

(def letter
  "expect letter character"
    (rewind (satisfy-char #(Character/isLetter %) "letter")))

(def upper
  "expect uppercase character"
    (rewind (satisfy-char #(Character/isUpperCase %) "uppercase")))

(def lower
  "expect lowercase character"
    (rewind (satisfy-char #(Character/isLowerCase %) "lowercase")))

(def any-char
  "expect any character"
    (rewind (satisfy char? "any character")))

{% endcodeblock %}

Let's try using parsers and parser combinators that we've defined so far.
For this we'll use *parse* function that is actually a helper to convert 
input data to **ParseStream** if it isn't already a **ParseStream** and apply *p* to it

{% codeblock lang:clojure %}

(defn parse
 "parse d data using p and return result, resets parse stream"
 [p d]
 (p (if (satisfies? ParseStream d)
     (do (reset d) d)
     (parse-stream d))))

{% endcodeblock %}

Now some simple tests

{% codeblock lang:clojure %}

(parse (chr \a) "a") 
-> \a

(parse (chr \b) "b") 
-> \b

(parse ws " ") 
-> \space

(parse ws "a")
-> #object[ccp.core.ParserError 0x95c543a "expecting whitespace on line 1 col 1"]

{% endcodeblock %}

Sometimes we need our parser to return a value different from the one that has been parsed.
Let's define two parser combinators one called *return* which substitutes a return value with provided one and another 
one called *null* which just returns nil instead of parse result

{% codeblock lang:clojure %}

(defn return
  "try parsing p and on success return r otherwise return parser error"
    [p r]
      (fn [ps] (parsed e (p ps) r e)))

(defn null
  "try parsing p and on success return nil"
    [p]
      (return p nil))

{% endcodeblock %}

What if we want to match beginning or end of **ParseStream**. We can do that since our **ParseStream** protocol has
two methods called *start-state?* and *end-state?*.

{% codeblock lang:clojure %}

(def bos
  "expect beginning of stream"
    (fn [ps] (if (start-state? ps) nil (parser-error ps "beginning of stream"))))

(def eos
  "expect end of stream"
    (rewind (satisfy (fn [_ ps] end-state? ps) "end of stream")))

{% endcodeblock %}

But what if we want to just generally parse any arbitary data, for example while parsing a **SequenceParseStream**.
We can do that too. We can even parse data with some particular elem using *=* and return it as parse result or 
some optionally specified failure message with name of the element we needed our parser to be equal to.

{% codeblock lang:clojure %}

(def any-elem
  "expect any element"
    (fn [ps] (next-elem ps)))

(defn elem
  "expect element e named with name, name is used for parse error"
    [e & {:keys [name]}]
      (rewind (satisfy #(= e %) (if (nil? name) (str e) name))))

{% endcodeblock %}

We can even parse beginning and end of line.

{% codeblock lang:clojure %}

(def bol
 "expect beginning of line, returns nil on success"
 (fn [ps]
  (if (start-state? ps)
   nil
   (let [c (current-elem ps)]
    (if (or (= c \newline) (= c \return))
     nil
     (parser-error ps "beginning of line"))))))

(def eol
 "expect end of line, supports windows, unix and mac newline types"
 (rewind
  (fn [ps]
   (let [e (next-elem ps)]
    (if (or (end-state? ps) (= e \newline))
     \newline
     (if (= e \return)
      (let [st (get-state ps) e (next-elem ps)]
       (if (= e \newline) e (do (restore-state ps st) \newline)))
      (parser-error ps "end of line character")))))))

{% endcodeblock %}

Wait, this all starts remind us about regular expressions, right?! Well ofcourse we can even make a parser that will be able to match some regular expression.
But we have two limitations here, in order for our regular expression parser to be able to work with standard Java regular expression methods  it needs to be String or any char element based **ParseStream** and it needs to implement **CharSequence** interface. 
So let's define **ParseStreamCharSequenceFactory** protocol and implement **StringParseStreamCharSequence** *deftype* to extend **StringParseStream** to be able to convert it to **CharSequence** at any time when needed. This functionality will be used for our *match* parser combinator that will have regular expression as it's input and will return parser that can match that regular expression.
Lots of code, I know. This is probably the most complicated parser combinator in entire *ccp* library. I won't start explaining every bit of it and leave that 
to you guys as a self study.

{% codeblock lang:clojure %}

(defprotocol ParseStreamCharSequenceFactory
  "A parse stream char sequence factory protocol"
    (char-sequence [ps]))

(deftype StringParseStreamCharSequence [ps p l]
 CharSequence
 (charAt [_ i]
  (let [i (+ p i)]
   (if (or (< i 0) (>= i (.length (.s ps))))
    (throw (IndexOutOfBoundsException.))
    (.charAt (.s ps) i))))
 (length [_] l)
 (subSequence [_ s e] (StringParseStreamCharSequence. ps (+ p s) (- e s)))
 (toString [_] (.substring (.s ps) p (+ p l))))

(deftype StringParseStream [^:unsynchronized-mutable p s]
 ....
 ParseStreamCharSequenceFactory
 (char-sequence
  [ps]
  (StringParseStreamCharSequence. ps (get-state ps) (- (.length (.s ps)) (get-state ps)))))

(defn match
 "match regex re, on success returns result of the match"
 [re]
 (let [re (if (string? re) (re-pattern re) re)]
  (rewind (fn [ps]
           (next-elem ps)
           (let [d (re-matches re (char-sequence ps))]
            (if d
             (loop [i (- (.length (if (vector? d) (first d) d)) 1)]
              (if (> i 0) (do (next-elem ps) (recur (- i 1))) d))
             (parser-error ps (str re))))))))

{% endcodeblock %}

Let's define another parser combinator. This one we'll call *collect* and it'll just collect exactly 
*n* number of elements from parse stream and will return vector containing those elements.
Note how we are using [transient](http://clojure.org/reference/transients) vector as accumulator in our loop/recur.
Using transient vector is more effective performance-wise as we don't need full power of persistent data structure and instead just want to accumulate result.

{% codeblock lang:clojure %}

(defn collect
 "collect exactly n elems and return vector containing those elements"
 [n]
 (cond
  (< n 0)
  (throw (IllegalArgumentException. "n can't be negative"))
  (= n 0)
  (fn [ps] [])
  :else
  (rewind (fn [ps]
           (let [nc n]
            (loop [e (next-elem ps) n n acc (transient [])]
             (if (nil? e)
              (parser-error ps (str n " element" (if (> n 1) "s" nil)))
              (if (= n 1)
               (persistent! (conj! acc e))
               (recur (next-elem ps) (- n 1) (conj! acc e))))))))))

{% endcodeblock %}

We can do the same but instead of collecting result just *consume* it and return nil.

{% codeblock lang:clojure %}

(defn consume
 "consume exactly n elems and return nil"
 [n]
 (cond
  (< n 0)
  (throw (IllegalArgumentException. "n can't be negative"))
  (= n 0)
  (fn [ps] nil)
  :else
  (rewind (fn [ps]
           (let [nc n]
            (loop [e (next-elem ps) n n]
             (if (nil? e)
              (parser-error ps (str n " element" (if (> n 1) "s" nil)))
              (if (= n 1)
               nil
               (recur (next-elem ps) (- n 1))))))))))

{% endcodeblock %}

We can even parse exactly some *string*

{% codeblock lang:clojure %}

(defn string
 "parse string s"
 [s]
 (if (empty? s) (throw (IllegalArgumentException. "parsing string can't be empty"))
  (rewind (fn [ps]
           (loop [e (next-elem ps) p 0]
            (if (= (.charAt s p) e)
             (if (= (+ p 1) (.length s))
              s
              (recur (next-elem ps) (+ p 1)))
             (parser-error ps (str "'" (.charAt s p) "' from '" s "' string"))))))))

{% endcodeblock %}

What if we want to parse one or more parsers and return value only of the last one?
Well we can define a parser combinator that will allow us to do that.

{% codeblock lang:clojure %}

(defn >>
 "execute parsers sequentually and return result of the last one"
 [& p]
 (fn [ps]
  (loop [[h & t] p]
   (parsed e (h ps)
    (if (empty? t) e (recur t))
    e))))

{% endcodeblock %}

And exactly same but return first parser result instead.

{% codeblock lang:clojure %}

(defn <<
 "execute parsers sequentually and return result of the first one"
 [& p]
 (fn [ps]
  (parsed first-elem ((first p) ps)
   (loop [[h & t] (rest p)]
    (if (nil? h)
     first-elem
     (parsed e (h ps) (recur t) e)))
   first-elem)))

{% endcodeblock %}

Now let's define some basic yet very powerful monadic bind *>>=* parser combinator.
Note how we apply function to result of the parse result of our parser. We can even have two kinds of 
functions, one with parse stream as second result and one without it. We could have defined our *<<* and *>>*
parser combinators using *>>=* parser combinator but our previously define loop/recur version will work faster and can grow 
infinitely without consuming the stack.

{% codeblock lang:clojure %}

(defn >>=
 "execute parser and return result of applying f to parsed result and optionally a parse stream"
 [p f]
 (if (= (n-args f) 2)
  (fn [ps]
   (parsed e (p ps) (f e ps) e))
  (fn [ps]
   (parsed e (p ps) (f e) e))))

{% endcodeblock %}

This can be used to define a *letp* macro which is actually a parser combinator too in some sense, well not exactly but close.

{% codeblock lang:clojure %}

(defmacro letp
 "binds parser results to variables sequentually, same as let but uses parsers as values"
 [[& parsers] & body]
 (if (or (zero? (count parsers)) (not (zero? (mod (count parsers) 2))))
  (throw (IllegalArgumentException. "invalid number of parsers"))
  (let [[e p & r] parsers]
   (if (nil? r)
    `(>>= ~p (fn [~e ps#] ~@body))
    `(>>= ~p (fn [~e ps#] ((letp ~r ~@body) ps#)))))))

{% endcodeblock %}

This can be used to manipulate parser results in any way and return result any way we want to.

{% codeblock lang:clojure %}

(parse (letp [a (chr \a) b (chr \b)] (str a b)) "ab")
-> "ab"

(parse (letp [a any-char b any-char] [a b]) "ab")
-> [\a \b]

(parse (letp [a digit b digit] (+ (read-string (str a)) (read-string (str b)))) "13")
-> 4

{% endcodeblock %}

How about some sequential parser combinator that collects result into a vector.
This one is kinda similar to *collect* but applies parsers instead of counting number of elements.

{% codeblock lang:clojure %}

(defn <*>
 "execute parsers sequentually and return vector of results, fails if one of them fails"
 [& p]
 (if (empty? p)
  (throw (IllegalArgumentException. "no parsers specified"))
  (fn [ps]
   (loop [[h & t] p acc (transient [])]
    (parsed e (h ps)
     (if (empty? t)
      (persistent! (conj! acc e))
      (recur t (conj! acc e)))
     e)))))

{% endcodeblock %}

We can use *<\*>* to define *<str\*>* and *<keyword\*>* parser combinators.
Those just apply *str* and *keyword* functions to parser results.

{% codeblock lang:clojure %}

(defn <str>
 "parse p and apply str to it's result"
 [p]
 (fn [ps]
  (parsed e (p ps) (apply str e) e)))

(defn <str*>
 "execute parsers sequentially and return string of results, fails if one of them fails"
 [& p]
 (<str> (apply <*> p)))

(defn <keyword>
 "parse p and apply keyword to it's result"
 [p]
 (fn [ps]
  (parsed e (p ps)
   (keyword
    (cond
     (string? e)
     e
     (coll? e)
     (apply str e)
     :else (str e)))
   e)))

(defn <keyword*>
 "execute parsers sequentually and return keyword of results, fails if one of them fails"
 [& p]
 (<keyword> (apply <str*> p)))

(parse (<*> any-char any-char) "ab")
-> [\a \b]

(parse (<str*> any-char any-char) "ab")
-> "ab"

(parse (<keyword*> any-char any-char) "ab")
-> :ab

{% endcodeblock %}

Now let's define yet another crucial alternatives *<|>* parser combinator.
This takes any number of parsers as input, and when parsing, tries each one of them and stops when 
either all of them fail or one of them consumes input while failing. 
The code is quite complicated as we need to deal with generating error message as well as checking
state of parse stream each time we are trying to parse some data.

{% codeblock lang:clojure %}

(defn <|>
 "execute parsers sequentually and return first successful result, fails if one of them fails while consuming input or all of them fail"
 [& p]
 (if (empty? p)
  (throw (IllegalArgumentException. "no parsers specified"))
  (fn [ps]
   (loop [[h & t] p ms (transient [])]
    (let [st (get-state ps) e (h ps)]
     (if (parser-error? e)
      (if (state-changed? ps st)
       e
       (if (empty? t)
        (let [ms (persistent! (conj! ms (parser-error-message e)))
         st (get-state ps)
         _ (next-elem ps)
         pe (parser-error ps (str/join " or " ms))
         _ (restore-state ps st)]
         pe)
        (recur t (conj! ms (parser-error-message e)))))
      e))))))

{% endcodeblock %}

Speaking of parser failures what if we want to get a failure message out of parser without actually applying parser to a parse stream.
Well we have some edge cases but still we can do that with a really simple trick by making a parser combinator that can make any parser *fail*. 
We'll just define some empty string stream and will use it to apply parser to it and return it's error message as if it was applied to parse
stream itself. We can then have *fail-message* function that will return the failure message itself.

{% codeblock lang:clojure %}

(def empty-stream (parse-stream ""))

(defn fail
 "fail p, parser p must not accept nil otherwise failure message won't be correct"
 [p]
 (restore (cond
           (= p bos)
           (fn [ps] (next-elem ps) (parser-error ps "beginning of stream"))
           (= p eos)
           (fn [ps] (next-elem ps) (parser-error ps "end of stream"))
           (= p any-elem)
           (fn [ps] (next-elem ps) (parser-error ps "any element"))
           :else
           (fn [ps]
            (next-elem ps)
            (parsed e (p empty-stream)
             (parser-error ps (str e))
             (parser-error ps (parser-error-message e))))
          )))

(defn fail-message
 "makes p fail and returns failure message, note: not a parser function"
 [p]
 (parser-error-message ((fail p) empty-stream)))

{% endcodeblock %}

Now that we have *fail-message* function we can define *no* parser combinator that will make 
any parser fail in case it succeeds and will succeed in case of failure.

{% codeblock lang:clojure %}

(defn no
 "if p fails succeds with nil, otherwise fails"
 [p]
 (fn [ps]
  (parsed e (p ps) (parser-error ps (str "not " (fail-message p))) nil)))

{% endcodeblock %}

Sometimes we want some optional value so why note define an *option* parser combinator too.

{% codeblock lang:clojure %}

(defn option
 "try p, if it fails without consuming any input return r, otherwise return value returned by p"
 [p r]
 (fn [ps] (let [st (get-state ps) e (p ps)]
           (if (parser-error? e)
            (if (state-changed? ps st) e r)
            e))))

{% endcodeblock %}

What if we want to repetitively parse some value one or more times and return vector of results.
We can do that with *many* and *many1* parser combinators.

{% codeblock lang:clojure %}

(defn many
 "try p, 0 or more times and return result"
 [p]
 (fn [ps]
  (loop [e (p ps) acc (transient [])]
   (if (parser-error? e)
    (persistent! acc)
    (recur (p ps) (conj! acc e))))))

(defn many1
 "try p, 1 or more times and return result"
 [p]
 (fn [ps]
  (let [e (p ps)]
   (if (parser-error? e)
    e
    (loop [acc (transient [e]) e (p ps)]
     (if (parser-error? e)
      (persistent! acc)
      (recur (conj! acc e) (p ps))))))))

{% endcodeblock %}

Our next parser combinator is quite complex so I'll try to explain it in more detail.
Sometimes when parsing programming languages or any symbolic expressions we need our operators to have 
different precedence based on some predefined rules that we have. For example in math expression like 1+2\*3 will
be interpreted as (+ 1 (\* 2 3)) since * operator has higher precedence compared to + operator. 
In order parse this kind of expressions we'll define two special parser combinators called  *op* and *op-expr* that need to be used with each other.
*op* parser combinator converts any parser that can parse some operator into a parser that returns parsed operator and it's precedence value.
This value is then used by *op-expr* in order return expression in exact precedence order as define by op parsers. The code is kinda complex and I'll live it 
as exercise for you to study and understand.

{% codeblock lang:clojure %}

(defn op
 "return operator parser p with precedence o"
 [p o]
 (fn [ps]
  (parsed e (p ps) `(~o ~e) e)))

(defn op-expr
 "parse parser p expression and then optionally parse any of operator parser from ops sequence followed by another p expression and return expressions ordered by operator precedence with f applied to each expression"
 ([p ops] (op-expr p ops identity))
 ([p ops f]
  (let [ops (apply <|> ops)]
   (letfn [(op>= [o1 o2] (>= (first o1) (first o2)))
    (exps [ps]
     (parsed e (p ps)
      (loop [acc (transient [e])]
       (parsed o (ops ps)
        (parsed e (p ps)
         (recur (conj! (conj! acc o) e))
         e)
        (persistent! acc)))
      e))
    (reduce-exps [es]
     (loop [es es acc []]
      (cond
       (>= (count es) 5)
       (let [[a o1 b o2 c & d] es]
        (if (op>= o1 o2)
         (recur (into [(f `(~(second o1) ~a ~b)) o2 c] d) acc)
         (recur (into [b o2 c] d) (into acc [a o1]))))
       (= (count es) 3)
       (let [[a o b] es] (into acc [(f `(~(second o) ~a ~b))]))
       :else (into acc es))))
        (reduce-while [es]
         (loop [es (reduce-exps es)]
          (if (> (count es) 1)
           (recur (reduce-exps es))
           es)))
        ]
        (fn [ps]
         (parsed es (exps ps) (first (reduce-while es)) es))))))

(parse (op-expr any-char [(op (chr \+) 1) (op (chr \*) 2)]) "a+b+c")
-> (\+ (\+ \a \b) \c)

(parse (op-expr any-char [(op (chr \+) 1) (op (chr \*) 2)]) "a+b*c")
-> (\+ \a (\* \b \c))

(parse (op-expr any-char [(op (chr \+) 1) (op (chr \*) 2)]) "a+d*c+e")
-> (\+ \a (\+ (\* \d \c) \e))

{% endcodeblock %}

I can go on with defining this parsers but this will make me only tired and this post only longer and hard to read so instead I'll just stop and 
show you guys something really awesome. This is *exp* parser combinator that can generate parsers from a simple Parser Expression Language inspired
by [Parsing Expression Grammar](https://en.wikipedia.org/wiki/Parsing_expression_grammar) and Regular Expressions. I won't describe it as it's 
still work-in-progress and might change in the future. Instead I'll show you some examples of using it. 
Those interested in source code  should look in git repo exp.clj file also note that what you might see there is quite ugly and complex but I hope to improve
it in the future.

{% codeblock lang:clojure %}

(parse (exp "abc") "abc")
-> [\a \b \c]

(parse (exp "'abc'") "abc")
-> "abc"

(parse (exp "a?") "b")
-> nil

(parse (exp "a*") "aaa")
-> [\a \a \a]

(parse (exp "a{2,3}") "aaaa")
-> [\a \a \a]

(parse (exp "a?b") "b")
-> [nil \b]

(parse (exp "~a?b") "b")
-> \b

(parse (exp "~a?bc") "bc")
-> [\b \c]

(parse (exp "/[abc]*/") "abc")
-> "abc"

(parse (exp "(abc)") "abc")
-> [\a \b \c]

(parse (exp "(abc)<0>") "abcabc")
-> [[\a \b \c] [\a \b \c]]

(parse (exp "\"(abc)") "abc")
-> "abc"

(parse (exp ":(abc)") "abc")
-> :abc

(parse (exp "[ab]*") "abbaaba")
-> [\a \b \b \a \a \b \a]

(parse (exp "___") "abc")
-> [nil nil nil]

(parse (exp "__\\_") "ab_")
-> [nil nil \_]

(parse (exp "\\s") " ")
-> \space

(parse (exp "\\w") "a")
-> \a

(parse (exp "\\d") "9")
-> \9

(parse (exp "(a(bc))") "abc")
-> [\a [\b \c]]

(parse (exp ":#(a(bc))") "abc")
-> :abc

(parse (exp "\\w\\a[a\\_]") "ab_")
-> [\a \b \_]

(parse (exp "([ab]{2})<a>" {:a (chr \a)}) "abab")
-> [[\a \b] \a]

(parse (exp "([ab]{2})<a>" {:a "ab"}) "baab")
-> [[\b \a] [\a \b]]

{% endcodeblock %}

That's it folks, we've defined a clean, simple and yet beautiful parser combinator library that is easy to grasp. Happy parsing!

![Piano & Violin](http://i.imgur.com/fgIohxm.png)
