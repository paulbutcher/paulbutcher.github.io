Title: An introduction to Datalog in Flix: Part 1
Date: 2022-10-22
Tags: flix, datalog, logic-programming

This is part 1 of a series. \[Part 1 | [Part 2](datalog2.html) | [Part 3](datalog3.html) | [Part 4](datalog4.html)\]

The code to accompany this series is available [here](https://github.com/paulbutcher/datalog-flix).

## Flix

[Flix](https://flix.dev) is an exciting new programming language which provides the power of strongly typed functional programming with the accessibility of "programmer friendly" languages like Go, Rust, or Ruby.

This article isn't about Flix, though, it's about Datalog, and the reason why Flix plus Datalog creates something greater than the sum of its parts.

## Logic Programming

Logic programming is one of the "big three" programming language paradigms:

**Imperative programming**: Languages like C, C++, Java, JavaScript, Go, Kotlin, …

**Functional programming**: Languages like Haskell, Clojure, Elixir, F#, …

**Logic programming**: Prolog, Datalog, Answer-set Programming, Mercury, …

But ... if it's one of the three major programming paradigms, how come nobody uses it? Perhaps you created a couple of toy Prolog programs at college, but you've almost certainly not used it since then.

There are good reasons why logic programming hasn't caught on outside of a very few specialised areas, but that may be about to change. By including logic programming as a first-class element of the language, Flix allows us to seamlessly move between functional and logic programming, leveraging the strengths of both.

This is analogous to the way that C# embeds database queries directly in the language through LINQ (and of course, you don't use LINQ to implement every single method; instead you use C# for most of your code and drop into LINQ when it makes sense).

In this first article, we'll explore the basics of Datalog. And then in the rest of the series we'll see how the combination of Datalog and Flix brings huge expressive power.

## Facts

Datalog programs are built from two components: *facts* and *rules*. Here are some facts about Game of Thrones:

```flix
GreatHouse("Stark").
GreatHouse("Targaryen").

ParentOf("Rhaegar Targaryen", "Jon Snow").
ParentOf("Aerys II Targaryen", "Daenerys Targaryen").

Coordinates("Kings Landing", 6.697, 12.759).
Coordinates("Winterfell", 5.666, 6.218).
```
You can read these facts as:

* Stark is one of the Great Houses.
* Rhaegar Targaryen is a parent of Jon Snow.
* Kings Landing is located at coordinates 6.697, 12.759.

A fact can have any number of terms of (almost) any type.

## Rules

Rules create new facts by making logical inferences from existing facts. Imagine, for example, that we know the following facts:

```flix
ParentOf("Tywin Lannister", "Cersei Lannister").
ParentOf("Tywin Lannister", "Jaime Lannister").
ParentOf("Cersei Lannister", "Joffrey Baratheon").
ParentOf("Cersei Lannister", "Myrcella Baratheon").
```
Then we can create additional facts about who is who's grandparent with the following rule:

```flix
GrandparentOf(x, y) :- ParentOf(x, c), ParentOf(c, y).
```
Which you can read as "`x` is a grandparent of `y` **if** we can find at least one person `c` where `x` is a parent of `c`, **and** `c` is a parent of `y`".

Here's a complete Flix program which does this:

```flix
def main(): Unit \ IO =
    let got = #{
        ParentOf("Tywin Lannister", "Cersei Lannister").
        ParentOf("Tywin Lannister", "Jaime Lannister").
        ParentOf("Cersei Lannister", "Joffrey Baratheon").
        ParentOf("Cersei Lannister", "Myrcella Baratheon").

        GrandparentOf(x, y) :- ParentOf(x, c), ParentOf(c, y).
    };
    let grandparents = query got select (x, y) from GrandparentOf(x, y);
    println(grandparents)
```
The code within the `#{ ... }` block is our datalog, the rest of the code is the Flix program that it's embedded within. Our Datalog program is executed when we call `query` to extract the new `GrandparentOf` facts.

Here's what it outputs:

```flix
(Tywin Lannister, Joffrey Baratheon) :: (Tywin Lannister, Myrcella Baratheon) :: Nil
```

This is a list of 2-element tuples, with `::` indicating the list "cons" operator and `Nil` the end of the list. We can interpret these results as saying that Tywin Lannister is the grandparent of both Joffrey Baratheon and Myrcella Baratheon.

Here's a slightly different program which uses the same facts to generate additional facts about siblings:

```flix
def main(): Unit \ IO =
    let got = #{
        ParentOf("Tywin Lannister", "Cersei Lannister").
        ParentOf("Tywin Lannister", "Jaime Lannister").
        ParentOf("Cersei Lannister", "Joffrey Baratheon").
        ParentOf("Cersei Lannister", "Myrcella Baratheon").

        SiblingOf(x, y) :- ParentOf(p, x), ParentOf(p, y).
    };
    let siblings = query got select (x, y) from SiblingOf(x, y);
    println(siblings)
```
The key line is:

```flix
SiblingOf(x, y) :- ParentOf(p, x), ParentOf(p, y).
```
Which you can read as "`x` is a sibling of `y` **if** we can find at least one person `p` where `p` is a parent of `x` **and** `p` is a parent of `y`".


And here's what it outputs:

```flix
(Cersei Lannister, Cersei Lannister) :: (Cersei Lannister, Jaime Lannister) ::
(Jaime Lannister, Cersei Lannister) :: (Jaime Lannister, Jaime Lannister) ::
(Joffrey Baratheon, Joffrey Baratheon) :: (Joffrey Baratheon, Myrcella Baratheon) ::
(Myrcella Baratheon, Joffrey Baratheon) :: (Myrcella Baratheon, Myrcella Baratheon) ::
Nil
```

Were you surprised that there were so many answers?

The list is as long as it is because:

1. If `SiblingOf(x, y)` is true, then so is `Sibling(y, x)` (i.e. if you are my sibling, then I am also your sibling).
2. As we've written it, `Sibling` considers anyone with a parent to be their own sibling.

The first point above is probably exactly what we want (i.e. we really do want the sibling relationship to be reflexive). But perhaps we don't want people being their own siblings.

Here's a slightly modified version of our program which adds a *guard* to our rule, which eliminates cases where `x` and `y` represent the same person:

```flix
def main(): Unit \ IO =
    let got = #{
        ParentOf("Tywin Lannister", "Cersei Lannister").
        ParentOf("Tywin Lannister", "Jaime Lannister").
        ParentOf("Cersei Lannister", "Joffrey Baratheon").
        ParentOf("Cersei Lannister", "Myrcella Baratheon").

        SiblingOf(x, y) :- ParentOf(p, x), ParentOf(p, y), if x != y.
    };
    let siblings = query got select (x, y) from SiblingOf(x, y);
    println(siblings)
```
And here's what it outputs:

```flix
(Cersei Lannister, Jaime Lannister) :: (Jaime Lannister, Cersei Lannister) ::
(Joffrey Baratheon, Myrcella Baratheon) :: (Myrcella Baratheon, Joffrey Baratheon) ::
Nil
```

## Conclusion

A Datalog program consists of *facts* and *rules*. Rules generate new facts from existing ones.

In the [next part](datalog2.html) of this series, we'll dig deeper into the integration between Flix and Datalog.

\[Part 1 | [Part 2](datalog2.html) | [Part 3](datalog3.html) | [Part 4](datalog4.html)\]
