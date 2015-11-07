---
layout: post
title: "Writing Micro Compiler in OCaml"
date: 2014-03-29 19:12:25 +0400
comments: true
categories: [compiler, ocaml, asm]
---

TL;DR *Writing micro compiler in OCaml*

At one point or another every single software developer in the world comes to a realization in his career when the time is ripe and it's time 
to write your own super cool programming language.

![Lemon Loli](http://i.imgur.com/KSiXBCN.png)

However the subject of creating your own programming language with an compiler is quite a complex one and can't be tackled without some pre-research.
That's how I've started reading [Crafting Compiler in C](http://www.amazon.com/Crafting-Compiler-Charles-N-Fischer/dp/0805321667/), an aged but 
really comprehensive book about developing your own compiler for an [Ada](http://en.wikipedia.org/wiki/Ada_%28programming_language%29)-like programming language.
Second chapter describes writing a really simple micro language targeting pseudo assembly-like output in order to explain the core concepts of developing your 
own compiler and writing an [LL(1)](https://en.wikipedia.org/wiki/LL_parser) parser.

Let's try rewriting this micro compiler in [OCaml](http://ocaml.org/), a language better suited for writing compilers that is becoming quite popular due to it's clean syntax and strict evaluation semantics combined 
with functional and object-oriented programming styles. If you are not familiar with OCaml try reading [Real World OCaml](https://realworldocaml.org/) first. 
Instead of outputting pseudo assembly our micro compiler will output a real [nasm](http://www.nasm.us/) source code which will be automatically compiled into a binary executable file.
<!--more-->
So let's start by describing simple micro language with an example source code
{% codeblock %}
begin
    a := 1;
    b := a + 1;
    b := b + 1;
    write (a,b);
    read(a,b);
    write (a+10, b+10);
end
{% endcodeblock %}
As you can see from example source code the program starts with **begin** keyword and ends with an **end** keyword. It has only integer variables which must be predefined by assignment operation before using in expressions, and it also has two simple functions **read** and **write**.

**read** takes a list of variable names separated by comma and reads user input from *stdin* into those variables

**write** takes a list of expressions and outputs them into *stdout*

Now in order to create an executable from this source code first we need to parse it. Since LL(1) type parser is enough to parse this kind of language, we'll need only one character lookahead. 
 Unfortunately OCaml doesn't have an unread operation like libc's **ungetc** so we'll need to define a simple stream reader which will have a *mutable char* and we will also count lines of source code read. 
We'll also define two utility functions which we will use later which will check if a character is alphanumeric and if it's a digit.
{% codeblock lang:ocaml %}
(* stream *)
type stream = { mutable chr: char option; mutable line_num: int; chan: in_channel }

let open_stream file = { chr=None; line_num=1; chan=open_in file }
let close_stream stm = close_in stm.chan
let read_char stm = match stm.chr with
                        None -> let c = input_char stm.chan in
                                if c = '\n' then 
                                    let _ = stm.line_num <- stm.line_num + 1 in c
                                else c
                      | Some c -> stm.chr <- None; c
let unread_char stm c = stm.chr <- Some c

(* character *)
let is_digit c = let code = Char.code c in 
                 code >= Char.code('0') && code <= Char.code('9')

let is_alpha c = let code = Char.code c in 
                 (code >= Char.code('A') && code <= Char.code('Z')) ||
                 (code >= Char.code('a') && code <= Char.code('z')) 
{% endcodeblock %}

Now for our parser we will be parsing source code one token at a time and we'll be recursively calling parsing methods and matching tokens as we go.
We'll define some additional utility functions which will be used during parsing including an *Syntax_error* exception which will be thrown if invalid
token is scanned.

**scan** will scan the stream for next token, that's where our token recognition logic is in.

**skip_blank_chars** function will skip any number of blank characters including new line characters

**next_token** will return last scanned token or will scan stream for next token

**match_next** will match last scanned token or will scan stream for next token, match it and return it

**match_token** will match the last scanned token against the specified token in parameter or will scan stream for next token and match against it

{% codeblock lang:ocaml %}
(* token *)
type token = Begin | End
           | Identifier of string 
           | Read | Write 
           | Literal of int 
           | Assign 
           | LeftParen | RightParen  
           | AddOp | SubOp
           | Comma | Semicolon 

type scanner = { mutable last_token: token option; stm: stream }
  
exception Syntax_error of string

let syntax_error s msg = raise (Syntax_error (msg ^ " on line " ^ (string_of_int s.stm.line_num)))


(* skip all blank and new line characters *)
let rec skip_blank_chars stm = let c = read_char stm in
                               if c = ' ' || c = '\t' || c = '\r' || c = '\n'
                               then skip_blank_chars stm
                               else unread_char stm c; ()

(* scan a stream and return next token *)
let scan s =   let stm = s.stm in let c = read_char stm in
               let rec scan_iden acc = let nc = read_char stm in
                                       if is_alpha nc || is_digit nc || nc='_' 
                                       then scan_iden (acc ^ (Char.escaped nc))
                                       else let _ = unread_char stm nc in 
                                            let lc = String.lowercase acc in
                                            if lc = "begin" then Begin
                                            else if lc = "end" then End
                                            else if lc = "read" then Read
                                            else if lc = "write" then Write
                                            else Identifier acc 
               in
               let rec scan_lit acc = let nc = read_char stm in
                                      if is_digit nc 
                                      then scan_lit (acc ^ (Char.escaped nc))
                                      else let _ = unread_char stm nc in 
                                           Literal (int_of_string acc)
               in
               if is_alpha c then scan_iden (Char.escaped c)
               else if is_digit c then scan_lit (Char.escaped c)
               else if c='+' then AddOp
               else if c='-' then SubOp
               else if c=',' then Comma
               else if c=';' then Semicolon 
               else if c='(' then LeftParen
               else if c=')' then RightParen
               else if c=':' && read_char stm = '=' then Assign
               else syntax_error s "couldn't identify the token"

let new_scanner stm = { last_token=None; stm=stm }

let match_next s = match s.last_token with
                      None -> let _ = skip_blank_chars s.stm in scan s
                    | Some tn -> s.last_token <- None; tn

let match_token s t = match_next s = t

let next_token s = match s.last_token with
                        None ->  (skip_blank_chars s.stm; 
                                  let t = scan s in
                                  s.last_token <- Some t; t)
                      | Some t -> t
{% endcodeblock %}

In order to generate an asm output we'll define an generator type which will contain the output channel 
and variable locations *Hashtbl*. Each variable's location will be defined as an integer offset from *esp* 
and since our micro language is simple we won't have to handle advanced aspects of variable scope and stack
handling. 

{% codeblock lang:ocaml %}
(* code generation *)
type generator = { vars: (string, int) Hashtbl.t; file: string; chan: out_channel }

let new_generator file = let fs = (Filename.chop_extension file) ^ ".s" in
                         { vars=Hashtbl.create 100; file=fs; chan=open_out fs }

let close_generator g = close_out g.chan

let gen g v = output_string g.chan v; output_string g.chan "\n"
{% endcodeblock %}

We'll also need to distinguish between ordinary variables and an temporary ones.
Our temporary variables will be defined automatically and will start from *__temp* following
variable location offset. This way we won't be storing temporary variables in *Hashtbl* and will be 
computing their offset from their name. We'll also define some additional generator helper functions to output asm code.

{% codeblock lang:ocaml %}
let bottom_var _ g = Hashtbl.fold (fun _ v c -> if v >= c then (v+4) else c) g.vars 0
let empty_var s g i = (bottom_var s g)+(4*(i-1))

let var_addr s g v = if String.length v > 6 && String.sub v 0 6 = "__temp" 
                then let i = String.sub v 6 ((String.length v) - 6) in "[esp+" ^ i ^ "]"
                else
                try "[esp+" ^ string_of_int (Hashtbl.find g.vars v) ^ "]"
                with Not_found -> syntax_error s ("identifier " ^ v ^ " not defined")
let var s g v = "dword " ^ (var_addr s g v)

let temp_var s g i = Identifier ("__temp" ^ (string_of_int (empty_var s g i)))

let is_alloc_var _ g v = Hashtbl.mem g.vars v

let alloc_var s g v = if is_alloc_var s g v then var s g v
                      else let _ = Hashtbl.replace g.vars v (empty_var s g 1) in var s g v

let token_var s g v = match v with
                         Identifier i -> var s g i
                       | _ -> syntax_error s "identifier expected"

let op g opcode a = gen g ("    " ^ opcode ^ "  " ^ a)
let op2 g opcode a b = gen g ("    " ^ opcode ^ "  " ^ a ^ ", " ^ b)
let push g a = op g "push" a
{% endcodeblock %}

Now in order to compile a source code we need to create a new generator, open an stream on a source file, parse it,
compile the asm source code using *nasm* and link the generated *.o* file into an elf executable.
{% codeblock lang:ocaml %}
(* compiling *)
let compile file =
    try
        let g = new_generator file in
        let stm = open_stream file in
        let out = Filename.chop_extension file in
        parse stm g; 
        close_stream stm; 
        close_generator g;
        let _ = Sys.command ("nasm -f elf " ^ g.file) in
        let _ = Sys.command ("gcc -o " ^ out ^ " " ^ out ^ ".o") in ()
    with Syntax_error e ->
            Format.printf "syntax error: %s\n" e;
            Format.print_flush()
       | Sys_error _ ->
            print_string "no such file found\n"

let help () = print_string "micro <file>\n"

let () = if Array.length Sys.argv = 1 then help ()
         else 
             let file = Array.get Sys.argv 1 
             in
             Format.printf "compiling %s\n" file;
             Format.print_flush ();
             compile file
{% endcodeblock %}

Our parsing method will be combined with semantics checking and will output *asm* code using generator functions which we will define later. 
Program begins from *begin* keyword, ends with *end* keyword and has 0 or more statements in between.
{% codeblock lang:ocaml %}
let parse stm g = 
        let s = (new_scanner stm) in
        try
            program s g
        with End_of_file -> syntax_error s "program reached end of file before end keyword"

let program s g = if match_token s Begin then 
                    let _ = generate_begin s g in
                    let _ = statements s g in
                    if match_token s End then 
                    let _ = generate_end s g in ()
                    else syntax_error s "program should end with end keyword"
                else syntax_error s "program should start with begin keyword"

let rec statements s g = if statement s g then statements s g else ()
{% endcodeblock %}
Each statement is either an *read*, *write* or an assignment to a variable.
{% codeblock lang:ocaml %}
let rec statements s g = if statement s g then statements s g else ()

let statement s g = let t = next_token s in
                  if match t with
                      Read -> read s g
                    | Write -> write s g
                    | Identifier i -> assignment s g
                    | _ -> false
                  then
                      if match_token s Semicolon then true
                      else syntax_error s "statement must end with semicolon"
                  else false
{% endcodeblock %}
Each assignment statement has an identifier token on it's left side followed by an assignment token and expression
on the right hand side. 
{% codeblock lang:ocaml %}
let assignment s g = let id = match_next s in
                     match id with
                        Identifier i -> (if match_token s Assign then
                                               let new_var = if is_alloc_var s g i then 0 else 1 in
                                               let id2 = expression s g (1+new_var) in
                                               match id2 with
                                                   Literal l2 -> let _ = generate_assign s g id id2 in true
                                                 | Identifier i2 -> let _ = generate_assign s g id id2 in true
                                                 | _ -> syntax_error s "literal or identifier expected"
                                         else syntax_error s "assign symbol expected")
                      | _ -> syntax_error s "identifier expected"
{% endcodeblock %}
Each expression statement is primary optionally followed by an operation token and another primary.
Primary might also be an expression in curly brackets.
{% codeblock lang:ocaml %}
let rec expression s g d = 
        let primary s = match (next_token s) with
                            LeftParen -> (let _ = match_token s LeftParen in
                                          let e = expression s g (d+1) in
                                          if match_token s RightParen then Some e
                                          else syntax_error s "right paren expected in expression")
                          | Identifier i -> let _ = match_token s (Identifier i) in Some (Identifier i)
                          | Literal l -> let _ = match_token s (Literal l) in Some (Literal l)
                          | _ -> None
        in 
        let lp = primary s in
        match lp with
            Some l -> (match (next_token s) with
                             AddOp -> let _ = match_token s AddOp in
                                      addop s g d l (expression s g (d+1))
                           | SubOp -> let _ = match_token s SubOp in
                                      subop s g d l (expression s g (d+1))
                           | _ -> l)
          | None -> syntax_error s "literal or identifier expected"
{% endcodeblock %}

Our micro language supports only two operations on integers, addition and subtraction, but
it can be easily extended to support more.

{% codeblock lang:ocaml %}
let addop s g d l r = match (l, r) with
                            (Literal l1, Literal l2) -> Literal (l1+l2)
                          | (Identifier i1, Literal l2) -> generate_add s g d l r
                          | (Literal l1, Identifier i2) -> generate_add s g d r l
                          | _ -> syntax_error s "expected literal or identifier for add operation"
let subop s g d l r = match (l, r) with
                            (Literal l1, Literal l2) -> Literal (l1-l2)
                          | (Identifier i1, Literal l2) -> generate_sub s g d l r
                          | (Literal l1, Identifier i2) -> generate_sub s g d l r
                          | _ -> syntax_error s "expected literal or identifier for sub operation"
{% endcodeblock %}

*write* statement is just comma separated list of expressions.

{% codeblock lang:ocaml %}
let write s g = let rec expressions c =
                    let e = (expression s g 1) in
                    if match e with
                        Identifier _ -> let _ = generate_write s g e in true
                      | Literal _ -> let _ = generate_write s g e in true
                      | _ -> false
                    then if (next_token s) = Comma then 
                            let _ = match_token s Comma in expressions (c+1) 
                         else (c+1) 
                    else c
                in 
                if match_token s Write then 
                    if match_token s LeftParen then
                        if expressions 0 > 0 then
                            if match_token s RightParen then true
                            else syntax_error s "right paren expected in write statement"
                        else syntax_error s "write statement expected atleast one expression"
                    else syntax_error s "left paren expected in write statement"
                else syntax_error s "write statement expected"
{% endcodeblock %}

*read* statement is a comma separated list of identifiers

{% codeblock lang:ocaml %}
let read s g = if match_token s Read then 
                if match_token s LeftParen then
                    let ids = identifiers s in 
                    if ids = [] then syntax_error s "read statement expects comma seperated identifier(s)"
                    else if match_token s RightParen then let _ = generate_reads s g (List.rev ids) in true
                         else syntax_error s "right paren expected in read statement"
                else syntax_error s "left paren expected in read statement"
             else syntax_error s "read statement expected"

let identifiers s = let rec idens ids =  
                        match (next_token s) with
                            Identifier i -> let _ = match_next s in
                                            let n = next_token s in
                                            if n = Comma then let _ = match_token s Comma in idens (Identifier i :: ids)
                                            else idens (Identifier i :: ids)
                          | _ -> ids
                    in idens []
{% endcodeblock %}

Now it's finally time to generate some *asm* code, let's start from *begin* and *end* of our program.
Our program will preallocate 1000 bytes on stack for variables, since all of our variables are static.
We'll also need to define external libc functions **scanf** and **printf**.
{% codeblock lang:ocaml %}
let generate_begin _ g = gen g 
"extern printf
extern scanf

section .data
    inf: db '%d', 0
    ouf: db '%d', 10, 0

section .text
    global main

main:
    sub   esp, 1000"

let generate_end _ g = gen g 
"    add   esp, 1000
exit:
    mov  eax, 1 ; sys_exit
    mov  ebx, 0
    int  80h"
{% endcodeblock %}
*read* and *write* statements will use libc **scanf** and **printf** functions to read integer variables
from *stdin* and output them on *stdout*.
{% codeblock lang:ocaml %}
let generate_read s g id = match id with
                            Identifier i -> (op2 g "lea" "eax" (var_addr s g i); 
                                             push g "eax"; 
                                             push g "inf"; 
                                             op g "call" "scanf";
                                             op2 g "add " "esp" "8")
                          | _ -> syntax_error s "generate read called with invalid argument"

let rec generate_reads s g = List.iter (generate_read s g)

let generate_write s g id = match id with
                            Identifier i -> (push g (var s g i); 
                                             push g "ouf"; 
                                             op g "call" "printf";
                                             op2 g "add " "esp" "8")
                          | _ -> syntax_error s "generate write called with invalid argument"
{% endcodeblock %}

Assignment statement is just an allocation of a variable followed by a copying variable from one location
into another. Our copy function understands literal variables and translates their values directly into *asm* code.

{% codeblock lang:ocaml %}
let generate_assign s g a b = match a with
                                Identifier i -> let _ = alloc_var s g i in generate_copy s g a b
                              | _ -> syntax_error s "generate assign called with invalid argument"

let generate_copy s g a b = match a with
                                Identifier i -> (match b with
                                                        Identifier i2 -> (op2 g "mov " "eax" (var s g i2); 
                                                                          op2 g "mov " (var s g i) "eax")
                                                      | Literal l -> op2 g "mov " (var s g i) (string_of_int l)
                                                      | _ -> syntax_error s "generate copy called with invalid argument")
                              | _ -> syntax_error s "generate copy called with invalid argument"
{% endcodeblock %}

Addition and subtraction operations use temporary variables to add and subtract
values from two variables and return result

{% codeblock lang:ocaml %}
let generate_add s g d id1 id2 = match (id1, id2) with 
                                     (Identifier i1, Identifier i2) -> (let v = temp_var s g d in
                                                                        let vi = token_var s g v in
                                                                        let _ = generate_copy s g v id1 in 
                                                                        let _ = op2 g "add " vi (var s g i2) in v)
                                   | (Identifier i1, Literal l2) -> (let v = temp_var s g d in
                                                                     let vi = token_var s g v in
                                                                     let _ = generate_copy s g v id1 in 
                                                                     let _ = op2 g "add " vi (string_of_int l2) in v)
                                   | _ -> syntax_error s "generate exp called with invalid argument" 

let generate_sub s g d id1 id2 = match (id1, id2) with 
                                     (Identifier i1, Identifier i2) -> (let v = temp_var s g d in
                                                                        let vi = token_var s g v in
                                                                        let _ = generate_copy s g v id1 in 
                                                                        let _ = op2 g "sub " vi (var s g i2) in v)
                                   | (Identifier i1, Literal l2) -> (let v = temp_var s g d in
                                                                     let vi = token_var s g v in
                                                                     let _ = generate_copy s g v id1 in 
                                                                     let _ = op2 g "sub " vi (string_of_int l2) in v)
                                   | (Literal l1, Identifier i2) -> (let v = temp_var s g d in
                                                                     let vi = token_var s g v in
                                                                     let _ = generate_copy s g v id1 in
                                                                     let _ = op2 g "sub " vi (var s g i2) in v)
                                   | _ -> syntax_error s "generate exp called with invalid argument" 
{% endcodeblock %}

And that's all of it! We now have a trivial micro compiler that generates binary executable file.
Source code can be found in my github [micro](https://github.com/troydm/micro/) repo. Writing an compiler
is quite complex and entertaining task but it's definitely worth the time spend on! 

![Kawaii Loli](http://i.imgur.com/ArEvld4.png)
