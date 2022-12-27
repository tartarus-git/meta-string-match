
*First of all, I apologize for the massive amounts of comments in the code. I know it might look a little ugly, but I like to
write things down that pop into my head and record the results of my research into the language spec as I go, for later reference in case
I need it. I could do this in a separate document I guess, but having it in the context of the code is useful.*

# The Code

It's a compile-time strings to [DFA](https://en.wikipedia.org/wiki/Deterministic_finite_automaton) (Deterministic Finite state Automaton (I think)) converter. I'm working on a version that'll do any arbitrary regex, but this is all I got for now.

If you've got a stream that you want to search through for a set of keywords, or you want to efficiently parse cmdline arguments, this is
an efficient way to do that. It converts your set of strings into a DFA (Simply a state table, so the x coordinate is your character value and
the y coordinate is your current state, and every element contains the next state to go to after that element. You basically jump through
the table and follow the trails laid out for you by my algorithm, and it'll tell you if you've matched any of your given strings and which one
you've matched).

# Usage

Put this in either global scope or local scope (although it seems a lot more efficient to have it in global scope, since then
it doesn't have to get initialized everytime you enter the function (although I guess the compiler could optimize that out, unless of course
you take the address of the table, since then you expect it to be in local scope and the as-if rule would be broken if it weren't there)):

```cpp
CREATE_META_STRING_MATCHER(<put the name of the matcher here>, <put the matcher spec here, see below>, <a comma-separated list of callbacks>);
```

+ `<the name of the matcher>`:

    this is what the variable (which'll be a normal, non-const, non-everything, simple variable) will be called,
    which will contain the DFA table and also has a simple function to interpret the table (although you'll probably want to write your
    own little interpreter for the table based on your use-case).
    Other variables will also be created out of necessity, because the library is forced to do a couple of small things outside of compile-time
    when initializing. Don't worry, your compiler should be able to easily optimize these non-compile-time things out, effectively making
    them compile-time, at no cost to runtime.
    The cool thing is, thanks to the preprocessor's intelligence, these other variables shouldn't interfere with
    other instances of themselves if you
    create multiple string matchers in the same scope. Their names are dependant on the specified matcher name, so only if you
    create two matchers with the same name will the names collide, but that doesn't matter anyway since then your matcher names will collide as well.

+ `<the matcher spec>`:

    string filled with target strings, separated by | characters. Spaces are not ignored, everything is taken quite literally.
    Couple things to be aware of:
    The backslash character allows you to escape characters that normally aren't part of the strings, which in this case is only the | character
    and obviously the backslash character itself.
    IMPORTANT: Because of the way strings work in C++, if you want to escape something in the matcher spec, you've got to write "\\|" for example,
    because C++ has to first escape the backslash in the string literal before I can use it to escape the | character.
    That means that putting a backslash character into a match string is done like this: "\\\\", I know, it's strange.
    ANOTHER THING: empty matcher specs are invalid and you'll get a compile error (you'll also get compilation errors for other errors, like
    putting a backslash at the end of the matcher spec without a character that follows it, or escaping a character that doesn't need to be
    escaped, etc...).
    Also, a series of | characters at the beginning or end of the matcher spec is simply ignored, although you still have to provide the callbacks
    for the empty strings, although these will never ever get called.
    Two | characters that have nothing in between are effectively condensed into one | character, although you still have to provide the callbacks,
    like above.

    It's basically like regex except the only features it has are matching letters and alternations.

+ `<list of callbacks>`:

    these can either be function pointers (void(*)(size_t)) or lambdas with no captures that have a size_t argument,
    or a mixture of both. Each callback is invoked when it's respective match is detected, and the size_t argument is set
    to the index of the last character of the match string in the input stream (NOT the index that is one past the end of the match string).
    Effectively, the list of callbacks must be exactly as long as the number of alternations plus 1.

#### The result of the macro:

Like I said, creates a variable that contains the DFA table and a helper function (and another function to reset the interpreter state,
although you can easily do that by hand).
The important function is match_character(char character):
You call it over and over, putting one character in after the other, and it'll jump through the table for you.
It'll return false if no match has been detected so far, and true as soon as one is detected.
Before returning, it'll call the callback for that match with the stream index as described above.
It'll also reset it's table state back to the first state and start considering every match again, like it did at the start.

**IMPORTANT**: Matches aren't always detected as soon as the match string has completely passed in the input stream. Sometimes,
the table has to wait a couple more characters before it can exclude the possibility that another match might yet come.
The reason is that the table accepts the longest match it can, and one match string can contain another match string,
so it needs to read a bit further sometimes to exclude any other possibilities.
THIS CAN POTENTIALLY BE A HUGE PROBLEM because if you simply keep throwing in characters, then there'll be spots in the input stream
that are effectively skipped by the matcher (the spots after matches that were required to exclude other possibilities).
These won't be checked for matches, which is bad.
Sometimes, this is ok, sometimes it isn't. If you don't like it for your situation, you can simply have a global variable that the
callbacks can store their received stream index into, and when the match_character function returns true, you can
go back in the input stream to one plus the index in the global variable and continue matching from there (remember to update the 
variable responsible for the current input stream index inside the string matcher instance).
Sometimes, the end of one matched string is the beginning of another, and doing the above would still possibly skip matches.
In this case, you've got to specialize the algorithm even more, but I won't get into that here.

**IMPORTANT**: The order in which you specify the strings in the matcher spec has absolutely no bearing on anything. You can also specify the
same string twice, but it is unspecified which of the callbacks will be called when that string actually gets matched.

The table interpreter isn't complicated at all, so if you look in the source code and find the part where it's implemented, that
may answer whatever other things are unclear about the runtime usage of the library.

Also, the repo contains a simple, hastily written test program. Look in test/main.cpp for an example of the usage of the library.

## TODO
+ Find the DFA table in the ELF and see which section it's stored in. Also look at the disassembly and make absolutely sure
it's being used efficiently. I've already skimmed it and it seems to be doing it's job great, but a second look can't hurt.
