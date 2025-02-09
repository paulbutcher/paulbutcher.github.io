Title: An introduction to Datalog in Flix: Part 3
Date: 2022-10-25
Tags: flix, datalog, logic-programming

This is part 3 of a series. \[[Part 1](datalog1.html) | [Part 2](datalog2.html) | Part 3 | [Part 4](datalog4.html)]

The code to accompany this series is available [here](https://github.com/paulbutcher/datalog-flix).

## Harder, Better, Faster, Stronger

It's very common that we want to find the "best" solution to a problem. Perhaps we want the shortest route between two points? Or the fastest? Or the most efficient? Or...?

Flix's implementation of Datalog provides a feature known as "lattice semantics" (we'll see why "lattice" later) which allows us to solve exactly these kinds of problems. To illustrate, consider the following function:

```flix
def foo(): List[(String, Int32)] =
    let rules = #{
        Foo("x", 1).
        Foo("x", 2).
        Foo("x", 5).
        Foo("y", 2).
        Foo("y", 3).
    };
    query rules select (x, y) from Foo(x, y)
```
This, unsurprisingly, just returns exactly the facts that we specified:

```flix
(x, 1) :: (x, 2) :: (x, 5) :: (y, 2) :: (y, 3) :: Nil
```
But now, let's modify it so that it uses lattice semantics:

```flix
def foo(): List[(String, Int32)] =
    let rules = #{
        Foo("x"; 1).
        Foo("x"; 2).
        Foo("x"; 5).
        Foo("y"; 2).
        Foo("y"; 3).
    };
    query rules select (x, y) from Foo(x; y)
```
All we've done is replace the commas (`,`) in our original `Foo` rules with semicolons (`;`). What it now returns is:

```flix
(x, 5) :: (y, 3) :: Nil
```

Switching to semicolons instructs Datalog to aggregate all the facts that have the same values on the left of the semicolon into a single fact, where the value on the right is the "maximum" of all the facts that are being combined. In our example above, the maximum value is 5 when the LHS is "x", and 3 when it's "y".

> There is some subtlety in what we mean when we say "maximum", as we'll see later, but for simple integers then what we get is exactly the same as if we had used `Int32.max`.

It seems like a small change, but it's a small change that dramatically increases the power at our fingertips.

## Minimums

Finding maximum values is great, but what if we want to find minimums instead? Flix allows us to reverse the sorting order of any type by using `Down`, so `Down[Int32]` sorts in the opposite order to `Int32`:

```flix
flix> 1 < 3
true
flix> 4 < 2
false
flix> Down(1) < Down(3)
false
flix> Down(3) < Down(2)
true
```
Using `Down` in Datalog lattice semantics has exactly the effect you might imagine, allowing us to find minimum values instead of maximum values:

```flix
def bar(): List[(String, Down[Int32])] =
    let rules = #{
        Bar("x"; Down(1)).
        Bar("x"; Down(2)).
        Bar("x"; Down(5)).
        Bar("y"; Down(2)).
        Bar("y"; Down(3)).
    };
    query rules select (x, y) from Bar(x; y)
```
Which returns:

```flix
(x, 1) :: (y, 2) :: Nil
```
So much for theory, let's see how this helps us solve a real problem.

## Six Degrees of Separation, Take 2

In [part 2](datalog2.html) of this series, we used Datalog to calculate degrees of separation between characters in Game of Thrones. The combination of logic programming and functional programming gave us a very simple solution. Now, we're going to use lattice semantics to make it even better.

As a refresher, here's the code from part 2 which creates our `Related` facts:

```flix
    let relationshipTypes = "parents" :: "parentOf" :: "killed" :: "killedBy" ::
        "serves" :: "servedBy" :: "guardianOf" :: "guardedBy" :: "siblings" ::
        "marriedEngaged" :: "allies" :: Nil;
    let relationships = Json.getRelationships(relationshipTypes);

    let related = inject relationships into Related;
```
And here is code which uses lattice semantics to calculate the degree of separation of every character from some root character (in this case Tyrion) in a single pass:

```flix
    let rules = #{
        Degree(x; Down(0)) :- Root(x).
        Degree(x; n + Down(1)) :- Degree(y; n), Related(y, x).
    };

    let root = inject "Tyrion Lannister" :: Nil into Root;

    println(query rules, related, root select (x, d) from Degree(x; d))
```
Before we go through this to see how it works, here's the (truncated) output:

```flix
(Aegon Targaryen, 3) :: (Aeron Greyjoy, 3) :: (Aerys II Targaryen, 2) ::
(Akho, 3) :: (Alliser Thorne, 3) :: (Alton Lannister, 2) :: ...
```
To see how this works, imagine what would happen if we wrote the above *without* lattice semantics:

```flix
        Degree(x, 0) :- Root(x).
        Degree(x, n + 1) :- Degree(y, n), Related(y, x).
```
This can be interpreted as saying: Character `x` is related to our root character by degree `n + 1` if we can find any character `y` which is related by degree `n`, and where `y` is related to `x`.

So Cersei, for example, is related to Tyrion by degree 1 because Tyrion is related to Tyrion by degree 0, and Tyrion is related to Cersei.

But ... Cersei is also related to Tyrion by degree 2 because Tywin is related to Tyrion by degree 1, and Tywin is related to Cercei. And Cersei is related to Tyrion by degree 3 because Jaime is related to Tyrion by degree 2, and Jaime is related to Cersei. And so on.

By contrast, Arya is not related to Tyrion at degree 1 or 2, but is related to him by degree 3, because Eddard Stark is related to Tyrion at degree 2, and Eddard is related to Arya. And Arya is related to Tyrion by degree 4 because Walder Frey is related to Tyrion at degree 3, and Walder is related to Arya. And so on.

So, as you can probably already see, with non-lattice semantics this will never terminate; we'll just keep calculating higher and higher degrees of separation until we run out of either memory or patience.

But, with lattice semantics (and given that we're using `Down` to reverse the ordering) we can constrain all of our degrees (the numbers on the right of the semicolon) to a single minimum value (whichever is the lowest they can ever take), which means:

1. Our function terminates (always helpful!), and
2. The degree calculations always return the length of the "shortest path" from Tyrion to every other character.

## Conclusion

So we're almost there: we now have a list of every character along with their minimum degree of separation from Tyrion. But if we're going to duplicate what we created in part 2, we need to go further and count how many characters there are at each degree of separation. We'll see how to do that in the [next part](datalog4.html) of this series.

\[[Part 1](datalog1.html) | [Part 2](datalog2.html) | Part 3 | [Part 4](datalog4.html)\]
