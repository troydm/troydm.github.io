---
layout: post
title: "Of All The Garbage in The World"
date: 2015-09-06 00:01:19 +0400
comments: true
categories: [linux, gc, memory, c] 
---

TL;DR *Writing tri-color (actually 4-color) incremental generational garbage collector in C*

[Previously](http://troydm.github.io/blog/2015/08/03/lifting-shadows-off-a-memory-allocation/) we've talked about manual memory allocation and unraveled some shadows around it. 
Now let's talk about automatic memory allocation, more specifically about [Garbage Collection](https://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29).
Most if not all modern Programming Languages contain a mechanism that handles allocation of memory automatically at precisely the moment application needs it and deallocation of that
memory at the moment it's not needed anymore. However determining when memory region (or object) is safe to deallocate is not an easy feat as we need to somehow determine if the region is needed or not.
And here is where *Garbage Collection* comes into a play. Simply speaking *Garbage Collection* is an algorithm that checks entire memory of an application for regions that aren't referenced anywhere inside application by other memory regions and deallocates these regions because they aren't needed anymore. 

![Space Dandy](http://i.imgur.com/v5Czgfd.png)

*Garbage Collection* was invented by [John McCarthy](https://en.wikipedia.org/wiki/John_McCarthy_%28computer_scientist%29) back in 1959 for [Lisp](https://en.wikipedia.org/wiki/Lisp_%28programming_language%29) and is still one of the hottest evolving topics in Computer Science nowadays, especially with [Java](https://en.wikipedia.org/wiki/Java_%28software_platform%29)'s GC becoming more famous with tons of blows and whistles which can be tuned to practically any application memory usage scenario. There are also alternative models to automatic memory handling such as [newLisp](https://en.wikipedia.org/wiki/NewLISP)'s ORO, [Rust](https://en.wikipedia.org/wiki/Rust_%28programming_language%29)'s ownership based life cycle management and [Objective-C](https://en.wikipedia.org/wiki/Objective-C)'s ARC but we won't be talking about those. The cost of checking entire memory region is not a cheap one so people realized ways to make it more reasonable and thus nowadays we have various types of *Garbage Collection* algorithms. There are tracing garbage collectors and reference counting ones, copying (moving) and non-copying (non-moving) ones, incremental and non-incremental, generational, concurrent, parallel and real-time ones. To learn more about all those types of *Garbage Collectors* I suggest you try reading [The Garbage Collection Handbook](http://www.amazon.com/Garbage-Collection-Handbook-Management-Algorithms/dp/1420082795/) which is a fundamental must read handbook for all those interested in the topic. But for rest of you lazy bunch let's start collecting all the garbage in the world by writing one garbage collector ourselves. 

<!--more-->

Let's write a single threaded tracing tri-color (actually 4-color, I'll explain later why) non-copying incremental n-generational (n-generational meaning that we can configure any number of generations dynamically on startup, but our implementation will have limitation of maximum 64 generations) *Garbage Collector* in *C*. We won't tackle multithreading, concurrency and parallel garbage collection for sake of simplicity as those are more complex to implement but maybe I'll do another blog post about them later this year. Our simple garbage collector will be single threaded only, meaning it will correctly work only in applications that don't use multiple threads or somehow allocate memory in parallel. This kind of *Garbage Collector* is sufficient for languages that aren't natively multithreaded, such as [Node.js](https://en.wikipedia.org/wiki/Node.js) and [Pharo](https://en.wikipedia.org/wiki/Pharo), so if you are planning to write such language yourself this might be a useful exercise for you. For those of you impatient ones there is a [simplegc](https://github.com/troydm/simplegc/) repo with all the source code to study. So let's get started.

We need a way to represent our memory regions so we'll go for the most obvious object oriented representation, meaning we'll just write a garbage collector for object-oriented programming language of ours.
Each object will be part of doubly linked list so we could trace it and hence it will contain a pointer to a pointer of a previous object pointing to it called **gc_prev** and a pointer to the next object called **gc_next**.
Each object will have a pointer to a **class** description which we'll talk about later.
Each object will also have 32-bit unsigned integer called **gc_mark** which will serve three purposes: to mark object's color, gc generation and count root references (I'll explain those later). Each object might also reference other objects for example we might have fields inside object which could contain other objects so hence we need **refs_count** variable to count number of fields object has.
Since each object will be of a class we also need to represent the class itself. It will contain function pointers that will be used during garbage collection. One of them is the most obvious function pointer which is a **gc_finalize** function. This function will be called precisely before the moment of freeing memory region which is not used anymore or simply speaking just before the object will be deleted by the *Garbage Collector*. **gc_mark_black** function pointer will point to a function used to mark object from *grey* to *black* and mark all objects it's referring (or contains) as *grey* for further processing. **gc_contains** will be a function used to check if an object is referenced (or contained) by another object.

{% codeblock lang:c %}
struct gc_object_class_t;

// gc object
typedef struct gc_object_t {
    struct gc_object_t** gc_prev;
    struct gc_object_t* gc_next;
    uint32_t gc_mark;
    struct gc_object_class_t* class;    
    uint16_t refs_count;
} gc_object;

// gc object class
typedef struct gc_object_class_t {
    void (*gc_mark_black)(gc_object* obj); // this marks  object from grey to black
    bool (*gc_contains)(gc_object* obj, gc_object* ref); // this checks if object contains reference object
    void (*gc_finalize)(gc_object* obj); // this is called before object is deallocated
} gc_object_class;
{% endcodeblock %}

Now let's talk about **gc_mark**. As I've said earlier it'll serve three purposes for our object: root reference count, object's color marking, and gc generation.
We'll use first 24 bits for root reference counting, next 2 bits will be used for color (as we'll have 4 possible colors we need 2 bits since 2^2 is 4) and remaining 6 bits will be used for gc generation (meaning we can't have more than 64 generations since 2^6 is 64, but that is enough for any use case).

<canvas id="gc_mark" width="400" height="75"></canvas> 
<script>
function drawRect(ctx,text,x,y,w,h){
    ctx.fillRect(x,y,w,h);
    ctx.moveTo(x,y);
    ctx.lineTo(x,y+h);
    ctx.stroke();
    ctx.moveTo(x,y);
    ctx.lineTo(x+w,y);
    ctx.stroke();
    ctx.moveTo(x+w,y);
    ctx.lineTo(x+w,y+h);
    ctx.stroke();
    ctx.moveTo(x,y+h);
    ctx.lineTo(x+w,y+h);
    ctx.stroke();
    var fs = ctx.fillStyle;
    ctx.fillStyle = "#000000";
    var tl = 16*text.length;
    ctx.fillText(text,x+(w/2)-(tl/4)+5,y+(h/2)+4);
    ctx.fillStyle = fs;
}
var c = document.getElementById("gc_mark");
var ctx = c.getContext("2d");
ctx.font="normal 12pt Sans";
ctx.fillStyle = "#000000";
ctx.fillText("gc_mark (32-bit unsigned integer)",5,15);
ctx.fillStyle = "#F8D7BB";
ctx.strokeStyle = "#000000";
drawRect(ctx,"24 bits",0,20,300,35);
drawRect(ctx," 2 bits",300,20,50,35);
drawRect(ctx," 6 bits",350,20,50,35);
ctx.fillStyle = "#000000";
ctx.fillText("root reference count",95,70);
ctx.fillText("color",310,70);
ctx.fillText("gen",365,70);
</script>

Let's define some useful macros to work with **gc_mark**. Most of them are related to bitwise magic so I won't explain them in details but instead leave that as self-study for you.

{% codeblock lang:c %}
// gc mark
#define gc_set_mark(o,r,gi) o->gc_mark = (r << 8) | gi
// generation part
#define gc_gen_part(o) (o->gc_mark & 0xFF)
#define gc_gen_num(o) (o->gc_mark & 0x3F)
#define gc_gen_set(o,g) gc_set_mark(o,gc_root_ref_count(o), gc_color_bit(o) | (g & 0x3F))
// color bits
#define gc_color_bit(o) (o->gc_mark & 0xC0)
#define gc_color(o) (gc_gen_part(o) >> 6)
#define gc_color_is_white(o) (gc_color_bit(o) == 0x00)
#define gc_color_is_grey(o) (gc_color_bit(o) == 0x40)
#define gc_color_is_black(o) (gc_color_bit(o) == 0x80)
#define gc_color_is_silver(o) (gc_color_bit(o) == 0xC0)
#define gc_color_is_silver_or_white(o) (gc_color_is_silver(o) || gc_color_is_white(o))
#define gc_mark_white(o) o->gc_mark &= 0xFFFFFF3F
#define gc_mark_grey(o) gc_mark_white(o); o->gc_mark |= 0x40
#define gc_mark_black(o) gc_mark_white(o); o->gc_mark |= 0x80
#define gc_mark_silver(o) o->gc_mark |= 0xC0
// root reference count
#define gc_root_ref_count(o) (o->gc_mark >> 8)
#define gc_inc_root_ref_count(o) gc_set_mark(o,(gc_root_ref_count(o)+1),gc_gen_part(o))
#define gc_dec_root_ref_count(o) gc_set_mark(o,(gc_root_ref_count(o)-1),gc_gen_part(o))
{% endcodeblock %}

Now let's describe configuration for our garbage collector. First thing first we need to limit duration
of our garbage collection cycle since we don't want it to be too long and are writing an incremental garbage 
collector after all. For this we'll have **max_pause** configuration property which will define maximum amount of 
nanoseconds gc cycle can run. Also since we need to somehow check if our cycle is exceeding it's pause limit or not
we just can't do it after every object we check during gc cycle, thus we need **pause_threshold** which is number of
objects checked after which gc pause check occurs. Next we define number of generations our garbage collector will have
using **gens_count** and provide generation configurations array using **gens**.
Each generation configuration will have **refresh_interval** which is number of nanoseconds passed after last full gc cycle
after which entire generation of objects should be rechecked. Now we need to clarify what is full gc cycle.
Full gc cycle is garbage collection cycle that is not interrupted after hitting max pause threshold.
Last thing our generation configuration has is **promotion_interval** which is number of nanoseconds passed after last full gc cycle
after which our objects in generation are promoted to higher generation. This configuration property is not relevant for the last generation since there is no next generation to promote to.

{% codeblock lang:c %}
// gc config
typedef struct {
    uint64_t max_pause; // gc max pause in nanoseconds
    uint32_t pause_threshold; // number of objects checked after which gc pause check should occur
    uint8_t gens_count; // number of generations
    gc_gen_config* gens; // generation configs array
} gc_config;

// generation config
typedef struct {
    uint64_t refresh_interval; // generation refresh interval in nanoseconds
    uint64_t promotion_interval; // generation promotion interval in nanoseconds
} gc_gen_config;
{% endcodeblock %}

Let's talk about why we need 4 colors for describing the state of the object in relevance to current garbage collection cycle.
In traditional tri-color garbage collector objects can be *white*, *grey* and *black*. Newly created objects are marked as *white*.
After object is identified to have a root reference it's color is changed from *white* to *grey*. Next all *grey* objects are checked for
references of other objects that they contain. For each contained reference if reference is *white* it's marked as *grey* for future checking after which the container object is marked as *black*. All remaining *grey* objects are subsequently checked for references of objects they contain. After this checks we have only *black* and *white* objects. Objects marked as *black* are reachable from the root and hence don't need to be garbage collected. Remaining white objects on the other hand are considered garbage and freed in the end. You can read about this in more detail [here](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Tri-color_marking). 

Traditional tri-color algorithm doesn't has a notion of generation in mind because checking objects only in some specific limited generation won't be correct for determining if objects are garbage or not as some of those objects might have references from other generations. Let's consider that you have a generation of objects and you suddenly want to recheck them all. What you will probably do is mark all objects as *white* again and then mark objects that have root references as *grey* and go from there. However the problem with this approach is that in the end we might have white objects that don't have direct references in generation they are part of but have indirect reference from root objects in another generation. This is why we need 4 colors. We will treat those objects as *silver* and won't consider them as *white* yet. For each such object we'll check all *black* objects in all generations for possible references. If such reference will be found we'll mark that object as *grey* and will promote it to the generation that has reference to it. This might introduce some generation bouncing but since all live objects in all generations are subsequently promoted after some period of time to last generation this won't be a problem. 

Since we need to be incremental during our garbage collection cycle for all white objects that are considered garbage we'll have a separate color that we'll call *transparent*, but we won't need additional bit in **gc_mark** to identify objects with such color. This kind objects won't be marked directly but instead will be part of **transparent** list that will be used during last step in our incremental garbage collection cycle to call **gc_finalize** function for each object in that list and **free** it from memory. This is needed because **gc_finalize** and **free** calls can be time costly operations hence we need to delay them until our gc cycle is fully over so they could be executed in the remaining cycle time. This way we can have a nearly timely precise incremental garbage collector that won't ever exceed our predefined **max_pause** time.

Since we are going to count time using nanoseconds we need a way to measure nanoseconds passed between execution of arbitrary code.
For this we'll use Linux *syscall* **clock_gettime**.

{% codeblock lang:c %}
// get current time in nano seconds
uint64_t get_nanotime(){
    struct timespec t;
    clock_gettime(CLOCK_MONOTONIC, &t);
    return (((uint64_t)t.tv_sec)*1000000000) + ((uint64_t)t.tv_nsec);
}
{% endcodeblock %}

Now let's define some global constants and variables for our garbage collector. Note how we are going to have 
single double linked lists for each gc color except *black*. 
Each generation will have it's own *black* objects list and hence we'll need an array of doubly linked objects lists.

{% codeblock lang:c %}
#define WHITE 0
#define GREY  1
#define BLACK 2
#define SILVER 3

// gc configuration
static gc_config conf;
// gc list variables
static gc_object* transparent = null;
static gc_object* white = null;
static gc_object* silver = null;
static gc_object* grey = null;
static gc_object** black;
{% endcodeblock %}

Finally we can start writing some functions. Let's start with Garbage Collector initialization function first.
This will be called on startup and only once. It checks if generation count is correct and next thing is it just copies over configuration object including
the contained generation configurations.

{% codeblock lang:c %}
// initialize garbage collector
void gc_init(gc_config* config){
    // number of generations can't be more than 64 or equal to 0
    if(config->gens_count == 0 || config->gens_count > 64){
        errno = EINVAL;
        return;
    }
    // copy config
    conf = *config;
    conf.gens = (gc_gen_config*)malloc(sizeof(gc_gen_config) * conf.gens_count);    
    black = (gc_object**)malloc(sizeof(gc_object*) * conf.gens_count);
    // foreach generation config
    for(uint8_t i = 0; i < conf.gens_count; ++i){
        // copy config and initialize generation
        conf.gens[i] = config->gens[i];
        conf.gens[i].refresh_time = get_nanotime();
        conf.gens[i].promotion_time = conf.gens[i].refresh_time;
        black[i] = null;
    }
}
{% endcodeblock %}

We also need a function to destroy our garbage collector when it's not needed anymore.
This needs to free up all doubly linked lists as those can be considered part of garbage collector.
Note how we call **gc_finalize** function before freeing up object.

{% codeblock lang:c %}
// completely free entire list finalizing all objects inside
inline void gc_free_list(gc_object* list){
    while(list != null){
        gc_object* obj = list;
        list = list->gc_next; 
        (obj->class->gc_finalize)(obj);
        free(obj);
    }
}

// deinitialize garbage collector
void gc_destroy(){
    // free all objects from all lists
    gc_free_list(transparent);
    gc_free_list(white);
    gc_free_list(silver);
    gc_free_list(grey);
    for(uint8_t i = 0; i < conf.gens_count; ++i)
        gc_free_list(black[i]);
    // remove black list and generation configs
    free(black);
    free(conf.gens);
}
{% endcodeblock %}

Now let's define some helper functions for working with doubly linked lists. 
We'll use those to add, remove, and move objects from one list to another.

{% codeblock lang:c %}
// add object to list
inline void gc_list_add(gc_object** list, gc_object* obj){
    obj->gc_next = *list;
    if(obj->gc_next != null)
        obj->gc_next->gc_prev = &(obj->gc_next);
    obj->gc_prev = list;
    *list = obj;
}

// remove object from current list
inline void gc_list_remove(gc_object* obj){
    *(obj->gc_prev) = obj->gc_next;
    if(obj->gc_next != null)
        obj->gc_next->gc_prev = obj->gc_prev;
}

// move object to list
inline void gc_list_move(gc_object* obj, gc_object** list){
    gc_list_remove(obj);
    gc_list_add(list,obj);
}

// move all objects from list to list
inline void gc_list_move_all(gc_object** from, gc_object** to){
    if(*to != null){
        gc_object* obj = *from;
        while(obj->gc_next != null)
            obj = obj->gc_next;
        obj->gc_next = *to;
        (*to)->gc_prev = &(obj->gc_next);
    }
    *to = *from;
    (*to)->gc_prev = to;
    *from = null;
}
{% endcodeblock %}

Now in order to determine if object is reachable or not we'll just use root reference counting instead of going over the stack and marking objects each time.
This might be considered more expensive as each time we set a reference to some object on the stack we need to increment it's root reference count. 
And every time we pop a stack frame or modify a reference on the stack we need to decrement root reference count. Doing this many times is expensive indeed however from my point of view this approach is more faster in long term compared to scanning the stack each time during garbage collection. But if you really want it our simple garbage collector can be easily modified to use the scanning approach if needed.

As you can see from code when we increment root reference count we also check if object is white and mark it as grey for future check by Garbage Collector. 
We could omit this step, however in doing so we would require to go through entire white objects set and check their root reference count each time during Garbage Collector cycle. As you can see there are no additional steps during decrement of root reference count, and that is because we don't need to do anything since when root reference count becomes 0 our object will still be refreshed during generation refresh phase and during it the root reference count will be checked anyway.

{% codeblock lang:c %}
// add gc root
void gc_add_root(gc_object* obj){
    // increase root ref count
    gc_inc_root_ref_count(obj);
    // mark object as grey if it's white or silver
    if(gc_color_is_silver_or_white(obj)){
        gc_list_move(obj,&grey);
        gc_mark_grey(obj);
    }
}

// remove gc root
void gc_remove_root(gc_object* obj){
    // decrease root ref count
    if(gc_root_ref_count(obj) != 0)
        gc_dec_root_ref_count(obj);
}
{% endcodeblock %}

Next thing we need to do is to actually allocate the object itself and initialize it's **gc_mark**.
We also need to add it to white objects list as all new objects should be added to white list.
Without it we won't be able to track allocated objects. Look at how we also initialize references to *null*
based on how much references an object could contain. This is specified using **refs_count** argument to the function.

{% codeblock lang:c %}
// allocate gc_object
gc_object* gc_alloc(uint32_t refs_count){
    // allocate new white object in generation 0 with 0 root ref count
    gc_object* obj = (gc_object*)malloc(sizeof(gc_object) + refs_count*sizeof(gc_object*));

    // initialize references
    obj->refs_count = refs_count; // number of references this object might contain
    gc_object** refs = (gc_object**)(obj+1); // start of refs array
    for(uint16_t i = 0; i < refs_count; ++i)
        refs[i] = null;

    // set gc mark
    obj->gc_mark = 0; // initial gc_mark value (0 generation white color)
    // add object to white list
    gc_list_add(&white,obj);
    return obj;
}
{% endcodeblock %}

Since our object could contain references we need a way to be able to modify those references.
Since we are introducing mutation we need to be able to revert the color of black object back to grey in order
to be able to recheck it during gc cycle. This operation is not expensive and is executed only once during mutation
between gc cycles if mutation of the object occurs thus we don't get any performance penalties by doing so each time on
reference modification.

{% codeblock lang:c %}
// set object reference to another object
void gc_set_ref(gc_object* obj, uint16_t ref_index, gc_object* ref){
    gc_object** refs = (gc_object**)(obj+1); // start of refs array
    refs[ref_index] = ref;
    // if object is black because it mutated we need to mark it grey again
    if(gc_color_is_black(obj)){
        gc_list_move(obj,&grey);
        gc_mark_grey(obj);
    }
}
{% endcodeblock %}

Let's start writing **gc** function. First we need to initialize cycle counters which we'll be used during our gc cycle. I've added
this variables to **gc_config** *struct* just for the sake of simplicity. 
First thing we are going to do during our gc cycle is to free all transparent objects that might have been left over from the previous gc cycle. Before freeing each object we call **gc_finalize** function from it's *class*. Note however that since we are writing an incremental garbage collector we need to be able to end the cycle prematurely if it takes too much time. To do so we'll be using **gc_cycle_check** *macro*. In order for *gc_cycle_check* to determine if the check for threshold should occur or not we increment **cycle_threshold** counter.
During **gc_cycle_check** we just check if **cycle_threshold** is bigger than configured **pause_threshold** and if it is we just check the amount of passed time in nano seconds since the start of the gc cycle. If the time passed exceeds **max_pause** configuration value we just call **gc_cycle_end** function and end cycle prematurely.
We'll rely on **gc_cycle_check** *macro* heavily through our entire gc cycle.

{% codeblock lang:c %}
#define gc_cycle_check \
if(conf.cycle_threshold >= conf.pause_threshold){ \
    if((get_nanotime() - conf.cycle_time) >= conf.max_pause){ \
        return gc_cycle_end(); \
    } \
    conf.cycle_objects += conf.cycle_threshold; \
    conf.cycle_threshold = 0; \
}

// end gc cycle
static inline uint64_t gc_cycle_end(){
    conf.cycle_objects += conf.cycle_threshold;
    conf.cycle_duration = get_nanotime() - conf.cycle_time;
    return conf.cycle_duration;
}

// start gc cycle
conf.cycle_time = get_nanotime();
conf.cycle_threshold = 0;
conf.cycle_objects = 0;
conf.cycle_collected = 0;
conf.cycle_full = false;

// transparent cleanup phase before cycle
while(transparent != null){
    (transparent->class->gc_finalize)(transparent);
    gc_object* obj = transparent;
    transparent = obj->gc_next;
    free(obj);
    conf.cycle_threshold += 11;
    conf.cycle_collected += 1;
    // check pause threshold
    gc_cycle_check
}
{% endcodeblock %}

After cleaning up transparent list next thing what we are going to do is promote generations that need promotion and refresh those that need refreshment.
Now this might sound something complex but it's pretty much straight forward. We just track time of last promotion and refreshment for each generation in **promotion_time** and
**refresh_time** variables and compare them with configuration intervals called **promotiona_interval** and **refresh_interval** configured individually for each generation.
Note how we skip promotion of the last generation as there is no generation to promote to. During promotion what we are doing is just incrementing generation setting and moving
object from one generation black list to another. Note that while doing that we also check gc pause threshold using **gc_cylce_check** *macro*. During refresh we are checking root
reference count and based on that move objects either to *grey* list or *silver*.

{% codeblock lang:c %}
// promotion phase
uint64_t time_now = get_nanotime();
for(uint8_t i = 0; i < conf.gens_count; ++i){
    conf.gens[i].cycle_refreshed = 0;
    conf.gens[i].cycle_promoted = 0;
    gc_object* obj = black[i];
    if(obj != null){
        // promote generation
        if(i != (conf.gens_count-1) && time_now - conf.gens[i].promotion_time > conf.gens[i].promotion_interval){
            do{
                gc_gen_set(obj,(i+1));
                // move to next generation
                gc_list_move(obj,&(black[i+1]));

                conf.cycle_threshold += 1;
                conf.gens[i].cycle_promoted += 1;
                // check pause threshold
                gc_cycle_check
                obj = black[i];
            }while(obj != null);
            conf.gens[i].promotion_time = get_nanotime();
        }

        // refresh generation
        if(time_now - conf.gens[i].refresh_time > conf.gens[i].refresh_interval){
            while(obj != null){
                // if has root references
                if(gc_root_ref_count(obj) > 0){
                    // mark as grey
                    gc_list_move(obj,&grey);
                    gc_mark_grey(obj);
                }else{
                    // mark as silver
                    gc_list_move(obj,&silver);
                    gc_mark_silver(obj);
                }
                conf.cycle_threshold += 1;
                conf.gens[i].cycle_refreshed += 1;
                // check pause threshold
                gc_cycle_check
                obj = black[i];
            }
            conf.gens[i].refresh_time = get_nanotime();
        }
    }
}
{% endcodeblock %}

Next phase after promotion/refreshment is marking phase.
During this phase we go over all *grey* objects and mark them as *black* using class assigned **gc_mark_black** function pointer.
Now in our *gc_object*'s class we assigned it to **gc_object_mark_black** function which is straightforward as we just iterate over
references object contains, mark them as *grey* if they are *white* or *silver* and after that mark object as *black*.
Note how we are using **gc_cycle_check_no_return** *macro* which is almost the same as **gc_cycle_check** *macro* except that it doesn't
 ends the cycle by calling **gc_cycle_end** as we are doing the same check in caller code using **gc_cycle_check** *macro*. 
This is needed to end cycle prematurely from callee *gc_mark_black* function.
This might sound as overly complex but this way we can extend our garbage collector to support objects with any kind of reference layout which might come handy for future.

{% codeblock lang:c %}
// mark phase
while(grey != null){
    (grey->class->gc_mark_black)(grey);
    conf.cycle_threshold += 1;
    // check pause threshold
    gc_cycle_check
}

#define gc_cycle_check_no_return \
if(conf.cycle_threshold >= conf.pause_threshold){ \
    if((get_nanotime() - conf.cycle_time) >= conf.max_pause){ \
        return; \
    } \
    conf.cycle_objects += conf.cycle_threshold; \
    conf.cycle_threshold = 0; \
}

// gc object mark black
void gc_object_mark_black(gc_object* obj){

    gc_object** refs = (gc_object**)(obj+1); // start of refs array

    // for each ref
    for(uint16_t i = 0; i < obj->refs_count; ++i){
        if(refs[i] != null && gc_color_is_silver_or_white(refs[i])){
            // mark object as grey
            gc_list_move(refs[i],&grey);
            gc_mark_grey(refs[i]);                        
            conf.cycle_threshold += 1;
            // check pause threshold
            gc_cycle_check_no_return
        }
    }

    // mark object as black
    gc_list_move(obj,&(black[gc_gen_num(obj)]));
    gc_mark_black(obj);                        
}
{% endcodeblock %}

Next comes the tricky part. Now we are left with *silver* objects that aren't entirely white or grey yet.
What we are going to do is to go through all generations starting with older ones and check if objects in that generation might contain 
our *silver* object. This might sound trivial but it's more tricky. We are going to check our *silver* objects using **gc_contains** function pointer in the *class*
which in our case will point to **gc_object_contains** function. This is all for the same sake of extensibility of our garbage collector.
Now if the object is not found in all black objects of all generations we can safely mark it as *white*. On the other hand if it's found  we need to 
first correct it's generation, mark it as *grey* and recheck all left over *grey* objects all over again in order to safely trace references that this object might contain which on the other hand might be some *silver* objects too. This way we decrease the amount of *silver* objects that we need to check to essential minimum as the operation itself is very expensive.

{% codeblock lang:c %}
// mark silver phase
while(silver != null){
    bool found = false;
    uint8_t to_gen;
    uint8_t i = conf.gens_count - 1;
    while(true){
        gc_object* obj = black[i];
        while(obj != null){
            if((obj->class->gc_contains)(obj,silver)){
                found = true;
                to_gen = i;
                break;
            }
            gc_cycle_check
            obj = obj->gc_next;
        }
        if(found || i == 0)
            break;
        --i;
    }
    if(found){
        // correct generation and mark as grey
        gc_gen_set(silver,to_gen);
        gc_mark_grey(silver);
        gc_list_move(silver,&grey);
        conf.cycle_threshold += 1;
        // run mark grey phase
        while(grey != null){
            (grey->class->gc_mark_black)(grey);
            conf.cycle_threshold += 1;
            // check pause threshold
            gc_cycle_check
        }
    }else{
        // mark as white
        gc_mark_white(silver);
        gc_list_move(silver,&white);
        conf.cycle_threshold += 1;
    }
    // check pause threshold
    gc_cycle_check
}

// this checks if object contains reference object
bool gc_object_contains(gc_object* obj, gc_object* ref){
    conf.cycle_threshold += 1;
    gc_object** refs = (gc_object**)(obj+1); // start of refs array
    for(uint16_t i = 0; i < obj->refs_count; ++i){
        if(refs[i] == ref)
            return true;
    }
    return false;
}
{% endcodeblock %}

Finally we need to sweep all *white* objects, by moving them to *transparent* list. 
Next we just cleanup the *transparent* list again while also checking for the pause threshold using **gc_cycle_check** *macro* again.
And finally we mark cycle as full and end it using **gc_cycle_end** function.

{% codeblock lang:c %}
// sweep phase
if(white != null){
    // make object transparent
    gc_list_move_all(&white,&transparent);
    conf.cycle_threshold += 50;
    // check pause threshold
    gc_cycle_check
}

// transparent cleanup phase after cycle
while(transparent != null){
    (transparent->class->gc_finalize)(transparent);
    gc_object* obj = transparent;
    transparent = obj->gc_next;
    free(obj);
    conf.cycle_threshold += 11;
    conf.cycle_collected += 1;
    // check pause threshold
    gc_cycle_check
}

// set cycle full
conf.cycle_full = true;

return gc_cycle_end();
{% endcodeblock %}

Now that we have a garbage collector we need a way to test it to be sure it's working correctly.
To do that we'll create a micro DSL test language that will look like this. This language is inspired by a similar
language that I've created to test memory allocator in previous article. This language is fairly compact.
Note that putting an object into the slot doesn't means it automatically has a root reference. Objects that don't have root reference or
are referenced by root objects might get deleted after gc but our test compiler will check that kind of objects during compilation so we won't have any *Segmentation Fault*.
{% codeblock %}
# one line comment starts with # character
0=12   # allocate an object that can contain 12 references and set it to slot 0
+0     # increment root reference count of the object that is in slot 0 
-0     # decrement root reference count of the object that is in slot 0
0[0]=1 # set first reference of object in slot 0 to be object in slot 1
gc     # call garbage collector
0=2 gc # multiple statements are separated by white space or new line
{% endcodeblock %}

In order to actually write tests for that I've created *gctestgen.rb* *Ruby* script that can generate random test files of any size.
This test have iterations. In each iteration first we allocate objects, then we decrement root reference counts of some random objects from previous iteration.
After that we initialize references in objects randomly and run garbage collection. This way we simulate behaviour of an application allocating objects and running garbage collection.
{% codeblock lang:bash %}
# -n number of iterations
# -c objects per iteration
# -f maximum number of references object can have
# -r percentage of root objects
# -s percentage of survivors
# -d percentage of root objects to decrement on next iteration
$ ./gctestgen.rb -n 60 -c 1000 -f 32 -s 20 -r 10 -d 5 > test.t
{% endcodeblock %}

Now in order to compile our test I wrote a mini meta-compiler **gctest.rb** in *Ruby* that generates *.c* file for testing and links it against our garbage collector and creates executable binary **gctest**.
While compiling it also simulates garbage collection (simple full garbage collection each time) and counts expected number of objects to survive entire test. 
After that it runs that executable and compares it's output to expected count to proof correctness of the garbage collector. The source code is in repository so you could all study it.
The following initial configuration is provided for testing which is just 3 generations.

{% codeblock lang:c %}
// initialize gc object class
cls.gc_mark_black = &gc_object_mark_black;
cls.gc_contains = &gc_object_contains;
cls.gc_finalize = &garbage_collected_finalize;

// initialize gc configuration
gc_config config;
gc_gen_config c[3];
c[0].refresh_interval = 500000ull; // 0.5 millis
c[0].promotion_interval = 1000000ull; // 1 millis
c[1].refresh_interval = 2000000ull; // 2 millis
c[1].promotion_interval = 6000000ull; // 6 millis
c[2].refresh_interval = 250000000ull; // 15 millis
c[2].promotion_interval = 0; // not needed for last generation
config.gens_count = 3;
config.gens = c;
config.pause_threshold = 100; // 100 objects
config.max_pause = 200000000; // 200 milliseconds
gc_init(&config);
{% endcodeblock %}

{% codeblock lang:bash %}
$ ./gctest.rb ./test.t
total object allocations: 60000
expected number of objects to survive gc: 4606
running gc test: ./test.t
gc collected 850 objects took 1.02 millis [ 0/0 0/0 0/0 ]
gc collected 850 objects took 0.07 millis [ 142/0 142/0 0/0 ]
gc collected 850 objects took 0.07 millis [ 148/0 0/0 0/0 ]
...........................................................
gc collected 856 objects took 2.93 millis [ 147/0 0/301 0/0 ]
gc collected 850 objects took 0.09 millis [ 150/0 359/0 0/0 ]
test ended, took 246.68 millis
gc collected 0 objects took 200.00 millis [ 153/0 155/0 0/8561 ]
gc collected 0 objects took 200.00 millis [ 0/0 0/0 0/0 ]
gc collected 0 objects took 200.00 millis [ 0/0 0/0 0/0 ]
gc collected 3955 objects took 111.26 millis [ 0/0 0/0 0/0 ]
checked objects that survived: 4606
garbage collected 55394
actual survivors 4606
expected survivors match with actual survivors
total gc calls: 60
total time spent in gc: 53.74 millis
{% endcodeblock %}


As you can see test took 250ms to allocate 60000 objects. Garbage collector was called 60 times and overall took 53.74ms after which only 4606 objects survived. 
We wrote a *Garbage Collector*, finally. That's it folks, have a nice garbage collector hacking time!!!
![Space Dandy & Loli](http://i.imgur.com/OIcxN1t.png)

