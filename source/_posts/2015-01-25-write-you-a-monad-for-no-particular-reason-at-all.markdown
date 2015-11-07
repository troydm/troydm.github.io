---
layout: post
title: "Write you a Monad for no particular reason at all!"
date: 2015-01-25 19:01:12 +0400
comments: true
categories: [java, haskell, monad, fp]
---

TL;DR *Writing functional programming library in Java*

[Functional Programming](http://en.wikipedia.org/wiki/Functional_programming) paradigm and languages that fully utilize it's benefits are slowly gaining popularity nowadays not only in academia but also in enterprise becoming de facto standard for future generations of programmers. Maybe in 10 years from now humanity will come up with a better idea of how to make bug-free quality software that does the job and takes less effort to produce, however currently we are living in a century of *Object Oriented* programming systems slowly morphing into *Functional Programming* systems. There are a lot of learning resources on internet about [Function Programming](https://www.quora.com/What-are-some-good-resources-for-learning-about-functional-programming-Why) in general and Functional Languages such as [Haskell](https://www.haskell.org/), [Erlang](http://www.erlang.org/), [Scala](http://www.scala-lang.org/), [Clojure](http://clojure.org/) and many other. You all probably heard about [Learn you a Haskell for Great Good](http://learnyouahaskell.com/) book, a really bright, colorful and interesting book that got me into Haskell programming few years ago. Some of you may even tried reading it. I think most of people without prior functional programming experience that start reading it sooner or later hit a bottom of their mental capacity while trying to comprehend *Monads*. This article is intended for this people! Yes this post is all about *Functors* and *Monads* you heard it right! In order to explore the ideas of functional programming let's create a small functional programming library in **Java** that will have *Functors* and *Monads* and by doing so learn about functional programming from a really unusual perspective of using Object Oriented programming. Our target compiler will be **Java 6** (yes we won't be using new fancy **Java 8**) as most of the people that are reading this now are probably more familiar with it rather than newer version. For those of you who are too lazy to follow can check the whole thing at my [fpj](https://github.com/troydm/fpj) repository, however I strongly encourage you to try reimplement it all by yourself from scratch. The information described in this post implies that you have prior experience or basic understand of *Monads* but want to better understand them internally. So let's get to work!

![Rikka loli](http://i.imgur.com/xT0ABh8.gif)

<!-- more -->

First of all let's start with key concept of functional programming, a _function_. A _function_ is something that takes **A** and produces **B** by a process called application of _function_. Let's implement it in **Java** using generic interface. Basically any object that implements that interface can be considered a valid function.

{% codeblock lang:java %}
public interface Function<A,B> {

    B apply(final A a);

}
{% endcodeblock %}

A predicate is a function that takes **A** and produces boolean value.

{% codeblock lang:java %}
public interface Predicate<A> extends Function<A,Boolean> {

}
{% endcodeblock %}

Wait what if we don't need to produce any result or have any argument what we should do then? A Unit is an object that represents exactly this or simply nothing.

{% codeblock lang:java %}
public class Unit {
    
    private static final Unit unit = new Unit();

    public static Unit unit(){
        return unit;
    }

}
{% endcodeblock %}

What if we need to take more than one object and produce more too? That's what a *Tuple* is for. You can even use *Tuple* to combine more than two objects by putting a *Tuple* inside another *Tuple*.

{% codeblock lang:java %}
public class Tuple<A,B> {
    
    private A a;
    private B b;

    public static <A,B> Tuple<A,B> tuple(final A a, final B b){
        if(a == null)
            throw new IllegalArgumentException("first argument has no value");
        if(b == null)
            throw new IllegalArgumentException("second argument has no value");
        return new Tuple<A,B>(a,b);
    }

    private Tuple(final A a, final B b){
        this.a = a;
        this.b = b;
    }

    public A first(){
        return this.a;
    }

    public A getFirst(){
        return this.a;
    }

    public B second(){
        return this.b;
    }

    public B getSecond(){
        return this.b;
    }

}
{% endcodeblock %}

Now let's discover a way of how to use functions. Let's define a helper class called *Functional* and put some static methods into it. The functions we are going to write are so simple and common that any functional programmer can't live without them. First of all what can we do with functions? Yes definitely we can combine a function with another function so basically if we have a function that takes **A** and produces **B**, and we have a function that takes **B** and produces **C** we can combine this two into a function that has a following type description **A** *->* **C** (following from here we'll be using a short notion of **A** *->* **B** to describe a function that takes **A** and produces **B**).


{% codeblock lang:java %}
    public static <A,B,C> Function<A,C> compose(final Function<A,B> f, final Function<B,C> g){
        return new Function<A,C>(){
            public C apply(final A a){
                return g.apply(f.apply(a));
            }
        };
    }
{% endcodeblock %}

What else can we do with functions? Yes, exactly we can map them over a *Collection* of objects and produce a *Collection* of results.
So if we have a function **A** *->* **B** and a *Collection* of **A** objects we can map it over that *Collection* to produce resulting *Collection* of **B** objects.

{% codeblock lang:java %}
    public static <A,B> Collection<B> map(final Function<A,B> f, final Collection<A> ac){
        return Functional.map(f, ac, new ArrayList<B>(ac.size()));
    }

    public static <A,B> Collection<B> map(final Function<A,B> f, final Collection<A> ac, final Collection<B> bc){
        for(A a : ac){
            bc.add(f.apply(a));
        }
        return bc;
    }
{% endcodeblock %}

If we don't need results we can just do foreach over this *Collection* same way, throwing away function result each time we apply the function.

{% codeblock lang:java %}
    public static <A,B> void foreach(final Function<A,B> f, final Collection<A> ac){
        for(A a : ac){
            f.apply(a);
        }
    }
{% endcodeblock %}

We can also fold a value over collection using a function. So basically if we have initial value of **A**, *Collection* of **B** and function that takes **A** *->* **B** *->* **A** we can fold it over by
applying the function to an initial value and first object in *Collection* and then repeating the process with the result of this application with the rest of the *Collection* objects.
In functional programming this is usually called left fold.

{% codeblock lang:java %}
    public static <A,B> A foldLeft(final Function2<A,B,A> f, final A a, final Collection<B> bc){
        A acc = a;
        for(B b : bc){
            acc = f.apply(acc,b);
        }
        return acc;
    }
{% endcodeblock %}

Now let's explore another way of making a function that takes several arguments and produces a result.
We can use *Tuple* for this however sometimes it's easier to define it more simpler like this.

{% codeblock lang:java %}
public interface Function2<A,B,C> {

     apply(final A a, final B b);

}
{% endcodeblock %}

This definition is interchangeable with definition of *Function&lt;Tuple&lt;A,B&gt;,C&gt;* so we can make a helper method in *Functional* class that can take a function that takes a *Tuple* and produce *Function2* if needed and vice versa.

{% codeblock lang:java %}
    public static <A,B,C> Function<Tuple<A,B>,C> toFunction(final Function2<A,B,C> f){
        return new Function<Tuple<A,B>,C>(){
            public C apply(final Tuple<A,B> t){
                return f.apply(t.first(),t.second());
            }
        };
    }

    public static <A,B,C> Function2<A,B,C> toFunction2(final Function<Tuple<A,B>,C> f){
        return new Function2<A,B,C>(){
            public C apply(final A a, final B b){
                return f.apply(Tuple.tuple(a,b));
            }
        };
    }
{% endcodeblock %}

We can then define a helper to flip the arguments of this function if needed

{% codeblock lang:java %}
    public static <A,B,C> Function2<B,A,C> flip(final Function2<A,B,C> f){
        return new Function2<B,A,C>(){
            public C apply(final B b, final A a){
                return f.apply(a,b);
            }
        };
    }
{% endcodeblock %}

We can partially apply an argument to a *Function2* of type **A** *->* **B** *->* **C** transforming it into *Function* of type **B** *->* **C**.

{% codeblock lang:java %}
    public static <A,B,C> Function<B,C> partialApply(final Function2<A,B,C> f, final A a){
        return new Function<B,C>(){
            public C apply(B b){
                return f.apply(a,b);
            }
        };
    }
{% endcodeblock %}

We can also delay the application of an argument to a function by creating another function that takes *Unit* instead of the argument. This concept is usually called lazy function.

{% codeblock lang:java %}
    public static <A,B> Function<Unit,B> lazyApply(final Function<A,B> f, final A a){
        return new Function<Unit,B>(){
            public B apply(Unit unit){
                return f.apply(a);
            }
        };
    }
{% endcodeblock %}

And finally we can do the same folding we did previously only starting from the last element of the *Collection* instead of the first. This is called folding from the right or simply right fold. We can make use of previously defined *foldLeft* method and *Java*'s *ListIterator* and *flip* helper method we defined earlier to do that easily.

{% codeblock lang:java %}
    public static <A,B> B foldRight(final Function2<A,B,B> f, final Collection<A> ac, B b){
        if(ac.size() == 0)
            return b;
        List<A> reversed = new ArrayList<A>(ac);
        Collections.reverse(reversed);
        return Functional.foldLeft(Functional.flip(f), b, reversed);
    }
{% endcodeblock %}

Now all this is cool and interesting but wait didn't we said at the beginning that this post is about Functors and Monads. Sure we did so let's start exploring these concepts!
First of all what is a *Functor*? A *Functor*&lt;A&gt; is some object that contains **A** and you can apply a function of type definition **A** *->* **B** to. This transforms it into another *Functor*&lt;B&gt; that contains result of that application, **B** value. For example we can consider a *Collection*&lt;A&gt; to be a *Functor*&lt;A&gt; because we can apply a function of **A** *->* **B** over all objects of that *Collection* and have a *Collection*&lt;B&gt; of results of that application. Simple? Yeah well not really that simple as it may sound however you'll understand it eventually I hope. 

{% codeblock lang:java %}
public interface Functor<A> {

    <B> Functor<B> fmap(final Function<A,B> f);

}
{% endcodeblock %}

But what about *Monad*? Well the concept of *Monad* is really close to the concept of *Functor*. *Monad*&lt;A&gt; is some object that contains *A* that can be transformed into *Monad*&lt;B&gt; by application of function of type definition *A -> Monad&lt;B&gt;*. In functional programming this is called monadic bind. Also if we have **B** value we can transform it into *Monad&lt;B&gt;*. This is called *return* but in our framework we'll name it shortly *ret* because *return* is **Java** _keyword_ which can't be used for a method name. But that's not all there is more! Any *Monad* is a *Functor* too our *Monad* interface will extend *Functor* interface!

{% codeblock lang:java %}
public interface Monad<A> extends Functor<A> {

    <B> Monad<B> bind(final Function<A,Monad<B>> f);

    <B> Monad<B> ret(final B b);

}
{% endcodeblock %}

Now we have a base for our functional programming framework. Let's write some monads to better understand the concepts behind it. But before we do that let's define some base abstract class
that will have one another kind of monadic bind method that can be used on any *Monad&lt;A&gt;* binding it to a *Monad&lt;B&gt;* just by specifying the *Monad&lt;B&gt;* itself instead of function. Also we can use *Function&lt;A,B&gt;* for monadic bind too by combining it's result with *ret*, this method we will call *liftM*!

{% codeblock lang:java %}
public abstract class AbstractMonad<A> implements Monad<A> {

    public <B> Monad<B> bind(final Monad<B> b){
        return bind(new Function<A,Monad<B>>(){
            public Monad<B> apply(final A a){
                return b;            
            } 
         });
    }

    public <B> Monad<B> liftM(final Function<A,B> f){
        return bind(new Function<A,Monad<B>>(){
            public Monad<B> apply(final A a){
                return ret(f.apply(a));            
            } 
         });
    }

}
{% endcodeblock %}

Let's write our first *Monad*. It's called *Maybe&lt;A&gt;* and it can either contain **A** value or be empty! As you can see we need some methods in order to look inside this *Monad* and check if it contains a value or not! Also we'll need to implement *fmap* method for it to be a *Functor* and *bind* and *ret* methods for it to be a *Monad*! Implementing *fmap* is simple, we simply look into the *Monad* if it contains a value we apply provided function to this value, if not then we just return empty *Maybe&lt;A&gt;* monad. Same with the bind method but we don't need additional step of creating a *Maybe&lt;A&gt;* monad as the provided function returns it!

{% codeblock lang:java %}
public class Maybe<A> extends AbstractMonad<A> {

    private A a;
    private static final Maybe nothing = new Maybe(null);

    public static <A> Maybe<A> just(final A a){
        if(a == null)
            throw new IllegalArgumentException("argument has no value");
        return new Maybe<A>(a);
    }

    public static <A> Maybe<A> nothing(){
        return (Maybe<A>) nothing;
    }
    
    private Maybe(A a){
        this.a = a;
    }

    public A value(){
        if(this.a == null)
            throw new IllegalStateException("has no value");
        return this.a;
    }

    public A getValue(){
        if(this.a == null)
            throw new IllegalStateException("has no value");
        return this.a;
    }

    public boolean isNothing(){
        return this.a == null;
    }

    public boolean hasValue(){
        return this.a != null;
    } 

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        if(isNothing()){
            return Maybe.nothing();
        }else{
            return f.apply(this.value());
        }
    }

    public <B> Monad<B> ret(final B b){
        return Maybe.just(b);
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        if(isNothing())
            return Maybe.nothing();
        else
            return Maybe.just(f.apply(this.value()));
    }
}
{% endcodeblock %}

Now that we have *Maybe* monad we can implement another method in *Functional* class called *find*. Basically what find does is that it takes some *Predicate* function and a *Collection* of objects to check this predicate
and it returns the first element of that *Collection* for which the *Predicate* returns *true*. If for all elements the *Predicate* is false it returns empty *Maybe* monad.

{% codeblock lang:java %}
    public static <A> Maybe<A> find(final Predicate<A> f, final Collection<A> ac){
        for(A a : ac){
            if(f.apply(a))
                return Maybe.just(a);
        }
        return Maybe.nothing();
    }

    public static <A> boolean exists(final Predicate<A> f, final Collection<A> ac){
        return Functional.find(f,ac).hasValue();
    }
{% endcodeblock %}

Next is *Either&lt;A,B&gt;* monad. This monad contains either left *A* or right *B* value. This can be used to propagate exceptions or be used in any case where you need to either return one value or another but not both at the same time. Note this is *Monad&lt;B&gt;*. This code is almost identical to *Maybe&lt;A&gt;* but instead of 
returning empty value we return the monad itself since in the case of left value it can be considered a *Either&lt;A,C&gt;* monad.

{% codeblock lang:java %}
public class Either<A,B> extends AbstractMonad<B> {

    private A a;
    private B b;

    public static <A,B> Either<A,B> left(final A a){
        if(a == null)
            throw new IllegalArgumentException("argument has no value");
        return new Either<A,B>(a, null);
    }

    public static <A,B> Either<A,B> right(final B b){
        if(b == null)
            throw new IllegalArgumentException("argument has no value");
        return new Either<A,B>(null, b);
    }

    private Either(A a, B b){
        this.a = a;
        this.b = b;
    }

    public boolean isLeft(){
        return this.b == null;
    }

    public boolean isRight(){
        return this.a == null;
    }

    public B right(){
        if(this.b == null)
            throw new IllegalStateException("is not right");
        return this.b;
    }

    public B getRight(){
        if(this.b == null)
            throw new IllegalStateException("is not right");
        return this.b;
    }

    public A left(){
        if(this.a == null)
            throw new IllegalStateException("is not left");
        return this.a;
    }

    public A getLeft(){
        if(this.a == null)
            throw new IllegalStateException("is not left");
        return this.a;
    }

    public <C> Monad<C> bind(Function<B,Monad<C>> f){
        if(isRight())
            return f.apply(this.right());
        else
            return (Monad<C>)this;
    }

    public <C> Monad<C> ret(C c){
        return Either.right(c);
    }

    public <C> Functor<C> fmap(Function<B,C> f){
        if(isRight())
            return Either.right(f.apply(this.right()));
        else
            return (Functor<C>)this;
    }

}
{% endcodeblock %}

Next is a *Reader&lt;A&gt;* *Monad*. This *Monad* is usually used to propagate immutable piece of data such as configuration across your code. It's an interesting case to implement. We could implement it using a *Tuple&lt;E,A&gt;* however it's traditionally implemented as *Function&lt;E,A&gt;*. In order to get *E* value we implement *ask* method. For the rest of the methods we usually call *runReader()* and then apply *E* value to them. This way we propagate the *E* value across binded *Reader* monad's safely.

{% codeblock lang:java %}
public class Reader<E,A> extends AbstractMonad<A> {

    private Function<E,A> f;

    public static <E,A> Reader<E,A> reader(final Function<E,A> f){
        if(f == null)
            throw new IllegalArgumentException("argument has no value");
        return new Reader<E,A>(f);
    }

    private Reader(final Function<E,A> f){
        this.f = f;
    }

    public Function<E,A> runReader(){
        return this.f;
    }

    public Reader<E,E> ask(){
        return new Reader<E,E>((Function<E,E>)Identity.identity());
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new Reader<E,B>(
                new Function<E,B>(){
                    public B apply(final E e){
                        return ((Reader<E,B>)f.apply(runReader().apply(e)))
                                    .runReader().apply(e);
                    }
                });
    }

    public <B> Monad<B> ret(final B b){
        return new Reader<E,B>(
                new Function<E,B>(){
                   public B apply(final E e){
                       return b;
                   }
                });
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new Reader<E,B>(
                new Function<E,B>(){
                   public B apply(final E e){
                       return f.apply(runReader().apply(e));
                   }
               });
    }
}
{% endcodeblock %}

Before implementing next *Monad* let's explore another concept of Functional Programming called *Monoid*. Basically *Monoid* is some entity that can be empty or non-empty and can be appended to another *Monoid*. Appending empty *Monoid* to non-empty *Monoid* gives you the same non-empty *Monoid*. Simplest example of *Monoid* are natural numbers. In case of natural numbers we can consider zero as empty *Monoid* and any non-zero number to be non-empty *Monoid* and operation of appending two *Monoid*s to be addition of natural numbers. Let's write an interface of *Monoid*

{% codeblock lang:java %}
public interface Monoid<A> {

    Monoid<A> mempty(); 

    Monoid<A> mappend(final Monoid<A> a);

}
{% endcodeblock %}

Before we continue implementing monads let's implement one example of *Monoid*. First we'll define an *AbstractMonoid* class that will have some additional methods called *mconcat* for *mappend*-ing *List* or *Collection* of *Monoid*s starting from an empty *Monoid*.

{% codeblock lang:java %}
public abstract class AbstractMonoid<A> implements Monoid<A> {

    public Monoid<A> mconcat(final List<Monoid<A>> al){
        return Functional.foldLeft(new Function2<Monoid<A>,Monoid<A>,Monoid<A>>(){
            public Monoid<A> apply(final Monoid<A> a, final Monoid<A> b){
                return a.mappend(b);
            }
        }, this.mempty(), al);
    }

    public Monoid<A> mconcat(final Collection<Monoid<A>> ac){
        return Functional.foldLeft(new Function2<Monoid<A>,Monoid<A>,Monoid<A>>(){
            public Monoid<A> apply(final Monoid<A> a, final Monoid<A> b){
                return a.mappend(b);
            }
        }, this.mempty(), ac);
    }

}
{% endcodeblock %}

And now we are ready to implement *List* *Monoid* which is basically a linked list of nodes that are either empty or contain a value and reference to next list node.
This implementing is ugly due to lack of [TCO](http://en.wikipedia.org/wiki/Tail_call) in **Java**, however it's not too complex and just repeats any linked list implementation in **Java** or any other 
programming language.

{% codeblock lang:java %}
public class List<A> extends AbstractMonoid<A> implements Functor<A> {

    private A hd;
    private List<A> tl;
    private static final List empty = new List(null,null);

    public static <A> List<A> list(final A head, final List<A> tail){
        if(head == null)
            throw new IllegalArgumentException("head has no value");
        if(tail == null)
            throw new IllegalArgumentException("tail has no value");
        return new List<A>(head,tail);
    
    }

    public static <A> List<A> list(final Collection<A> ac){
        List<A> l = null;
        List<A> r = null;

        for(final A a : ac){
            if(l == null){
                l = new List<A>(a, null);
                r = l;
            }else{
                l.tl = new List<A>(a, null);            
                l = l.tl;
            }
        }

        if(r == null)
            return (List<A>)List.empty();
        else
            l.tl = List.empty();

        return r;
    }

    public static <A> List<A> empty(){
        return empty;
    }

    private List(final A head, final List<A> tail){
        this.hd = head;
        this.tl = tail;
    }

    public A value(){
        if(this.hd == null)
            throw new IllegalStateException("head has no value");
        return this.hd;
    }

    public A getValue(){
        if(this.hd == null)
            throw new IllegalStateException("head has no value");
        return this.hd;
    }

    public int length(){
        int len = 0;
        List<A> l = this;

        while(l.hasValue()){
            len++;
            l = l.tail();
        }

        return len;
    }

    public A head(){
        if(this.hd == null)
            throw new IllegalStateException("head has no value");
        return this.hd;
    }

    public List<A> tail(){
        if(this.tl == null)
            throw new IllegalStateException("has no tail");
        return this.tl;
    }

    public boolean isEmpty(){
        return this.hd == null;
    }

    public boolean isEmptyTail(){
        return this.tl.isEmpty();
    }

    public boolean hasValue(){
        return this.hd != null;
    } 

    public Monoid<A> mempty(){
        return List.empty();
    }

    public Monoid<A> mappend(final Monoid<A> a){

        if(isEmpty())
            return a;
        
        List<A> r = this;

        while(!r.isEmptyTail()){
            r = r.tl;
        }

        r.tl = (List<A>)a;

        return this;
    }

    public <B> Functor<B> fmap(final Function<A,B> f){

        if(isEmpty())
            return List.empty();

        List<A> al = this; 
        List<B> r = new List<B>(f.apply(al.value()),(List<B>)List.empty());
        List<B> r2 = r;

        while(!al.isEmptyTail()){
            al = al.tail();
            r2.tl = new List<B>(f.apply(al.value()), r2.tl);
            r2 = r2.tl;
        }
        
        return r;
    }
}
{% endcodeblock %}

Now that we have *Monoid* let's implement a *Writer&lt;W,A&gt;* monad. It's used when we need to collect some values as we go, for example log messages. Note that W needs to be a valid *Monoid*, without this
we can't collect values into one entity. Inside we use *Tuple&lt;A,W&gt;* to store values. We have *tell* method to *mappend* new *Monoid* values to *Writer* and by unwrapping the *Tuple&lt;A,W&gt;* we can 
bind it to another *Writer&lt;W,B&gt;* monad.

{% codeblock lang:java %}
public class Writer<W extends Monoid,A> extends AbstractMonad<A> {

    private Tuple<A,W> t;

    public static <W extends Monoid,A> Writer<W,A> writer(final Tuple<A,W> t){
        if(t == null)
            throw new IllegalArgumentException("argument has no value");
        return new Writer<W,A>(t);
    }

    public static <W extends Monoid,A> Writer<W,A> writer(final A a, final W w){
        return new Writer<W,A>(Tuple.tuple(a,w));
    }

    private Writer(Tuple<A,W> t){
        this.t = t;
    }

    public Tuple<A,W> runWriter(){
        return this.t;
    }

    public Writer<W,Unit> tell(final W w){
        return new Writer<W,Unit>(Tuple.tuple(Unit.unit(), 
                                  (W)runWriter().second().mappend(w)));
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        final Writer<W,B> m = (Writer<W,B>)f.apply(runWriter().first());
        return new Writer<W,B>(Tuple.tuple(m.runWriter().first(), 
                               (W)runWriter().second().mappend(m.runWriter().second())));
    }

    public <B> Monad<B> ret(final B b){
        return new Writer<W,B>(Tuple.tuple(b, runWriter().second()));
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new Writer<W,B>(Tuple.tuple(f.apply(runWriter().first()),
                               runWriter().second()));
    }
}
{% endcodeblock %}

Next let's implement another monad called *State&lt;S,A&gt;*. Basicly *State* *Monad* is needed when we need some mutable value across whole computation and in a sense it's like programming in any imperative language.
*State* monad inside is actually a *Function&lt;S,Tuple&lt;A,S&gt;&gt;*. Basically we can imagine a *State* monad as a *Function* that takes some state *S* and returns *Tuple* that contains *A* and possibly changed state *S*. In a sense it's implementation is like *Writer* and *Reader* monads combined. We have two methods *get* and *set* to get current state value and modify it during computation.

{% codeblock lang:java %}
public class State<S,A> extends AbstractMonad<A> {

    private Function<S,Tuple<A,S>> f;

    public static <S,A> State<S,A> state(final Function<S,Tuple<A,S>> f){
        if(f == null)
            throw new IllegalArgumentException("argument has no value");
        return new State<S,A>(f);
    }

    public static <S,A> State<S,A> state(final A a){
        if(a == null)
            throw new IllegalArgumentException("argument has no value");
        return new State<S,A>(new Function<S,Tuple<A,S>>(){
            public Tuple<A,S> apply(final S s){
                return Tuple.tuple(a,s);
            }
        });
    }

    private State(final Function<S,Tuple<A,S>> f){
        this.f = f;
    }

    public Function<S,Tuple<A,S>> runState(){
        return this.f;
    }

    public State<S,S> get(){
        return new State<S,S>(new Function<S,Tuple<S,S>>(){
            public Tuple<S,S> apply(final S s){
                return Tuple.tuple(s,s);
            }
        });
    }

    public State<S,Unit> put(final S ns){
        return new State<S,Unit>(new Function<S,Tuple<Unit,S>>(){
            public Tuple<Unit,S> apply(final S s){
                return Tuple.tuple(Unit.unit(),ns);
            }
        });
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new State<S,B>(new Function<S,Tuple<B,S>>(){
            public Tuple<B,S> apply(final S s){
                return Tuple.tuple(((State<S,B>)f.apply(
                            runState().apply(s).first()
                        )).runState().apply(s).first(), s);
            }
        });
    }

    public <B> Monad<B> ret(final B b){
        return new State<S,B>(new Function<S,Tuple<B,S>>(){
            public Tuple<B,S> apply(final S s){
                return Tuple.tuple(b,s);
            }
        });
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new State<S,B>(new Function<S,Tuple<B,S>>(){
            public Tuple<B,S> apply(final S s){
                return Tuple.tuple(f.apply(runState().apply(s).first()),s);
            }
        });
    }
}
{% endcodeblock %}

Now what if we need *Reader* monad and at the same time we need some mutation that *State* monad provides. Yes we need something that will combine the power of both monads into one.
This is what *Monad Transformer* can do. First of all any *Monad Transformer* is also a *Monad*. We can *lift* any *Monad&lt;A&gt;* into *MonadTrans&lt;A&gt;*.

{% codeblock lang:java %}
public interface MonadTrans<A> extends Monad<A> {

    MonadTrans<A> lift(final Monad<A> m);

}
{% endcodeblock %}

Before we'll start implementing *Monad Transformer*s let's write some abstract base class that every *Monad Transformer* will inherit. 
It will have only one method called *lift* that will allow any *Monad* to be lifted into *Monad Transformer*.

{% codeblock lang:java %}
public abstract class AbstractMonadTrans<A> extends AbstractMonad<A> implements MonadTrans<A> {

    public MonadTrans<A> lift(final Monad<A> m){
        return (MonadTrans<A>)m.bind(new Function<A,Monad<A>>(){
            public Monad<A> apply(final A a){
                return ret(a);
            }
        });
    }
}
{% endcodeblock %}

First let's implement *MaybeT&lt;M,A&gt;* monad transformer. Note that *M* needs to be a valid *Monad* and so we use *extends* type definition.
Internally *MaybeT* is a *Monad&lt;Maybe&lt;A&gt;&gt;*. We check it's value by binding a function and inside we check if value is not empty we apply the provided function, if not we return wrapping it into
internal monad using *ret* method.

{% codeblock lang:java %}
public class MaybeT<M extends Monad,A> extends AbstractMonadTrans<A> {

    private Monad<Maybe<A>> m;

    public static <M extends Monad,A> MaybeT<M,A> maybeT(final Maybe<A> a, final M m){
        if(a == null)
            throw new IllegalArgumentException("argument has no value");
        if(m == null)
            throw new IllegalArgumentException("monad has no value");
        return new MaybeT<M,A>(m.ret(a));
    }

    public static <M extends Monad,A> MaybeT<M,A> maybeT(final Monad<Maybe<A>> m){
        if(m == null)
            throw new IllegalArgumentException("monad has no value");
        return new MaybeT<M,A>(m);
    }

    private MaybeT(final Monad<Maybe<A>> m){
        this.m = m;
    }

    public Monad<Maybe<A>> runMaybeT(){
        return this.m;
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new MaybeT<M,B>(runMaybeT().bind(new Function<Maybe<A>,Monad<Maybe<B>>>(){
            public Monad<Maybe<B>> apply(final Maybe<A> a){
                if(a.isNothing())
                    return runMaybeT().ret((Maybe<B>)Maybe.nothing());
                else{
                    return ((MaybeT<M,B>)f.apply(a.value())).runMaybeT();
                }
            } 
        }));
    }

    public <B> Monad<B> ret(final B b){
        return new MaybeT<M,B>(runMaybeT().ret(Maybe.just(b)));
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new MaybeT<M,B>(runMaybeT().bind(new Function<Maybe<A>,Monad<Maybe<B>>>(){
            public Monad<Maybe<B>> apply(final Maybe<A> a){
                return runMaybeT().ret((Maybe<B>)a.fmap(f));            
            } 
        }));
    }
}
{% endcodeblock %}

Next is *EitherT&lt;M,A,B&gt;* monad transformer. It's written almost the same as *MaybeT* but inside it's a *Monad&lt;Either&lt;A,B&gt;&gt;* but instead we operate with right and left values inside bind function.

{% codeblock lang:java %}
public class EitherT<M extends Monad,A,B> extends AbstractMonadTrans<B> {

    private Monad<Either<A,B>> m;

    public static <M extends Monad,A,B> EitherT<M,A,B> eitherT(final Either<A,B> a, final M m){
        if(a == null)
            throw new IllegalArgumentException("argument has no value");
        if(m == null)
            throw new IllegalArgumentException("monad has no value");
        return new EitherT<M,A,B>(m.ret(a));
    }

    public static <M extends Monad,A,B> EitherT<M,A,B> eitherT(final Monad<Either<A,B>> m){
        if(m == null)
            throw new IllegalArgumentException("monad has no value");
        return new EitherT<M,A,B>(m);
    }

    private EitherT(final Monad<Either<A,B>> m){
        this.m = m;
    }

    public Monad<Either<A,B>> runEitherT(){
        return this.m;
    }

    public <C> Monad<C> bind(final Function<B,Monad<C>> f){
        return new EitherT(runEitherT().bind(new Function<Either<A,B>,Monad<Either<A,C>>>(){
            public Monad<Either<A,C>> apply(final Either<A,B> a){
                if(a.isLeft())
                    return runEitherT().ret((Either<A,C>)Either.left(a.left()));
                else{
                    return ((EitherT<M,A,C>)f.apply(a.right())).runEitherT();
                }
            }
        }));
    }

    public <C> Monad<C> ret(final C c){
        return new EitherT(runEitherT().ret(Either.right(c)));
    }

    public <C> Functor<C> fmap(final Function<B,C> f){
        return new EitherT(runEitherT().bind(new Function<Either<A,B>,Monad<Either<A,C>>>(){
            public Monad<Either<A,C>> apply(final Either<A,B> a){
                return runEitherT().ret((Either<A,C>)a.fmap(f));            
            }
        }));
    }
}
{% endcodeblock %}

Now let's implement *ReaderT&lt;E,M,A&gt;* monad transformer. Internally it's implemented as *Function&lt;E,Monad&lt;A&gt;&gt;*. Basically it's implementation mimics that of *Reader* monad however
instead of *A* value we are dealing with *Monad&lt;A&gt;*.

{% codeblock lang:java %}
public class ReaderT<E,M extends Monad,A> extends AbstractMonadTrans<A> {

    private Function<E,Monad<A>> f;

    public static <E,M extends Monad,A> ReaderT<E,M,A> readerT(final Function<E,Monad<A>> f){
        if(f == null)
            throw new IllegalArgumentException("argument has no value");
        return new ReaderT<E,M,A>(f);
    }

    private ReaderT(final Function<E,Monad<A>> f){
        this.f = f;
    }

    public Function<E,Monad<A>> runReaderT(){
        return this.f;
    }

    public ReaderT<E,M,E> ask(){
        return new ReaderT<E,M,E>(new Function<E,Monad<E>>(){
            public Monad<E> apply(final E e){
                return runReaderT().apply(e).ret(e);
            }
        });
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new ReaderT<E,M,B>(new Function<E,Monad<B>>(){
            public Monad<B> apply(final E e){
                return runReaderT().apply(e).bind(f);            
            }
        });
    }

    public <B> Monad<B> ret(final B b){
        return new ReaderT<E,M,B>(new Function<E,Monad<B>>(){
            public Monad<B> apply(final E e){
                return runReaderT().apply(e).ret(b);            
            }
        });
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new ReaderT<E,M,B>(new Function<E,Monad<B>>(){
            public Monad<B> apply(final E e){
                return runReaderT().apply(e).bind(new Function<A,Monad<B>>(){
                    public Monad<B> apply(final A a){
                        return runReaderT().apply(e).ret(f.apply(a));
                    }
                });            
            }
        });
    }

}
{% endcodeblock %}

Next monad transformer is *WriterT&lt;W,M,A&gt;*. This one is implemented internally as *Monad&lt;Tuple&lt;A,W&gt;&gt;* and follows the same pattern as *MaybeT* and *EitherT* monad transformer implementations.

{% codeblock lang:java %}
public class WriterT<W extends Monoid,M extends Monad,A> extends AbstractMonadTrans<A> {

    private Monad<Tuple<A,W>> m;

    public static <W extends Monoid,M extends Monad,A> WriterT<W,M,A> writerT(final A a, final W w, final M m){
        return new WriterT<W,M,A>((Monad<Tuple<A,W>>)m.ret(Tuple.tuple(a,w)));
    }

    private WriterT(final Monad<Tuple<A,W>> m){
        this.m = m;
    }

    public Monad<Tuple<A,W>> runWriterT(){
        return this.m;
    }

    public WriterT<W,M,Unit> tell(final W w){
        return new WriterT<W,M,Unit>(runWriterT().bind(
            new Function<Tuple<A,W>,Monad<Tuple<Unit,W>>>(){
                public Monad<Tuple<Unit,W>> apply(final Tuple<A,W> t){
                    return runWriterT().ret(Tuple.tuple(Unit.unit(),(W)t.second().mappend(w))); 
                }
        }));
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new WriterT<W,M,B>(runWriterT().bind(new Function<Tuple<A,W>,Monad<Tuple<B,W>>>(){
            public Monad<Tuple<B,W>> apply(final Tuple<A,W> t){
                return ((WriterT<W,M,B>) f.apply(t.first())).runWriterT().bind(
                    new Function<Tuple<B,W>,Monad<Tuple<B,W>>>(){
                        public Monad<Tuple<B,W>> apply(final Tuple<B,W> t2){
                            return runWriterT().ret(Tuple.tuple(t2.first(),t.second()));
                        }                
                });
            }
        }));
    }

    public <B> Monad<B> ret(final B b){
        return new WriterT<W,M,B>(runWriterT().bind(new Function<Tuple<A,W>,Monad<Tuple<B,W>>>(){
            public Monad<Tuple<B,W>> apply(final Tuple<A,W> t){
                return runWriterT().ret(Tuple.tuple(b,t.second()));
            }
        }));
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new WriterT<W,M,B>(runWriterT().bind(new Function<Tuple<A,W>,Monad<Tuple<B,W>>>(){
            public Monad<Tuple<B,W>> apply(final Tuple<A,W> t){
                return runWriterT().ret(Tuple.tuple(f.apply(t.first()),t.second()));
            }
        }));
    }
}
{% endcodeblock %}

Last monad transformer we are implementing is *StateT&lt;S,M,A&gt;*. It's the most complex implementation among all monad transformers.
Internally it mimics *State* monad however it's implemented as *Function&lt;S,Monad&lt;Tuple&lt;A,S&gt;&gt;&gt;* so general implementation
follows that of *State* with the difference that function returns monad but not a value. You can study this source code line by line
to better understand all the inner workings however I strongly encourage you to try implement it yourself. In a matter of fact try writing all the monad transformers from scratch yourself 
and you'll grok their beauty easily.

{% codeblock lang:java %}
public class StateT<S,M extends Monad,A> extends AbstractMonadTrans<A> {

    private Function<S,Monad<Tuple<A,S>>> f;

    private StateT(final Function<S,Monad<Tuple<A,S>>> f){
        this.f = f;
    }

    public Function<S,Monad<Tuple<A,S>>> runStateT(){
        return this.f;
    }

    public StateT<S,M,S> get(){
        return new StateT<S,M,S>(new Function<S,Monad<Tuple<S,S>>>(){
            public Monad<Tuple<S,S>> apply(final S s){
                return runStateT().apply(s).ret(Tuple.tuple(s,s));
            }
        });
    }

    public StateT<S,M,Unit> put(final S ns){
        return new StateT<S,M,Unit>(
            new Function<S,Monad<Tuple<Unit,S>>>(){
                public Monad<Tuple<Unit,S>> apply(final S s){
                    return runStateT().apply(s).ret(Tuple.tuple(Unit.unit(),ns));
                }
        });
    }

    public <B> Monad<B> bind(final Function<A,Monad<B>> f){
        return new StateT<S,M,B>(
            new Function<S,Monad<Tuple<B,S>>>(){
                public Monad<Tuple<B,S>> apply(final S s){
                    return runStateT().apply(s).bind(
                        new Function<Tuple<A,S>,Monad<Tuple<B,S>>>(){
                            public Monad<Tuple<B,S>> apply(final Tuple<A,S> t){
                                return f.apply(t.first()).bind(
                                    new Function<B,Monad<Tuple<B,S>>>(){
                                        public Monad<Tuple<B,S>> apply(final B b){
                                            return runStateT().apply(s).ret(Tuple.tuple(b,s));
                                        }
                                });
                            }
                    });
                }
        });
    }

    public <B> Monad<B> ret(final B b){
        return new StateT<S,M,B>(
            new Function<S,Monad<Tuple<B,S>>>(){
                public Monad<Tuple<B,S>> apply(final S s){
                    return runStateT().apply(s).ret(Tuple.tuple(b,s));
                }
        });
    }

    public <B> Functor<B> fmap(final Function<A,B> f){
        return new StateT<S,M,B>(new Function<S,Monad<Tuple<B,S>>>(){
            public Monad<Tuple<B,S>> apply(final S s){
                return runStateT().apply(s).bind(
                    new Function<Tuple<A,S>,Monad<Tuple<B,S>>>(){
                        public Monad<Tuple<B,S>> apply(final Tuple<A,S> t){
                            return runStateT().apply(s).ret(
                                Tuple.tuple(f.apply(t.first()),s)
                            );                    
                        }
                });
            }
        });
    }

}
{% endcodeblock %}

That's all folks! We've implemented a small functional library in **Java**, it's not complete and not as powerful as [Functional Java](http://www.functionaljava.org/), however doing so brings us a little bit closer to functional zen. Long road still lays ahead of us as we journey this path of functional programming, but we are stronger than we were! Good luck to you all adventurers and happy functional programming!

![Rikka Kawaii](http://i.imgur.com/9gKTf4u.jpg)
