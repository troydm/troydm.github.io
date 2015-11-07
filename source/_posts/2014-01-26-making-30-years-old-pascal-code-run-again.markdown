---
layout: post
title: "Making 30 years old Pascal code run again"
date: 2014-01-26 22:48
comments: true
categories: [pascal, fpc, prolog] 
---

TL;DR *Fixing some really old Prolog implementation written in Pascal*

Recently I've been interested in [Logic Programming](https://en.wikipedia.org/wiki/Logic_programming), notably in learning [Prolog](https://en.wikipedia.org/wiki/Prolog)
so I'm in a process of reading two great books, [Programming for Artificial Intelligence](http://www.amazon.com/Prolog-Programming-Artificial-Intelligence-Bratko/dp/0201403757/) and 
[The Art of Prolog](http://www.amazon.com/The-Art-Prolog-Programming-Techniques/dp/0262192500/). If you want to get a quick feel of *Prolog* I recommend you take a look at 
Bernardo Pires's [Gentle Introduction to Prolog](https://bernardopires.com/2013/10/try-logic-programming-a-gentle-introduction-to-prolog/) and [Prologomenon](http://prologomenon.wordpress.com/) blog.
To put it simply *Prolog* is all about logic, deduction and backtracking 

![Sherlock Loli](http://i.imgur.com/Mwlmvyk.jpg)

While browsing [/r/prolog](http://reddit.com/r/prolog) I've stumbled upon [Prolog for Programmers](https://sites.google.com/site/prologforprogrammers/) originally published in 1985, an old book indeed
and honestly sometimes hard to follow. I can't recommend it as a starter book about *Prolog* but it's still quite interesting to read. However it has a whole two chapters describing implementation of 
*Prolog* interpreter which is quite a complex task and sparkled my interest in continuing reading this book. Authors provide source code of two version of Prolog interpreter, 
the one they originally wrote in *Pascal* back in 1983 and it's port to *C* in 2013 which, as stated on their website, was done because they couldn't compile old *Pascal* code 
with [Free Pascal Compiler](http://www.freepascal.org/) and because... well *Pascal* is quite out of fashion nowadays. Couldn't compile?! Well, challenge accepted!

<!-- more -->

We'll start with *toy.p* (original source code) and *syskernel* (boot file written in toy's original prolog syntax) file. Let's create a *Makefile* and try compiling the software with *fpc*.

{% codeblock %}
############
FPC=/usr/bin/fpc
FPCFLAGS=-O3 
############
all: toy

toy: toy.p 
        $(FPC) $(FPCFLAGS) $<
        
############
clean:
        rm -f toy*.o
                
distclean: clean
        rm -f toy
{% endcodeblock %}

Let's try running *make*

{% codeblock %}
/usr/bin/fpc -O3  toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(260,1) Error: Goto statements aren't allowed between different procedures
toy.p(306,1) Error: Goto statements aren't allowed between different procedures
toy.p(348,18) Error: Incompatible type for arg no. 2: Got "Array[1..35] Of Char", expected "LongInt"
toy.p(351,20) Error: Incompatible type for arg no. 2: Got "Array[1..35] Of Char", expected "LongInt"
toy.p(453,21) Fatal: Syntax error, "identifier" expected but "STRING" found
Fatal: Compilation aborted
{% endcodeblock %}

First two errors are *goto* statement related, let's see the source code.
{% codeblock lang:pascal %}

label 1, 2;  (* error halt & almost-fatal error recovery only *)

procedure halt;
(* this might be implementation-dependent *)
begin   writeln;   writeln ( ' ******toyprolog aborted******');
    goto 1                                                                                                                                                                                                                            
end;   

procedure errror ( id : errid );
begin   writeln;
        write ( ' ++++++error : ' );
        ....

        if id in [ ctovflw, protovflw, loadfile, sysinit, usereof ] then halt
        else goto 2 
end;

begin (*********** toy prolog ************)
        initvars;
        loadsyskernel;
2:      repeat  readtogoal ( goalstmnt, ngoalvars );
                resolve ( goalstmnt, ngoalvars )
        until terminate;
1:      closefile ( true );   closefile ( false )
end.

{% endcodeblock %}

Well, apparently back in the 80's you could jump between procedures in *Pascal* using *goto* statements, some primitive try/catch mechanism. 
Now that's a neat thing but unfortunetly no, you can't do that anymore. Let's change this code to use exceptions. One thing to note we can substitute
*goto 1* with *halt(1)* which will make our program exit on critical error. Also we'll need to add *-S2* flag to *FPCFLAGS* variable in *Makefile* since we'll
be using exceptions which is related to classes mechanism of *Pascal*. And just to be sure we don't override any system procedures we'll rename our *halt* to
*haltsys*

{% codeblock lang:pascal %}

use sysutils;

type 
        ....
        error = class(Exception);

procedure haltsys;
(* this might be implementation-dependent *)
begin   
    writeln;   writeln ( ' ******toyprolog aborted******');
    halt(1)
end;

procedure errror ( id : errid );
begin   writeln;
        write ( ' ++++++error : ' );
        ....
        if id in [ ctovflw, protovflw, loadfile, sysinit, usereof ] then haltsys
        else raise error.create( 'error' )
end;

(*********** toy prolog ************)
begin 
    initvars;
    loadsyskernel;
    repeat 
        try 
            readtogoal ( goalstmnt, ngoalvars );
            resolve ( goalstmnt, ngoalvars )
        except on error do end;
    until terminate;
    closefile ( true );   closefile ( false )
end.

{% endcodeblock %}

Let's try running *make* again

{% codeblock %}
/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(349,18) Error: Incompatible type for arg no. 2: Got "Array[1..35] Of Char", expected "LongInt"
toy.p(352,20) Error: Incompatible type for arg no. 2: Got "Array[1..35] Of Char", expected "LongInt"
toy.p(454,21) Fatal: Syntax error, "identifier" expected but "STRING" found
{% endcodeblock %}

Good, now *goto* related error is gone and next we have is something bizzare, let's see the source code.

{% codeblock lang:pascal %}

procedure openfile ( name : ctx;  forinput : boolean );
(* open the file (si or so according to 2nd parameter)
   whose name is given by a string in character table.
   a file is opened rewound. if a previous file is open, it is closed.  *)
const  ln = 35; (* for RSX-11 *)
var   nm : array [ 1..ln ] of char;   k : 1..ln;
begin   k:= 1;
	while ( k <> ln ) and ( ct [ name ] <> chr ( eos ) ) do begin
		nm [ k ] :=  ct [ name ];   name:= name + 1;   k:= k + 1
	end;
	if ct [ name ] <> chr ( eos ) then errror ( longfilename );
	for k:= k to ln do nm [ k ] := ' ';
	closefile ( forinput );               (* only 1 file per stream *)
	if forinput then begin
		reset ( si, nm );    seeing:= true
	end
	else begin
		rewrite ( so, nm );    telling:= true;
		solinesize:= 0
	end
end (*openfile*);

{% endcodeblock %}

Well apparently *reset* and *rewrite* procedures don't accept file name as second argument anymore, 
instead we need to first *assign* file name and then open it for reading.

{% codeblock lang:pascal %}

procedure openfile ( name : ctx;  forinput : boolean );
(* open the file (si or so according to 2nd parameter)
   whose name is given by a string in character table.
   a file is opened rewound. if a previous file is open, it is closed.  *)
const  ln = 35; (* for RSX-11 *)
var   nm : array [ 1..ln ] of char;   k : 1..ln;
begin   k:= 1;
	while ( k <> ln ) and ( ct [ name ] <> chr ( eos ) ) do begin
		nm [ k ] :=  ct [ name ];   name:= name + 1;   k:= k + 1
	end;
	if ct [ name ] <> chr ( eos ) then errror ( longfilename );
	for k:= k to ln do nm [ k ] := ' ';
	closefile ( forinput );               (* only 1 file per stream *)
	if forinput then begin
            assign(si, nm);
            reset ( si );    seeing:= true
	end
	else begin
            assign(so, nm);
            rewrite ( so );    telling:= true;
            solinesize:= 0
	end
end (*openfile*);

{% endcodeblock %}

Let's try running *make* again

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(456,21) Fatal: Syntax error, "identifier" expected but "STRING" found

{% endcodeblock %}

Let's see what we have here

{% codeblock lang:pascal %}

function charlast ( string : ctx )  : ctx;
(* locate the last character (except eos) of this string *)
begin   while  ct [ string ] <> chr ( eos )  do string:= string + 1;
        charlast:= string - 1     (*correct because lowest string not empty*)
end;

{% endcodeblock %}

We can't use *string* as identifier since it's used as a type keyword nowadays so let's change that

{% codeblock lang:pascal %}

function charlast ( str : ctx )  : ctx;
(* locate the last character (except eos) of this str *)
begin   while  ct [ str ] <> chr ( eos )  do str:= str + 1;
        charlast:= str - 1     (*correct because lowest str not empty*)
end;

{% endcodeblock %}

Let's try running *make* again

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(922,7) Fatal: Syntax error, "identifier" expected but "is" found

{% endcodeblock %}

Ahh, same identifier problem but now with *is* keyword, let's quickly change that too.
Next is same problem but now with *class* keyword, okey fixed that one too. And again *is* keyword related problems, fixed.

And now running *make* again

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(1951,36) Error: Identifier not found "success"
toy.p(1956,10) Error: Identifier not found "id"
toy.p(1957,18) Error: Constant and CASE types do not match
toy.p(1957,27) Error: Identifier not found "success"
toy.p(1958,8) Error: Constant and CASE types do not match
toy.p(1959,18) Error: Constant and CASE types do not match
toy.p(1960,18) Error: Constant and CASE types do not match
........

{% endcodeblock %}

ZOMFG, tons of errors :( no worries no worries let's see the code!

{% codeblock lang:pascal %}

procedure sysroutcall (* ( id : sysroutid;  var success, stop : boolean ) *);
(* perform a system routine call *)
var   k : nsysparam;
begin   syserror:= false;   success:= true;     (* might change yet *)
	for k:= 1 to getarity ( ccall ) do begin
		spar [ k ] :=  argument ( ccall, ancenv, k );
		if isint( spar [ k ] ) then sparv [ k ]:= intval( spar [ k ] )
	end;
	case id of
	idfail          : success:= false;      (* keep this as first  *)
	idtag ,
	idcall          : ;             (* never called ! (cf. control) *)
	idslash         : slash;
	idtagcut        : tagcut ( success );
	idtagfail       : tagfail ( success );
	idtagexit       : tagexit ( success );
	idancestor      : ancestor ( success );
     .........
	end (*case*)
end (*sysroutcall*);

{% endcodeblock %}

Ehh... why is the procedure argument block commented? Wait we have another *sysroutcall*
defined in a file previously which is... 

{% codeblock lang:pascal %}

procedure sysroutcall ( id : sysroutid;  var success, stop : boolean );
forward; (*----------------------------------------------------------*)

{% endcodeblock %}

Ahh it's just a forward definition, well aparently nowadays if you are defining a forward
definition that doesn't mean you don't need to specify procedure arguments again. 
Let's uncomment the arguments and try running *make* again.

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(2132,33) Fatal: Syntax error, ":" expected but ";" found
Fatal: Compilation aborted

{% endcodeblock %}

Let's see the code

{% codeblock lang:pascal %}

function rdterm (*  : integer *);
(* read a term and return a prot for a non-var or a negated offset for a var.
   sequences processed recursively to allow proper ground prot treatment.   *)
var   sign : -1..1;   varoff : varnumb;   prot : integer;   dot : protx;
begin   skipbl;

{% endcodeblock %}

Function result type is commented, probably a typo, let's uncomment it and run *make* again.

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(2138,22) Warning: Function result variable does not seem to initialized
toy.p(2280,26) Error: Incompatible type for arg no. 2: Got "Constant String", expected "LongInt"
toy.p(2314,9) Error: Label used but not defined "2"
toy.p(2314,9) Fatal: Syntax error, ";" expected but "REPEAT" found
Fatal: Compilation aborted

{% endcodeblock %}

That's a weird error. Let's examine the code.

{% codeblock lang:pascal %}

function rdterm   : integer;
(* read a term and return a prot for a non-var or a negated offset for a var.
   sequences processed recursively to allow proper ground prot treatment.   *)
var   sign : -1..1;   varoff : varnumb;   prot : integer;   dot : protx;
begin   skipbl;
        writeln('rdterm ',cch);
	if cch = '(' then begin         (* eg.  a . (b . c) . d *)
		rd;   prot:= rdterm;   skipbl;
		if cch <> ')' then synterr;   rd
	end
	else
	if cch = '_' then begin                 (* a dummy variable *)
		rd;   prot:= dumvarx            (* treated as non-var here *)
	end
	else
	if cch = ':' then begin                 (* a variable *)
		rd;   varoff:= rddigits;   prot:= - varoff;
		if varoff + 1 > nclvars then nclvars:= varoff + 1
	end
	else
	if ( cch = '+' ) or ( cch = '-' ) or ( cc [cch] = cdigit ) then begin
		if cch = '-' then sign:= -1 else sign:= 1;
		if cc [ cch ] <> cdigit then rd;
		(* number itself processed as positive :  this
		   causes loss of smallest integer in two's complement *)
		prot:= newintprot ( sign * rddigits )
	end
	else begin
            prot:= rdnonvarint;
            skipbl
        end;
	if cch <> '.' then rdterm:= prot
	else begin                      (* a sequence, as it turns out *)
		dot:= initprot ( std [atmdot] );   mkarg ( dot, car, prot );
		rd;   skipcombl;
		mkarg ( dot, cdr, rdterm );   rdterm:= wrapprot ( dot )
	end
end (*rdterm*);

{% endcodeblock %}

Seems like *fpc* can't distinguesh between a recursive function call and a function result value, 
so let's add some brackets for function calls.

{% codeblock lang:pascal %}

function rdterm   : integer;
(* read a term and return a prot for a non-var or a negated offset for a var.
   sequences processed recursively to allow proper ground prot treatment.   *)
var   sign : -1..1;   varoff : varnumb;   prot : integer;   dot : protx;
begin   skipbl;
        writeln('rdterm ',cch);
	if cch = '(' then begin         (* eg.  a . (b . c) . d *)
		rd;   prot:= rdterm();   skipbl;
		if cch <> ')' then synterr;   rd
	end
	else
	if cch = '_' then begin                 (* a dummy variable *)
		rd;   prot:= dumvarx            (* treated as non-var here *)
	end
	else
	if cch = ':' then begin                 (* a variable *)
		rd;   varoff:= rddigits;   prot:= - varoff;
		if varoff + 1 > nclvars then nclvars:= varoff + 1
	end
	else
	if ( cch = '+' ) or ( cch = '-' ) or ( cc [cch] = cdigit ) then begin
		if cch = '-' then sign:= -1 else sign:= 1;
		if cc [ cch ] <> cdigit then rd;
		(* number itself processed as positive :  this
		   causes loss of smallest integer in two's complement *)
		prot:= newintprot ( sign * rddigits )
	end
	else begin
            prot:= rdnonvarint;
            skipbl
        end;
	if cch <> '.' then rdterm:= prot
	else begin                      (* a sequence, as it turns out *)
		dot:= initprot ( std [atmdot] );   mkarg ( dot, car, prot );
		rd;   skipcombl;
		mkarg ( dot, cdr, rdterm() );   rdterm:= wrapprot ( dot )
	end
end (*rdterm*);

{% endcodeblock %}

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
toy.p(2280,26) Error: Incompatible type for arg no. 2: Got "Constant String", expected "LongInt"
toy.p(2322) Fatal: There were 1 errors compiling module, stopping
Fatal: Compilation aborted

{% endcodeblock %}

Ahh we are getting closer to the end, same *reset* related error, let's fix it by adding *assign*
and run *make* again.

{% codeblock %}

/usr/bin/fpc -O3 -S2 toy.p
Free Pascal Compiler version 2.4.0-2ubuntu1.10.04 [2011/06/17] for i386
Copyright (c) 1993-2009 by Florian Klaempfl
Target OS: Linux for i386
Compiling toy.p
Linking toy
/usr/bin/ld: warning: link.res contains output sections; did you forget -T?
2322 lines compiled, 0.3 sec

{% endcodeblock %}

And we did it! Congratulations, it compiles. Let's try running *toy* executable!

{% codeblock %}

bash$ ./toy
Toy-Prolog listening:
?- X=1.
X = 1
;
no
?- display('Hello World!'), nl.
Segmentation fault

{% endcodeblock %}

Well it runs, but function call crashes the toy prolog interpreter.
Let's investigate this issue using *gdb* debugger. 
We'll need to compile code with *-g* flag to do actual debugging so just add
it to your *FPCFLAGS* variable in *Makefile*

Now let's start debugging

{% codeblock %}

bash$ gdb ./toy
GNU gdb (GDB) 7.1-ubuntu
Copyright (C) 2010 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
(gdb) run
Starting program: /home/troydm/projects/toytest/toy 
Toy-Prolog listening:
?- a.     

Program received signal SIGSEGV, Segmentation fault.
0x0804a831 in PURGETRAIL (LOW=37294) at toy.p:1224
1224                    if (tt [high] < frozenvars) or (tt [high] > frozenheap)

{% endcodeblock %}

It seems *purgetail* procedure is at fault.

{% codeblock lang:pascal %}

procedure purgetrail ( low : ttx );
(* remove unnecessary trail entries at and above low , either after
   a successful unifyordont call or after popping backtrack-points in
   a non-backtracking context ( unfreeze ). this is necessary, as
   unfrozen vars might be moved or destroyed - it also saves trail space. *)
var   high : ttx;
begin   for high:= low to ttop - 1 do
            if (tt [high] < frozenvars) or (tt [high] > frozenheap)
                                                                then begin
                    tt [ low ] :=  tt [ high ];   low:= low + 1
            end;
	ttop:= low
end;

{% endcodeblock %}

Hmm, it's starts from *low* and goes till *ttop-1* to remove unused trail entries from array.
Let's add some *writeln* output of *low* and *ttop* values to see what makes it really crash.

{% codeblock lang:pascal %}

procedure purgetrail ( low : ttx );
(* remove unnecessary trail entries at and above low , either after
   a successful unifyordont call or after popping backtrack-points in
   a non-backtracking context ( unfreeze ). this is necessary, as
   unfrozen vars might be moved or destroyed - it also saves trail space. *)
var   high : ttx;
begin   
        writeln('low = ',low,' ttop = ', ttop);
        for high:= low to ttop - 1 do
                if (tt [high] < frozenvars) or (tt [high] > frozenheap)
                                                                    then begin
                        tt [ low ] :=  tt [ high ];   low:= low + 1
                end;
        ttop:= low
end;

{% endcodeblock %}

Let's run *toy* again

{% codeblock %}

bash$ ./toy
Toy-Prolog listening:
?- a.
low = 1 ttop = 3
low = 0 ttop = 1
low = 0 ttop = 1
low = 0 ttop = 3
low = 0 ttop = 1
low = 0 ttop = 1
low = 0 ttop = 1
low = 0 ttop = 4
low = 0 ttop = 1
low = 0 ttop = 1
low = 0 ttop = 1
low = 0 ttop = 0
Segmentation fault

{% endcodeblock %}

It crashes when both values are 0's. Hmm it seems we don't need to iterate anything unless low < ttop,
Since the trail array is empty, so let's add this fix into *purgetrail*.

{% codeblock lang:pascal %}

procedure purgetrail ( low : ttx );
(* remove unnecessary trail entries at and above low , either after
   a successful unifyordont call or after popping backtrack-points in
   a non-backtracking context ( unfreeze ). this is necessary, as
   unfrozen vars might be moved or destroyed - it also saves trail space. *)
var   high : ttx;
begin   if low < ttop then 
        for high:= low to ttop - 1 do
		if (tt [high] < frozenvars) or (tt [high] > frozenheap)
								    then begin
			tt [ low ] :=  tt [ high ];   low:= low + 1
		end;
	ttop:= low
end;

{% endcodeblock %}

Let's run *make* and then run *toy* interpreter again

{% codeblock %}

bash$ ./toy
Toy-Prolog listening:
?- display('Hello World!'), nl.
Hello World!
yes

{% endcodeblock %}

And we did it! It works :) It might have some other bugs since it's an old software but I haven't encountered any more yet.
I've also ported *btoy.p* the same way stumbling upon same kind of errors and same *purgetrail* bug.

![Hyouka](http://i.imgur.com/NX5NdN5.png)
