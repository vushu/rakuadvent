### Autogenerating Raku bindings!

#### Preface
For this advent post I will tell you about how I got into raku and my struggles of making raylib-raku bindings.
I already have some knowledge about C, which helped me tremendously when making the bindings.
I didn't explain much about C [passing by value or passing by reference](https://www.geeksforgeeks.org/difference-between-call-by-value-and-call-by-reference/).
So I suggest learning a bit of C if you are more interested.

#### Encountering Raku
I discovered Raku by coincedence in a [youtube video](https://www.youtube.com/watch?v=UVUjnzpQKUo&t=289s) and I got intrigued by how expressive it is.
While reading through the docs, the feature that caught my eye was the first class support for grammars!
Trying out the grammar was very intuitive if you have worked with some parser generator where [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) is used, then it should be quite similar.
I will definitely make a toy compiler/interpreter using Raku at some point.
Before doing that I wanted to make a chip-8 emulator in Raku and want [Raylib](https://github.com/raysan5/raylib) for rendering since I have used it before.
Sadly there was no bindings for it in raku, but then maybe I could make the bindings?

#### Making native bindings
So I got a bit sidetracked from making the emulator and began reading through the docs for creating the bindings using Raku [NativeCall](https://docs.raku.org/language/nativecall).
I began translating some simple functions in raylib.h to to raku nativecall, my first attempt was something like this:

```php
use NativeCall;

constant LIBRAYLIB = 'libraylib.so';

class Color is export is repr('CStruct') is rw {
    has uint8 $.r;
    has uint8 $.g;
    has uint8 $.b;
    has uint8 $.a;
}

sub InitWindow(int32 $width, int32 $Height, Str $name) is native(LIBRAYLIB) { * }
sub WindowShouldClose(--> bool) is native(LIBRAYLIB) { * }
sub BeginDrawing() is native(LIBRAYLIB) { * }
sub EndDrawing() is native(LIBRAYLIB) { * }
sub CloseWindow() is native(LIBRAYLIB) { * }
sub ClearBackground(Color $color) is native(LIBRAYLIB) { * }

my $white = Color.new(r => 255,  g => 255,  b => 255,  a => 255);
InitWindow(800, 450, "Raylib window in Raku!");
while (!WindowShouldClose()) {
    BeginDrawing();
    ClearBackground($white);
    EndDrawing();
}
CloseWindow();

```
Yay we got a window!

![First window](first-window.png)

But something is clearly off, the background wasn't white as I defined it to be.
I turns out that `ClearBackground` expects `Color` as value type as shown below:
```
RLAPI void ClearBackground(Color color);       
```

The problem is Raku NativeCall only supports passing as reference!

After asking the community for guidens, I got the solution to pointerize the function, 
meaning we need to make a new wrapper function example: `ClearBackground_pointerized(Color* color)` which takes Color as a pointer and then call the original function with the dereferenced value:

```c
void ClearBackground_pointerized(Color* color)
{
     ClearBackground(*color);
}
```
Since Color must be pointer we need to allocate it on the heap using C's [malloc](https://en.wikipedia.org/wiki/C_dynamic_memory_allocation) function.
We need to expose a `malloc_Color` function in C-intermediate code to be able to call this from Raku.

```c
Color* malloc_Color(unsigned char  r, unsigned char g, unsigned char b, unsigned char a) {
   Color* ptr = malloc(sizeof(Color));
   ptr->r = r;
   ptr->g = g;
   ptr->b = b;
   ptr->a = a;
   return ptr;
}
```
If you malloc you also need to free it, or else we will memory leak.
We need to expose another function for calling free on Color.

```c
void free_Color(Color* ptr){
   free(ptr);
}
```

To make this more Object-Oriented we supply the Color class with init method for mallocing it on the heap. 
We handle freeing Color by using the submethod DESTROY, whenever the GC decides to collect the resource it will also free Color from the heap.

```php
class Color is export is repr('CStruct') is rw {
    has uint8 $.r;
    has uint8 $.g;
    has uint8 $.b;
    has uint8 $.a;
    method init(uint8 $r,uint8 $g,uint8 $b,uint8 $a) returns Color {
        malloc-Color($r,$g,$b,$a);
    }
    submethod DESTROY {
        free-Color(self);
    }
}
```

The intermediate C-code ofcourse needs to also be compiled into the raylib library.

```
gcc -g -fPIC intermediate-code.c -lraylib -o modified/libraylib.so
```

Let's try again:

```php
...
# using the modified libraylib.so
constant LIBRAYLIB = 'modified/libraylib.so';
...
# Not using init to malloc
my $white = Color.init(r => 255,  g => 255,  b => 255,  a => 255);
InitWindow(800, 450, "Raylib window in Raku!");
while (!WindowShouldClose()) {
    BeginDrawing();
    ClearBackground($white);
    EndDrawing();
}
CloseWindow();
```

Yes now it works as expected!

![White window](white-window.png)

Phew!  
All this work has to be done for every functions that is using value types. 
Looking at [raylib.h](https://github.com/raysan5/raylib/blob/master/src/raylib.h).
That's many functions!

Maybe that's a reason for why nobody has made bindings for raylib, because it's way too tedious!!!

At that point I wished that NativeCall would handle this us and almost didn't bother working on making the bindings.

Then I had a great idea! Raku has Grammar support! What if I just parse the header file and automatically generate the bindings using the actions.

### Generating bindings

So I began defining the grammar for raylib and not for C, since that's a bigger task.

#### Grammar

```php
grammar RaylibGrammar {
    token TOP {
        [ <defined-content> ]*
    }

    rule defined-content {
        | <macro-block>
        | <closing-bracket>
        | <typedef>
        | <function>
        | <include>
        | <define-decl>
        | <statement>
    }

    rule macro-block {
        | <extern>
        | <if-macro-block>
        | <ifndef-block>
        | <else-macro-line>
        | <elif-macro-line>
        | <error-macro-line>
    }

    rule typedef-ignored {
        'typedef' 'enum' 'bool' <block> 'bool' ';'
    }

    rule extern {
        'extern' <string> '{'
    }

    token closing-bracket {
        '}'
    }

    rule if-macro-block {
        '#if' \N* \n? <defined-content>* '#endif'
    }

    rule ifndef-block {
        '#ifndef' <identifier> <defined-content>* '#endif'
    }
    rule error-macro-line {
        '#error' \N* \n?
    }
    rule else-macro-line {
        '#else' \n?
    }

    rule elif-macro-line {
        '#elif' \N* \n?
    }

    rule include {
        '#include' '<' ~ '>' [ <identifier> '.h' | <string> ]
    }

    rule if-defined {
        '#if' \N* \n?
    }

    rule elif-defined {
        '#elif' '!'? 'defined' '(' .* ')' <defined-content>* <endif>
    }

    token elif {
        '#elif'
    }

    token endif {
        '#endif'
    }

    rule statement {
        | <comment>
        | <var-decl>
        | <enum-var-decl>
    }

    rule block {
        '{' ~ '}' <statement>*
    }

    rule function {
        <api-decl>? <type> <pointer>* <identifier> '(' ~ ')' <parameters>? ';'
    }

    rule parameters {
        | '...'
        | <const>? <unsigned>? <type> <pointer>* <identifier>? [',' <parameters>]*
    }

    ...

}
```

Here is the full [raylib grammar](https://github.com/vushu/raylib-raku/blob/main/lib/Raylib/Grammar.rakumod)

#### Actions
Now we will define the Actions to handle the cases for converting C to Raku bindings.

Below is simplified pointerization logic which is extracted from in [raylib-raku](https://github.com/vushu/raylib-raku) module that I made.
The [Actions](https://github.com/vushu/raylib-raku/blob/main/lib/Raylib/Actions.rakumod) just contain arrays of strings holding the 
generated Raku bindings and the C-pointerized code.

First we need to pointerize a function only if it's a value type. 
So to deduce this `type` must be an identifier and `pointer` must be nil.

Using Raku's [multiple dispatch](https://docs.raku.org/syntax/multi) and `where` clauses is a very slick way to handle different conditions.

```php
multi method function($/ where $<type><identifier> && !$<pointer>) {
    my $return-type = ~$<type>;
    my $function-name = ~$<identifier>;
    my @current-identifiers;

    # pointerizing the parameters which also extracts the identifiers inside the parameters
    my $pointerized-params = self.pointerize-parameters($<parameters>, @current-identifiers);

    # then we use the current-identifiers for creating the call to the original c-function
    my $original-c-function = self.create-call-func(@current-identifiers, $<identifier>);

    # creating the c-wrapper function;
    my $wrapper = "$return-type $function-name_pointerized ($pointerized_parameters) \{ 
        return $original-c-function;
    \}";

    # when calling .made we convert it to raku types
    my $raku-func = "our sub $function-name $<parameters>.made $<type>.made is export is native(LIBRAYLIB) is symbol('$function-name_pointerized') \l{ * \}";

    @pointerized-bindings.push($wrapper);
    @raku-bindings.push($raku-func);
}

```

Rest of the code basically handles strings creation according to the type and or if the parameter needs to get pointerized.

```php

# map for c to raku types
has @.type-map = "int" => "int32", "float" => "num32", "double" => "num64", "short" => "int16", "char"  => "Str", "bool" => "bool", "void" => "void", "va_list" => "Str";

method type($/) {
    if ($<identifier>) {
        make ~$<identifier>;
    }
    else {
        # translating c type to raku type
        make %.type-map{~$/};
    }
}

# Generating call to original function
method create-call-func(@current-identifiers, $identifier) {
        my $call-func = $identifier ~ '(';
        for @current-identifiers.kv -> $idx, $ident {
            my $add-comma = $idx gt 0 ?? ', ' !! '';
            # If it's a pointer then we must deref
            if ($ident[2]) {
                $call-func ~= ($add-comma ~ "*$ident[1]");
            }
            # No deref
            else {
                $call-func ~= ($add-comma ~ "$ident[1]");
            }
        }
        $call-func ~= ");";
        return $call-func;
}

# Generating pointerized parameters
method pointerize-parameters($parameters, @current-identifiers) {
        my $tail = "";
        # recursively calling pointerize-parameters on the rest 
        my $rest = $parameters<parameters> ?? $parameters<parameters>.map(-> $p {self.pointerize-parameters($p, @current-identifiers)}) !! "";
        if $rest {
            $tail = ',' ~ ' ' ~ $rest;
        }

        # if is value type, do pointerization.
        if ($parameters<type><identifier> && !$parameters<pointer>) {
            return "$($parameters<type>)* $parameters<identifier>" ~ $tail;
        }
        else {
            return "$($parameters<type>) $parameters<identifier>" ~ $tail;
        }
}
```
Again using multiple dispatch and `where` makes it easy to handle different C-types.

```php
# Handling void
multi method parameters($/ where $<pointer> && $<type> eq 'void') {
    make "Pointer[void] \$$<identifier>, {$<parameters>.map: *.made.join(',')}";
}

# Handling int
multi method parameters($/ where $<pointer> && $<type> eq 'int') {
    make "int32 \$$<identifier> is rw, {$<parameters>.map: *.made.join(',')}";
}

# Handling char*
multi method parameters($/ where $<pointer> && $<type> eq 'char' && !$<const>) {
    make "CArray[uint8] \$$<identifier>, {$<parameters>.map: *.made.join(',')}";
}

etc...

```
### Generated code

The code above shows how we use the grammar and action to deduce the generating pointerized C-functions which was the most problematic case.

Ofcourse we also need to handle malloc and free, callbacks, const, unsigned integers, and more.
I left those out since I think the the code above demonstrates that we can use grammar and action to handle tricky cases for creating Raku bindings.

I took me about a week to finish the code generation logic.
The generated Raku bindings are [here](https://github.com/vushu/raylib-raku/blob/main/lib/Raylib/Bindings.rakumod)
and the generated C-pointerized-functions are [here](https://github.com/vushu/raylib-raku/blob/main/resources/raylib_pointerized_wrapper.c)
.

### Conclusion

Overall it was a success!

By using grammar and actions we overcame the painful task of manually making the bindings.

Now Raku is also among the list of [raylib bindings](https://github.com/raysan5/raylib/blob/master/BINDINGS.md).  

See https://github.com/vushu/raylib-raku if curious about the module.

The code isn't my proudest piece of work, it was somewhat quick and dirty. My plan is to revise it and make the code generation re-useable.

After making the bindings I actually made the [chip-8 emulator](https://github.com/vushu/chip-8-raku) as planned.

Well that's it!

Merry Christmas to you all!




