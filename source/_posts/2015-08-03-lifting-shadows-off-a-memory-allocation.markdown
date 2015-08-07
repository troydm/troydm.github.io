---
layout: post
title: "Lifting Shadows off a Memory Allocation"
date: 2015-08-03 18:32:35 +0400
comments: true
categories: [linux, memory, c] 
---

Any sufficiently advanced technology is indistinguishable from magic or so they say. Today we are going to lift some shadows from
the very basic thing that is dynamic memory management in an application process. This knowledge is essential for anyone who wants to write his/her own *Programming Language* so gaining this knowledge is not an optional thing since it opens a whole new lever of understanding how dynamic memory is managed by application and is split between different applications. To do that we need to have some basic understanding of how the actual physical thing is used by operating system. For those who don't know anything about *Virtual Memory* or what *Memory Pages* are and which algorithms modern operating systems are using to manage those I suggest reading [Operating Systems Concepts](http://www.amazon.com/Operating-System-Concepts-Abraham-Silberschatz/dp/1118063333/) and for those who want to know all the guts of how *Linux* does this under the hood there is [Understanding Linux Kernel](http://www.amazon.com/Understanding-Linux-Kernel-Third-Daniel/dp/0596005652/). 
![Police Loli](http://i.imgur.com/VphDXNR.png)
<!--more-->

For the lazy rest to put it simply physical memory is split into fixed size blocks which are called *Memory Pages* and depending on the computer architecture and operating system can be 2kb 4kb 16kb 32kb or even 4mb in size. Anytime we request memory from operating system using some *syscall* it's best to have the size divisible by the actual operating operating system's *Memory Page* size. If used simply this tends to waste a lot of memory so hence we need a way to be more efficient with our requests and also we need to think of a way to recycle the memory requested before which isn't already needed by the application. All this opens a new area for exploration which is called Memory Allocation Algorithms and is quite a hot topic because not only we need to manage the recycling complexity but also concurrency and parallelism of the actual algorithms to be more efficient with modern multi-processor computers. 
Some notable examples of algorithm that do that and are used in real world are [Doug Lea's malloc](http://g.oswego.edu/dl/html/malloc.html), [TLSF](http://tlsf.baisoku.org/) and [jemalloc](http://www.canonware.com/jemalloc/), also there are much more out there. In order to explore some basic ideas about memory allocators we'll write the simplest algorithm commonly called power of 2. The essential idea behind it is to split blocks of memory into sizes divisible by 2 such as 32, 64, 128 and etc. By doing so we could reuse already used blocks by breaking them into more smaller onces. For example 128 block can be broken into 2 blocks of 64 and each of them can also further be broken to 32 sized blocks.
 
<canvas id="memoryBlock" width="400" height="75"></canvas> 
<script>
var c = document.getElementById("memoryBlock");
var ctx = c.getContext("2d");
ctx.font="normal 12pt Sans";
ctx.fillStyle = "#F8D7BB";
ctx.strokeStyle = "#000000";
ctx.fillRect(0,0,400,75);
ctx.moveTo(0,0);
ctx.lineTo(400,0);
ctx.stroke();
ctx.moveTo(0,0);
ctx.lineTo(0,75);
ctx.stroke();
ctx.moveTo(0,75);
ctx.lineTo(400,75);
ctx.stroke();
ctx.moveTo(400,0);
ctx.lineTo(400,75);
ctx.stroke();
ctx.moveTo(0,25);
ctx.lineTo(400,25);
ctx.stroke();
ctx.moveTo(200,25);
ctx.lineTo(200,75);
ctx.stroke();
ctx.moveTo(100,50);
ctx.lineTo(100,75);
ctx.stroke();
ctx.moveTo(0,50);
ctx.lineTo(200,50);
ctx.stroke();
ctx.moveTo(300,50);
ctx.lineTo(300,75);
ctx.stroke();
ctx.moveTo(200,50);
ctx.lineTo(400,50);
ctx.stroke();
ctx.moveTo(200,50);
ctx.lineTo(400,50);
ctx.stroke();
ctx.fillStyle = "#000000";
ctx.fillText("128",185,18);
ctx.fillText("64",92,43);
ctx.fillText("64",292,43);
ctx.fillText("32",46,67);
ctx.fillText("32",148,67);
ctx.fillText("32",246,67);
ctx.fillText("32",338,67);
</script>
In general this tends to waste a lot of memory and introduces memory fragmentation. In worst case scenario half of all requested memory is wasted but usually in real world cases the actual number is near 25%. The memory wasting cost has computing performance gain as the algorithm itself is not hard to implement and works faster than more complex but conservative ones. Hence we are going to write a simple memory allocator also commonly called *malloc* since this is the library function name used by standard *C* library for allocating dynamic memory of variable size for the application process. We are going to implement our version of that function and related functions. Our target platform will be *Linux* and *gcc* compiler since we aren't going to bother with portability as this will take much more time and effort. Final version of source code can be browsed in [mymalloc](https://github.com/troydm/mymalloc) repo but I strongly encourage you to write the whole thing from scratch yourself just to better understand the actual processes under the hood.

First let's start with function definitions that we need to reimplement. Create a file called **mymalloc.h**. We'll also define some functions for debug/stats purposes as those will be helpful during development process. Note: all this functions are defined by glibc in linux so gcc will complain about them being redefined during compilation but you can ignore those messages.

{% codeblock lang:c %}
void* calloc(size_t nmemb, size_t size);
void* malloc(size_t s);
void* realloc(void* p, size_t ns);
void free(void* p);
// for debug use only
void print_block_info(void* p);
void print_freelist();
{% endcodeblock %}

Now let's create a file called **mymalloc.c** and define the actual functions. We'll start with some initial values which are provided by operating system. The defined values are self explaining however I need to clarify the choice with 32 bytes for minimum block size. Since we'll manage memory blocks as doubly linked list nodes, each memory block needs to have space for at least two pointers and one size value which can't be more than the pointer size so that's 3 pointers for each node. Maximum size for 3 pointers for 64-bit operating system is 8 bytes so it's overall 24 bytes however block size needs to be power of 2 value and nearest size is 2^5 hence 32.
{% codeblock %}
// for code clarity for pointers we use null instead of 0
#define null 0

// initial values
#define PAGE_SIZE (sysconf(_SC_PAGESIZE))
#define MIN_BLOCK_SIZE 32 // bytes
{% endcodeblock %}

We need a place to save free memory blocks and a way to manage them so let's first describe what memory block is and define initial doubly linked list for it that will contain *start* and *end* nodes pointing at each other. Those nodes won't be used and are there just so that our code won't contain conditional *if* statements when handling nodes.


<canvas id="freelist" width="600" height="75"></canvas> 
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
    ctx.fillText(text,x+(w/2)-(tl/4),y+(h/2)+4);
    ctx.fillStyle = fs;
}

function drawArrow(ctx,x1,y1,x2,y2){
    ctx.moveTo(x1,y1);
    ctx.lineTo(x2,y2);
    ctx.stroke();

    ctx.moveTo(x1,y1);
    ctx.lineTo(x1+3,y1-4);
    ctx.stroke();
    ctx.moveTo(x1,y1);
    ctx.lineTo(x1+3,y1+4);
    ctx.stroke();

    ctx.moveTo(x2,y2);
    ctx.lineTo(x2-3,y2-4);
    ctx.stroke();
    ctx.moveTo(x2,y2);
    ctx.lineTo(x2-3,y2+4);
    ctx.stroke();
}

var c = document.getElementById("freelist");
var ctx = c.getContext("2d");
ctx.font="normal 12pt Sans";
ctx.fillStyle = "#F8D7BB";
ctx.strokeStyle = "#000000";

drawRect(ctx,"start",0,20,100,50);
drawRect(ctx,"memory block",150,20,200,50);
drawRect(ctx,"end",400,20,100,50);
drawArrow(ctx,80,45,165,45);
drawArrow(ctx,340,45,415,45);

ctx.fillStyle = "#000000";
ctx.fillText("Freelist",2,13);
</script>

{% codeblock lang:c %}
// memory block structure
typedef struct memory_block_t {
    size_t size;
    struct memory_block_t* prev;
    struct memory_block_t* next;
} memory_block;

// free memory block list
static memory_block freelist[] = { { 0, null, &(freelist[1]) },  { 0, &(freelist[0]), null} };
#define freelist_start (freelist[0].next)
#define freelist_begin (&(freelist[0]))
#define freelist_end (&(freelist[1]))
{% endcodeblock %}

Note that when memory block will be used by an application it needs to have only size value reserved because we don't need doubly linked pointers as it's not in a free block list. Thus we need a method to reserve memory block's size field when we return it to the application. In order to ease interaction with memory block structure and it's pointers let's define some useful macros that will help us elevate the task of specifying casts manually. Pointer arithmetic can be sometimes daunting and puzzling in *C* but if used correctly it's very powerful feature and macro system helps us do exactly that. We also need an easy way to link two memory blocks with each other and even sometimes replace those links so we'll define some macros for that kind of operations too.
{% codeblock %}
// useful macros
#define byte_ptr(p) ((uint8_t*)p)
#define shift_ptr(p,s) (byte_ptr(p)+s)
#define shift_block_ptr(b,s) ((memory_block*)(shift_ptr(b,s)))
#define block_data(b) (shift_ptr(b,sizeof(size_t)))
#define data_block(p) (shift_block_ptr(p,-sizeof(size_t)))
#define block_end(b) (shift_block_ptr(b,b->size))

#define block_link(lb,rb) \
    rb->prev = lb; \
    lb->next = rb;

#define block_link_left(lb,b) \
    block_link(b->prev,lb) \
    block_link(lb,b)

#define block_link_right(b,rb) \
    block_link(rb,b->next) \
    block_link(b,rb)

#define block_unlink(b) \
    block_link(b->prev,b->next)

#define block_unlink_right(b) \
    block_unlink(b->next);

#define block_replace(b,nb) \
    block_link(b->prev,nb) \
    block_link(nb,b->next)
{% endcodeblock %}

Let's start with *malloc* function, first of all let's correct initial size and add to it *sizeof(size_t)* since each allocated by an application block will preserve it.
{% codeblock lang:c %}
void* malloc(size_t s){
    // check for 0 size
    if(s == 0)
        return null;
    // add size of size_t as we need to save size of memory block
    s += sizeof(size_t);
    // find suitable memory size
    size_t ns = find_optimal_memory_size(s);
{% endcodeblock %}
Next as you can see we need to find an optimal size for requested size *s*. We'll use a famous trick with bitwise shifting value to right in order to get optimal size for our memory block starting with *MIN_BLOCK_SIZE*. So the sizes in *while* loop will go from 32, 64, 128, 256 and further.
{% codeblock lang:c %}
// find optimal memory block size for size s
static inline size_t find_optimal_memory_size(size_t s){
    size_t suitable_size = MIN_BLOCK_SIZE;
    
    while(suitable_size < s)
        suitable_size = suitable_size << 1;

    return suitable_size;
}
{% endcodeblock %}

Next we need to handle large memory blocks. We'll be allocating those using *mmap* *syscall* which gives us an arbitary memory block for requested size usually rounded to page size. In worst case we are loosing *PAGE_SIZE*-1 bytes, however for large memory blocks which we are going to handle separately we can ignore it because block might be resized using *mremap* *syscall* in the future reclaiming page which wasn't used entirely before. 
Note: we've also added definition of MMAP_SIZE to our initial value definitions and set it equal to 1 MB.

{% codeblock lang:c %}
    // if size is greater than or equals MMAP_SIZE we are going to use mmap
    if(ns >= MMAP_SIZE){
        void* m = mmap(NULL,s,PROT_READ|PROT_WRITE,MAP_PRIVATE|MAP_ANONYMOUS,-1,0);    
        if(m == MAP_FAILED)
            return null;
        memory_block* b = (memory_block*)m;
        b->size = s;
        lock
        mmap_size += s;
        unlock
        return block_data(b);
    }
{% endcodeblock %}
As you've already noticed we've used strange *lock* and *unlock* macros there. That's because we'll also be counting overall *mmap* blocks in *mmap_size* variable purely for statistical purposes. However since *malloc* function can be called from different threads simultaneously we need a way to synchronize modifications to this variable. And not only this variable can be modified simultaneously. Unfortunately as we are striving for most efficiency we can't use immutable data structures for *freelist* and we need a way to concurrently modify it so that in the end all modifications would be consistent. Let's use atomic [spinlock](https://en.wikipedia.org/wiki/Spinlock) for all our concurrency needs. For a description of what atomic spinlock is try reading [Marek's Reinventing Spinlocks](https://idea.popcount.org/2012-09-12-reinventing-spinlocks/) as the code below was shamelessly copied from there. Also instead of simple freelist we could use lock-free doubly linked list data structure but since we are focusing our efforts on memory allocation I won't describe it in here.
{% codeblock lang:c %}
// locking
volatile bool locked = 0;

static inline void spinlock(){
    if (!__sync_bool_compare_and_swap(&locked, 0, 1)){ 
        int i = 0;
        do { 
            if (__sync_bool_compare_and_swap(&locked, 0, 1))
                break; 
            else{
                if(i == 10){
                    i = 0;
                    sched_yield(); 
                }else
                    ++i;
            }
        } while (1); 
    }
}

#define lock spinlock();

#define unlock \
    __asm__ __volatile__ ("" ::: "memory"); \
    locked = 0;
{% endcodeblock %}

Let's continue our effort with *malloc* function. Now we need to handle blocks with size less than *MMAP_SIZE*

{% codeblock lang:c %}
    lock
    // find free memory block
    memory_block* block = find_suitable_block(ns);
    if(block != null){
        unlock
        // shift pointer into data block pointer
        return block_data(block);
    }            
{% endcodeblock %}

In order to find suitable block from memory list we'll just look through it.

{% codeblock lang:c %}
// find suitable memory block for size s
static inline memory_block* find_suitable_block(size_t ns){

    memory_block* b = freelist_start;
    while(b != freelist_end){
        if(b->size >= ns){
            return split_memory_block(b,ns);                    
        }
        b = b->next;    
    }

    return null;
}
{% endcodeblock %}

Now what if block is sufficient for our needs. We need to split it or return it whole if it's needed entirely. The half left will be added back to *freelist* if *remainder* is more than or equals *MIN_BLOCK_SIZE* since there is no way we could add a block with less size.
{% codeblock lang:c %}
// split memory block into 2 pieces one of size s and the other is remainder e.g. memory_block_size-s
// if remainder is less than MIN_BLOCK_SIZE we just take whole block
static inline memory_block* split_memory_block(memory_block* b, size_t s){
    size_t remainder = b->size - s;
    if(remainder >= MIN_BLOCK_SIZE){
        memory_block* nb = shift_block_ptr(b,s);
        nb->size = remainder;
        block_replace(b,nb);
        b->size = s;
    }else{
        block_unlink(b);
    }
    return b;
}
{% endcodeblock %}

But wait, what if there are no free memory blocks? Let's allocate some memory from heap using *sbrk* *syscall*. Now the difference with *sbrk* versus *mmap* is that *sbrk* allocates continuous virtual memory or simply speaking grows heap outward (note: stack grows inward and heap outward so when those two meet each other ka-boom!). For those of you who don't know what heap is try reading [this](http://www.geeksforgeeks.org/memory-layout-of-c-program/). Unfortunately *sbrk* is slower than *mmap* performance-wise, but we need a specific property of memory allocated using *sbrk* so we'll be using it exclusively for that. Each newly allocated block has higher memory address than previous one and both of them are adjacent. We'll use this property to always have a sorted freelist in order to connect adjacent memory blocks more easily and quicker. This is our way to fight memory fragmentation. Also in order to minimize number of *sbrk* calls we'll allocate memory by larger chunks specified in *ALLOC_SIZE* instead of just number of pages we need.
{% codeblock lang:c %}
    // no free memory blocks found
    // we need to allocate new one that would be suitable for our needs using sbrk
    size_t pages_size = ((ns/PAGE_SIZE)+1)*PAGE_SIZE;
    if(pages_size < ALLOC_SIZE)
        pages_size = ALLOC_SIZE;
    // allocate memory with sbrk
    void* p = sbrk(pages_size);    
    if(p == (void*)-1){
        unlock
        return null;
    }
{% endcodeblock %}

Next after allocating new block using *sbrk* we just split it and add back to *freelist*. We are also counting the overall heap size allocated by *sbrk* calls just for stats. Also we save *heap_start* and *heap_end* in order to distinguish memory blocks allocated using *sbrk* from *mmap* blocks.
{% codeblock lang:c %}
    block = (memory_block*)p;
    heap_size += pages_size;
    block->size = ns;
    heap_end = shift_block_ptr(block,pages_size);
    heap_start = shift_block_ptr(heap_end,-heap_size);
    ns = pages_size - ns;
    if(ns >= MIN_BLOCK_SIZE){
        memory_block* b = shift_block_ptr(block,block->size);
        b->size = ns;
        add_block(b);
    }
    unlock

    return block_data(block);
{% endcodeblock %}

Now the only thing left is to add the free block to the *freelist*. This might sound trivial but we need to handle a little bit more complexity than there might seem. First of all we need to insert blocks in sorted order. Next after inserting free block we need to merge it with adjacent blocks near on the left and right from it, if those blocks are continuously allocated in virtual memory space.
{% codeblock lang:c %}
// add block to free list
static inline void add_block(memory_block* block){
    if(freelist_start != freelist_end){
        // find superseding memory block
        // and insert current one before it
        memory_block* b = freelist_start;
        while(1){
            // superseding memory block will have higher memory address
            if(b > block){
                block_link_left(block,b);
                break;            
            }
            // check if we hit end
            if(b->next == freelist_end){
                block_link_right(b,block);
                break;            
            }else{
                b = b->next;
            }
        }

        // merge adjacent blocks
        bool merged = true;
        while(merged){
            // merge right adjacent block
            if(block_end(block) == block->next){
                block->size += block->next->size;
                block_unlink_right(block);
                continue;
            }                    
            // merge left adjacent block
            if(block_end(block->prev) == block){
                block = block->prev;
                block->size += block->next->size;
                block_unlink_right(block);
                continue;
            }
            merged = false;
        }
    }else{
        // add first memory block
        memory_block* b = freelist_begin;
        block_link_right(b, block);
    }
}
{% endcodeblock %}

That's it with *malloc* now let's write *free*. As you can see below we check if memory block is allocated using *mmap* and *munmap* it. Otherwise we add it back to *freelist*.

{% codeblock lang:c %}
// mmap
#define is_mmap_block(b) (!(heap_start <= b && b < heap_end))

void free(void* p){
    // check for null pointer
    if(p == null)
        return;

    // shift pointer back into memory block pointer
    memory_block* b = data_block(p);

    lock
    if(is_mmap_block(b)){
        mmap_size -= b->size;
        unlock
        munmap(b,b->size);        
        return;    
    }

    // add removed block into freelist
    add_block(b);
{% endcodeblock %}
Finally we are checking the end of heap and give back free memory to operating system using *sbrk* with negative size if the size is at least *GIVE_BACK_SIZE* which we've also added to initial values.
{% codeblock lang:c %}
    // give last memory block that isn't needed back to the operating system
    b = freelist_end->prev;
    if(block_end(b) == heap_end){
        intptr_t inc = b->size;
        if(inc >= GIVE_BACK_SIZE){
            if(b == heap_start){
                if(inc > GIVE_BACK_SIZE){
                    inc = inc - GIVE_BACK_SIZE;
                    sbrk(-inc);
                    b->size = GIVE_BACK_SIZE;
                }
            }else{
                block_unlink(b);
                sbrk(-inc);
                heap_size -= inc;
                heap_end = shift_block_ptr(heap_end,-inc);
                heap_start = shift_block_ptr(heap_end,-heap_size);
            }
        }
    }
    unlock
}
{% endcodeblock %}

Next after *free* let's implement *realloc*. As you can see it mostly repeats malloc, however we are using *mremap* for *mmap* blocks since those are already allocated and we need to resize them.
{% codeblock lang:c %}
void* realloc(void* p, size_t s){
    if(p == null){
        return malloc(s);
    }else if(s == 0){
        free(p);
        return null;
    }
    
    // shift pointer back into memory block pointer
    memory_block* b = data_block(p);

    // find out which new optimal size we need
    size_t ss = s+sizeof(size_t);
    size_t ns = find_optimal_memory_size(ss);

    // if memory is mmap we need to use mremap
    lock
    if(is_mmap_block(b)){
        mmap_size -= b->size;
        mmap_size += ss;
        unlock
        void* p = mremap(b,b->size,ss,MREMAP_MAYMOVE);
        if(p == MAP_FAILED)
            return null;
        b = (memory_block*)p;
        b->size = ss;
        return block_data(b);
    }
{% endcodeblock %}

Next we check if block already contains the necessary size or the size can be obtained by connecting adjacent blocks.
{% codeblock lang:c %}
    if(ns < MMAP_SIZE){
        // check if size is already sufficient
        if(b->size >= ns){
            unlock
            return p;
        }

#ifdef MERGE_ADJ_ON_REALLOC
        // try merging with adjacent blocks
        memory_block* nb = merge_with_adjacent_block(b,ns);
        if(nb != null){
            unlock
            // shift pointer into data block pointer
            return block_data(nb);
        }
#endif
    }
    unlock
{% endcodeblock %}

Merging with adjacent block is quite complex operation thus we make it configurable so that if necessary we can always disable it. We handle both left and right adjacent cases. Left one is more complex and right one is simpler. Also we need to split block before connecting it with our initial one if the size has *remainder*, hence the complexity.
{% codeblock lang:c %}
// merge with adjacent block so that overall new size would be s
static inline memory_block* merge_with_adjacent_block(memory_block* block, size_t s){

    if(freelist_start == freelist_end)
        return null;

    memory_block* b = freelist_start;
    memory_block* be = block_end(block);
    do {
        
        // left adjacent
        // code is slightly more complex as we need to copy data over
        if(block_end(b) == block){
            if((b->size + block->size) >= s){
                size_t remainder = (b->size + block->size) - s;
                // we need to backup block pointers as they might be overwritten by memcpy
                memory_block* temp_prev = b->prev;
                memory_block* temp_next = b->next;
                memcpy(block_data(b),block_data(block), block->size - sizeof(size_t));
                if(remainder >= MIN_BLOCK_SIZE){
                    b->size = s;
                    memory_block* nb = shift_block_ptr(b,s);
                    nb->size = remainder;
                    block_link(temp_prev,nb);
                    block_link(nb,temp_next);
                }else{
                    // unfortunetly here we can't use b->prev as 
                    // it might have been overwritten by memcpy
                    // so we need to remove remaining block entirely using temporary pointers
                    block_link(temp_prev,temp_next);
                }
                return b;
            }
        }

        // right adjacent
        if(be == b){
            if((block->size + b->size) >= s){
                size_t remainder = (block->size + b->size) - s;
                if(remainder >= MIN_BLOCK_SIZE){
                    memory_block* nb = shift_block_ptr(block,s);
                    nb->size = remainder;
                    block_replace(b,nb);
                    block->size = s;
                }else{
                    block_unlink(b);
                }
                return block;
            }
            break;
        }

        b = b->next;
    } while(b != freelist_end && b >= be);

    return null;
}
{% endcodeblock %}

Finally we handle the case when there is no memory to merge and no to resize. We just allocate a new memory and copy data over into it. Afterwards we deallocate the previous memory block.
{% codeblock lang:c %}
    void* np = malloc(s);
    if(np != null){
        // copy old data block into new one
        memcpy(np,p,s > b->size ? b->size : s);

        // free old data block
        free(p);

        // return newly allocated block
        return np;
    }
    
    return null;
}
{% endcodeblock %}

Main work is done, remaining are just supplementary functions which are self explanatory.
{% codeblock lang:c %}
void* calloc(size_t nmemb, size_t size){
    size = nmemb*size;
    void* p = malloc(size);
    if(p != null)
        memset(p,0,size);
    return p;
}

void print_block_info(void* p){
    // shift pointer back into memory block pointer
    memory_block* b = data_block(p);
    lock
    print_block(b);
    unlock
}

void print_freelist(){
    lock
    printf("[heap size %d mb mmap_size %d mb, ",(heap_size/(1024*1024)),(mmap_size/(1024*1024)));
    printf("freelist {");
    memory_block* b = freelist_start;
    while(b != freelist_end){
        printf(" -> %p[%u|%p|%p]",b,b->size,b->prev,b->next);    
        // detect infinite loop if any
        if(b == b->next){
            printf(" -> infinite loop\n");
            break;
        }
        b = b->next;
    }
    unlock
    printf(" }\n");
}
{% endcodeblock %}

Now to test/benchmark this whole thing I wrote a little application which is called *memsim* and is in [mymalloc](https://github.com/troydm/mymalloc/) repo. It reads memory allocation simulation file without requesting any dynamic memory from filesystem and calls malloc, realloc, free and user provided custom stats function while counting timings and overall runtime without using any heap memory, internally allocating memory on stack using *alloca* function, so that it's workings don't affecting the testing process. In order to generate a complex memory simulation (which is just a simple text file) I wrote *genrandms* application that just does that. Now a little benchmarks to compare our malloc with glibc *malloc* on some intense scenario. *mymemsim* and *sysmemsim* are *memsim* executables linked against our malloc implementation and glibc malloc implementation.

{% codeblock lang:bash %}
# Single threaded scenario repeat 50 times   
$ ./mymemsim -t 1 -r 50 test.ms 
memory simulation took 3457ms
$ ./sysmemsim -t 1 -r 50 test.ms 
memory simulation took 15157ms
{% endcodeblock %}
Now let's try a multithreading benchmark
{% codeblock lang:bash %}
# Multi threaded 20 threads each repeat scenario 20 times
$ ./mymemsim -t 20 -r 20 test.ms 
memory simulation took 64268ms
$ ./sysmemsim -t 20 -r 20 test.ms 
memory simulation took 117117ms
{% endcodeblock %}

Unfortunately our malloc implementation is not as multithread friendly as it can be. We literally have one single global spinlock and we always lock it when we allocate memory. We can improve on that by splitting *freelist* into different partitions. This will sacrifice memory fragmentation fighting however will improve scalability over multiple processors as we can have different spinlocks for each partition and won't have to wait between them. The implementation is more complex and can be studied in [mymalloc](https://github.com/troydm/mymalloc) repo **mysmalloc.c** file despite overall idea being simple. Let's benchmark it again.
{% codeblock lang:bash %}
# Multi threaded 20 threads each repeat scenario 20 times
$ ./mymemsim -t 20 -r 20 test.ms 
memory simulation took 69510ms
$ ./mysmemsim -t 20 -r 20 test.ms 
memory simulation took 43322ms
{% endcodeblock %}
As you can see we've improved performance by partially sacrificing fragmentation, improving scalability, and partially increasing memory usage. Further improvements and experiments are possible as possibilities are countless however our time has run out so that's it for today folks! Have a nice memory hacking time!
![Smoking is bad for health](http://i.imgur.com/qBrTRxO.png)
