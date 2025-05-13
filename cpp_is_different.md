# C++ is different from languages you're used to
*Unless the language you're used to is C/Rust/etc.*

This article does not teach C++ in all of its glory.

This article teaches *one specific insight* that will help you as you continue to learn all of the rest.

One short text, one footnote, one practice problem.

## The short text:

You're not writing C++ as a joke, you're writing it because you have a problem with very specific requirements that are hard or impossible to solve in the other language you typically write, and so the heavy guns of C++ are called in.

And you run into some annoyance where C++ is weird, and you think "why not fix this? The obvious default is to just" - and: No. C++ will expect you to have specific opinions on how to solve problems, to not be satisfied with a 99% fine default solution, because if a default solution was fine, *you'd use the other language.*

That's it, that's the insight. C++ will expect you to control many things you don't actually need to use for solving your problem today, because *somebody* will need them to solve *some* problem *some* day.

Though some things *can* have good standardized solutions, they just cannot be built-in solutions that everybody have to use. Those go in the standard library.

## The footnote:

Not all the weird things are weird for a good reason, some parts of C++ *are* just bad - it's an old language, built on top of an even older language. But they didn't know better and neither did you.

## The practice problem:

Try to fill in the empty function in this program using *only* language built-ins, without any library features.

The test cases stop at string length 17, but the solution should be about as robust against long strings as the C# example solution.

Interactive editor, thank you Matt Godbolt: https://godbolt.org/z/q8TnYv44Y

```c++
#include <cassert>
#include <string>

std::string second_ordered_by_first(std::string);

int main() {
    assert(second_ordered_by_first("") == "");
    assert(second_ordered_by_first("a1 n1 a2 n2 a3 n3") == "n1 n2 n3");
    assert(second_ordered_by_first("a1 n1 a3 n3 a2 n2") == "n1 n2 n3");
    assert(second_ordered_by_first("a3 n3 a2 n2 a1 n1") == "n1 n2 n3");
    assert(second_ordered_by_first("a3 n2 a1 n3 a2 n1") == "n3 n1 n2");
}

// Your implementation starts here.
// You may find it helpful to create the following supporting features as you fill in the second_ordered_by_first
// - container structures
// - a sort function
// - a string class

std::string second_ordered_by_first(std::string str) {

    return str;
}
```

Note: This is not intended to be an *easy* practice problem. I expect you to stop before you reach the end, unless you spend a *lot* more time than makes sense. Instead, it is intended to shock you out of programming assumptions you may be bringing along from other languages, defaults you expect to just *have*, which C++ does not have at all. C++ doesn't have a built-in *string* class, that's how dire the situation is!

For reference, here's an easy way to do it in `C#` if you're not being careful about performance: https://godbolt.org/z/d86GM4zKn

```csharp
static string SecondOrderedByFirst(string input)
{
        var values = input
            .Split(' ')
            .Chunk(2)
            .OrderBy(c => c.First())
            .Select(c => c.Last());
        
        var first = values.First();
        
        return values
            .Skip(1)
            .Aggregate(first, (x, y) => x + ' ' + y);
}
```