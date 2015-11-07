---
layout: post
title: "Rewriting Micro Compiler in OCaml using ocamllex and ocamlyacc"
date: 2014-11-14 22:58:29 +0400
comments: true
categories: [compiler, ocaml, asm]
---

TL;DR *Rewriting micro compiler in OCaml*

In [my previous post](http://troydm.github.io/blog/2014/03/29/writing-micro-compiler-in-ocaml/) I've talked about writing micro compiler in OCaml under 300 lines of source code. There are number of ways to make our work easier and number of source code lines significantly smaller.

![Potato Loli](http://i.imgur.com/2q3i5KA.gif)

Let's rewrite our micro compiler using tools called *lexer* and *parser* generators. We'll be using tools called **ocamllex** and **ocamlyacc** which are distributed with *OCaml* compiler and are modeled after
famous **lex** and **yacc** tools for *Unix* operating systems. Those tools actually have better modern analogues called **flex** and **bison** which are described in detail in [Flex & Bison: Text Processing Tools](http://www.amazon.com/flex-bison-Text-Processing-Tools/dp/0596155972/) book.
Nowadays however if you are writing a professional compiler in *OCaml* I strongly suggest you consider using [sedlex](https://github.com/alainfrisch/sedlex) and [menhir](http://gallium.inria.fr/~fpottier/menhir/) instead of **ocamllex** and **ocamlyacc** as both tools are quite outdated and 
lack some significant features that their modern analogues have such as unicode support for lexing, parameterized parser generation and built-in grammar interpreter. So what are lexer and parser generators?
To put it simply **ocamllex** and **ocamlyacc** take special *.mll* and *.mly* definition files of lexer and parser semantics mixed with *OCaml* source code and generate an *.ml* source code files that do the actual token generation and parsing for you. 
Pretty neat indeed, and it's actually easier to use than it sounds so let's rewrite our micro compiler using those tools. We'll be using original source code of micro compiler as reference only as entire code base needs to be changed. You can see the end result of our rewrite in [micro](https://github.com/troydm/micro/tree/simple) git repository branch called *simple*. For the actual description of the micro language see my previous post. So let's get started!
<!--more-->

First let's create our files for lexer and parser called *lexer.mll* and *parser.mly*. Now let's define our tokens in *parser.mly*. As you can see it's defined
using special syntax that is described in [ocamlyacc manual](http://caml.inria.fr/pub/docs/manual-ocaml/lexyacc.html).

{% codeblock lang:ocaml %}
%token BEGIN END EOF
%token <string> IDENTIFIER
%token <int> LITERAL
%token READ WRITE
%token ASSIGN 
%token LEFTPAREN RIGHTPAREN  
%token ADDOP SUBOP
%token COMMA SEMICOLON 
{% endcodeblock %}

Now let's define lexical semantics for those tokens in *lexer.mll* file. The tokens we defined in parser.mly will be generated into *parser.mli* interface
file so first of all let's include those from parser module by adding a header to *lexer.mll* file. The part between *{* and *}* is defined in *OCaml* syntax
and will be translated into the generated lexer.ml file without any changes. For the description of *ocamllex* definition file consult it's [manual](http://caml.inria.fr/pub/docs/manual-ocaml/lexyacc.html)

{% codeblock lang:ocaml %}
{

open Parser

}
{% endcodeblock %}

Next let's define some lexical semantics next for blank characters, digits and alpha numerical characters. As you can see those are defined using 
regular expression syntax and can be referencing each other. This way we can define identifier lexical semantics by just referencing alpha and digit definitions. 

{% codeblock lang:ocaml %}
let blank = [' ' '\r' '\t']
let digit = ['0'-'9']
let digits = digit*
let alpha = ['a'-'z' 'A'-'Z']
let iden = alpha (alpha | digit | '_')*
{% endcodeblock %}

Now let's define a rule that will give us the actual tokens. As you can see the part between *{* and *}* is in *OCaml* syntax and just gives us back the tokens we've defined.

{% codeblock lang:ocaml %}
rule micro = parse
    | ":="     { ASSIGN }
    | '+'      { ADDOP }
    | '-'      { SUBOP }
    | ','      { COMMA }
    | ';'      { SEMICOLON }
    | '('      { LEFTPAREN }
    | ')'      { RIGHTPAREN }
    | digits as d { 
        (* parse literal *)
        LITERAL (int_of_string d)
    }
{% endcodeblock %}

Now let's add the line counting and syntax error reporting to our *lexer.mll* header.

{% codeblock lang:ocaml %}
(* current token line number *)
let line_num = ref 1

exception Syntax_error of string

let syntax_error msg = raise (Syntax_error (msg ^ " on line " ^ (string_of_int !line_num)))
{% endcodeblock %}

Next we'll add the rule for counting new lines and reporting syntax errors if lexer encounters unknown token. Also we need to generate *EOF* token when lexer will encounter the end of file. To skip blank characters, as those aren't need in our micro compiler, we'll just recursively call lexer's micro rule providing it with *lexbuf* which is described in manual.

{% codeblock lang:ocaml %}
rule micro = parse
    .....
    | '\n'     { incr line_num; micro lexbuf } (* counting new line characters *)
    | blank    { micro lexbuf } (* skipping blank characters *)
    | _        { syntax_error "couldn't identify the token" }
    | eof      { EOF } (* no more tokens *)
{% endcodeblock %}

Woah woaw wow, much forget, wait!!! Aren't we missing something?! Ah yes, identifiers, indeed! First of all let's define a table of known keywords in the header as we need to handle them differently.

{% codeblock lang:ocaml %}
(* keyword -> token translation table *)
let keywords = [
    "begin", BEGIN; "end", END; "read", READ; "write", WRITE
]
{% endcodeblock %}

And now let's handle the actual identifier tokens.

{% codeblock lang:ocaml %}
rule micro = parse
    .....
    | iden as i {
        (* try keywords if not found then it's identifier *)
        let l = String.lowercase i in
        try List.assoc l keywords
        with Not_found -> IDENTIFIER i   
    }
{% endcodeblock %}

Congratulations we have a lexer ready, now it's a parsing time! Let's define a parser for our micro code. Parser definition syntax is pretty straightforward and reminds kinds of [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_Form) definition. 
If you don't understand it just by looking at it or don't know what *BNF* is I suggest you take a look into description of definition syntax by consulting ocamlyacc manual.
As you can see our parser starts with a program definition which starts with *begin statement* followed by *statements* and followed by *end statement* definitions and ends with *EOF* token.
The part between *{* and *}* is *OCaml* source code that is executed after the parsing of the definition is done. When parser does it's job it raises *End_of_file* exception which we'll 
be handling as end of parsing. So basically when parser sees *BEGIN* token it just executes *generate_begin* function which we'll define shortly. Same with *END* token only now we are executing
*generate_end* function instead. Statements definition is recursive and just references a *statement* followed by *semicolon* followed by more *statements* or none.

{% codeblock lang:ocaml %}
%start program
%type <unit> program

%%

program:
|   begin_stmt statements end_stmt EOF { raise End_of_file }
;

begin_stmt:
|   BEGIN { generate_begin () }
;

end_stmt:
|   END { generate_end () }
;

statements:
| { }
| statement SEMICOLON statements { }
;
{% endcodeblock %}

Now let's write some code to generate the assembly for the *begin* and *end* statements. We'll put all our code generation functions into a separate module called *codegen* so let's create
a new file called *codegen.ml* and add some code generation methods into it.

{% codeblock lang:ocaml %}
(* code generation *)
let chan = ref stdout

let set_chan new_chan = chan := new_chan

let gen v = output_string !chan v; output_string !chan "\n"

let generate_begin () = gen
"extern printf
extern scanf

section .data
    inf: db '%d', 0
    ouf: db '%d', 10, 0

section .text
    global main

main:
    sub   esp, 4096"

let generate_end () = gen
"    add   esp, 4096
exit:
    mov  eax, 1 ; sys_exit
    mov  ebx, 0
    int  80h"
{% endcodeblock %}

Now for parsing expression we need some way to count the depth of the expression for temporary variables so we'll be just incrementing it as we go deeper and then reset it to 0 after 
the statement is over. This approach is not optimized one as it would mean some extra stack usage for same depth AST nodes however for sake of simplicity we'll leave it as it is. 
So now our *parser.mly* header should look like this. 

{% codeblock lang:ocaml %}
%{

open Codegen

let depth = ref 0
let depth_incr f = incr depth; f !depth
let depth_reset = depth := 0

%}
{% endcodeblock %}

So let's define description of *statement* in parser

{% codeblock lang:ocaml %}
statement:
| IDENTIFIER ASSIGN expression { generate_assign $1 $3; depth_reset }
| READ LEFTPAREN identifier_list RIGHTPAREN { generate_reads $3 }
| WRITE LEFTPAREN expression_list RIGHTPAREN { generate_writes $3; depth_reset }
;
{% endcodeblock %}

Let's start with simpler *read statement*. Now as you can see for *read statement* we are using identifiers only so we don't need to do anything with depth of expression so we aren't 
calling depth_reset function. Now in order to get identifier list we'll be just parsing identifiers and collecting them into the list.

{% codeblock lang:ocaml %}
identifier_list:
| IDENTIFIER { [$1] } 
| IDENTIFIER COMMA identifier_list { $1 :: $3 }
;
{% endcodeblock %}

Now let's generate some assembly for reading data into identifiers. First let's start with some operation helper functions. The read statement is just pretty straightforward and 
just generates a scanf function call for every identifier.

{% codeblock lang:ocaml %}
let op opcode a = gen ("    " ^ opcode ^ "  " ^ a)
let op2 opcode a b = gen ("    " ^ opcode ^ "  " ^ a ^ ", " ^ b)
let push a = op "push" a

let generate_read i = op2 "lea" "eax" (var_addr i); 
                      push "eax"; 
                      push "inf"; 
                      op "call" "scanf";
                      op2 "add " "esp" "8"

let generate_reads = List.iter generate_read
{% endcodeblock %}

As you can see we are using *var_addr* function which we didn't defined yet, this function will give us the actual offset address on the stack based on the name of identifier.
We also handle **__temp** identifiers separately as those are just temporary variables which are discarded every time the statement is over, their offset is specified in their name after 
the **__temp** part, so for example **__temp1** is an temporary variable with offset 1. Let's write some code that handles all of this.  All the named variables that are defined are saved into 
*vars* *Hashtbl* that contains *variable name -> stack offset* information.

{% codeblock lang:ocaml %}
exception Codegen_error of string

let codegen_error msg = raise (Codegen_error msg)

let vars = ref (Hashtbl.create 100)

let var_addr v = if String.length v > 6 && String.sub v 0 6 = "__temp" 
                 then let i = String.sub v 6 ((String.length v) - 6) in "[esp+" ^ i ^ "]"
                 else
                 try "[esp+" ^ string_of_int (Hashtbl.find !vars v) ^ "]"
                 with Not_found -> codegen_error ("identifier " ^ v ^ " not defined")
{% endcodeblock %}

Now for the write statement instead of identifiers we could have expressions, so let's define those.

{% codeblock lang:ocaml %}
expression_list:
| expression { [$1] }
| expression COMMA expression_list { $1 :: $3 }
;
{% endcodeblock %}

And the code that generates write statements is almost the same as read but instead of calling scanf we'll be calling printf function.

{% codeblock lang:ocaml %}
let generate_write i = push i; 
                       push "ouf"; 
                       op "call" "printf";
                       op2 "add " "esp" "8"

let generate_writes = List.iter generate_write
{% endcodeblock %}

Let's now declare an expression

{% codeblock lang:ocaml %}
expression:
| IDENTIFIER { var $1 }
| LITERAL { generate_literal $1 } 
| addop { $1 }
| subop { $1 }
;
{% endcodeblock %}

For simple cases expression can be just a *variable* or a *literal* so let's write some code that generates assembly for those.

{% codeblock lang:ocaml %}
let var v = "dword " ^ (var_addr v)

let generate_literal = string_of_int
{% endcodeblock %}

Let's now handle *addition* and *substraction* expressions.

{% codeblock lang:ocaml %}
addop:
| LITERAL ADDOP LITERAL { generate_literal ($1 + $3) }
| expression ADDOP expression { (depth_incr generate_add) $1 $3 }
;

subop:
| LITERAL SUBOP LITERAL { generate_literal ($1 - $3) }
| expression SUBOP expression { (depth_incr generate_sub) $1 $3 }
;
{% endcodeblock %}

As you can see simple cases such as when we have literal from both sides are handled by our parser itself.
Now let's write some more code generation functions for expressions. We'll be using temporary variables so
we need to identify a bottom variable at the stack that will be on the top offset. And from there we'll use depth
of current expression in order to have a unique position for the temporary variable. This part can be written 
slightly in a different way for example instead of counting depth we'll be counting the temporary variables and resetting them
each time a statement is over. Both approaches are almost the same so we'll just leave it like that. 

{% codeblock lang:ocaml %}
let bottom_var () = Hashtbl.fold (fun _ v c -> if v >= c then (v+4) else c) !vars 0
let empty_var i = (bottom_var ())+(4*(i-1))
let temp_var i = "__temp" ^ (string_of_int (empty_var i))

let is_var v = 
    let re = Str.regexp_string "[esp+" in
    try ignore (Str.search_forward re v 0); true
    with Not_found -> false

let generate_copy a b = if a = b then () else
                        if is_var b then begin op2 "mov " "eax" b; op2 "mov " a "eax" end
                        else op2 "mov " a b
    
let generate_add d id1 id2 = let v = var (temp_var d) in
                             generate_copy v id1; 
                             if is_var id2 then begin op2 "mov " "eax" id2; op2 "add " v "eax" end
                             else op2 "add " v id2; v

let generate_sub d id1 id2 = let v = var (temp_var d) in
                             generate_copy v id1; 
                             if is_var id2 then begin op2 "mov " "eax" id2; op2 "sub " v "eax" end
                             else op2 "sub " v id2; v
{% endcodeblock %}

Now the only thing left to do is generate some code for *assignment* expression and our parser is done.

{% codeblock lang:ocaml %}
let is_alloc_var v = Hashtbl.mem !vars v

let alloc_var v = if is_alloc_var v then var v
                  else let _ = Hashtbl.replace !vars v (empty_var 1) in var v

let generate_assign a b = generate_copy (alloc_var a) b
{% endcodeblock %}

And now for the final part we need to write the actual compiling function. It will parse the micro source file specified as an argument and 
will compile it using *nasm* and *gcc*. Let's create *micro.ml* file and write some final code.

{% codeblock lang:ocaml %}
(* compiling *)
let compile f = 
        let out = (Filename.chop_extension f) in
        let out_chan = open_out (out ^ ".s")
        and lexbuf = Lexing.from_channel (open_in f) in
        try
            let rec parse () = 
                Parser.program Lexer.micro lexbuf; parse () in
            Codegen.set_chan out_chan;
            ignore(parse ());
        with 
          End_of_file -> 
            begin
                close_out out_chan;
                ignore(Sys.command ("nasm -f elf32 " ^ out ^ ".s"));
                ignore(Sys.command ("gcc -m32 -o " ^ out ^ " " ^ out ^ ".o"))
            end
        | Lexer.Syntax_error s ->
            print_string s;
            exit 1

let help () = print_string "micro <file>\n"

let () = if Array.length Sys.argv = 1 then help ()
         else 
             let file = Array.get Sys.argv 1 in
             Format.printf "compiling %s\n" file;
             Format.print_flush ();
             compile file
{% endcodeblock %}


Phew, writing posts is exhausting but finally our micro compiler is ready! Let's test it. You need to have *nasm* and *gcc* installed on your *Linux* system in order to run it.

{% codeblock lang:bash %}
./micro ./examples/hello.mc
./examples/hello
{% endcodeblock %}

And it works! Congratulations! We are done! So what we learned today? Building compiler is actually easier than it seems! Happy compiling everyone!

![Sad Loli](http://i.imgur.com/cmm6AWx.jpg)
