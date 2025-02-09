Title: An introduction to Datalog in Flix: Part 4
Date: 2022-10-26
Tags: flix, datalog, logic-programming

This is part 4 of a series. \[[Part 1](datalog1.html) | [Part 2](datalog2.html) | [Part 3](datalog3.html) | Part 4\]

The code to accompany this series is available [here](https://github.com/paulbutcher/datalog-flix).

In the previous part of this series, we saw how to use lattice semantics to calculate all our degrees of separation within the Game of Thrones dataset in one pass. But we didn't quite get to our final solution because we weren't yet calculating counts of the number of characters at each degree.

In this part, we'll tie off that loose end, and discover why lattice semantics are "lattice" semantics in the process.

Of course, we could just count up our values in Flix (not Datalog) code, and that would be a perfectly reasonable approach. But as we'll see Datalog gives us a really elegant solution by leveraging *partial ordering*.

## Partial Ordering

So far we've been working with simple integers, and integers are (surprise!) *ordered*. So 1 is less than 2 and 3 is less than 126 and so on.

In fact, not only are integers ordered, they *totally* ordered. You can pick any pair of numbers `a` and `b`, and at least one of `a <= b` or `b <= a` will be true (they might both be true if `a` equals `b`).

But, not every set of values is totally ordered. A real world example of a partial ordering is ancestry; given two people a and b, it might easily be the case that *neither* "a is an ancestor of b" *nor* "b is an ancestor of a" is true.

For example, "Eddard Stark is an ancestor of Arya Stark" is true. But neither "Cersei Lannister is an ancestor of Tyrion Lannister" nor "Tyrion Lannister is an ancestor of Cersei Lannister" are true.

However, we can ask "who is the most recent common ancestor of Cersei and Tyrion" (in this case Tywin Lannister). We generally refer to this as the *least upper bound*.

## Least Upper Bound

In part 3, we said that lattice semantics chose the "maximum" value from all possible values. This was a simplification: they actually chose the least upper bound.

For integers, which we were working with in part 3, the least upper bound *is* the maximum value. But for other types, those that are partially ordered (as we saw with ancestry above), the least upper bound could be something else.

An interesting example is sets. It may not be the case that either "set a is a subset of set b" or "set b is a subset of set a" is true. But they will always have a least upper bound which is the *union* of a and b.

For example, the least upper bound of `Set#{"Tyrion Lannister", "Cersei Lannister"}` and `Set#{"Tyrion Lannister", "Arya Stark"}` is `Set#{"Tyrion Lannister", "Cersei Lannister", "Arya Stark"}`.

> This is where the "lattice" in lattice semantics comes from: a partially ordered set which defines a least upper bound is called a [*lattice*](https://en.wikipedia.org/wiki/Lattice_(order)) in mathematics.

Happily, this is exactly what we need to solve our degrees of separation problem.

## Six Degrees of Separation, Take 3

As a reminder, here were the rules we used in part 3 of this series:

```flix
        Degree(x; Down(0)) :- Root(x).
        Degree(x; n + Down(1)) :- Degree(y; n), Related(y, x).
```
Here are the new rules we're going to add:

```flix
        AggregatedDegree(n; Set#{x}) :- fix Degree(x; n).
        DegreeCount(n, Set.size(s)) :- fix AggregatedDegree(n; s).
```
We start by inferring new `AggregatedDegree` facts from the `Degree` facts we calculated previously. We're using the fact that the least upper bound of a collection of sets is the union of those sets, so the value on the right side of the semicolon will be the union of all of the character names in `Degree`.

> You might be wondering what the `fix` is for in our new rules? If you look at the two sides of the rule (either side of the `:-`) you can see that on the left we're using `n` as a normal value (it's on the left of the semicolon), whereas on the right it's a lattice value (it's on the right of the semicolon). Flix requires that we use `fix` if we ever mix a value this way; it ensures that we completely calculate all the `Degree` facts before starting to create `AggregateDegree` facts.

So now we have a number of `AggregatedDegree` facts, one for each degree of separation, where the right hand side is a set of all the characters of that degree. The final step is to convert those facts into `DegreeCount` facts by finding the size of each set.

Here's the whole thing. As you can see, it's even more elegant than the solution we came up with in part 2 (which was already much simpler than we could have achieved without using Datalog).

```flix
def main(): Unit \ IO =

    let relationshipTypes = "parents" :: "parentOf" :: "killed" :: "killedBy" ::
        "serves" :: "servedBy" :: "guardianOf" :: "guardedBy" :: "siblings" ::
        "marriedEngaged" :: "allies" :: Nil;
    let relationships = Json.getRelationships(relationshipTypes);

    let related = inject relationships into Related;

    let rules = #{
        Degree(x; Down(0)) :- Root(x).
        Degree(x; n + Down(1)) :- Degree(y; n), Related(y, x).
        AggregatedDegree(n; Set#{x}) :- fix Degree(x; n).
        DegreeCount(n, Set.size(s)) :- fix AggregatedDegree(n; s).
    };

    let root = inject "Tyrion Lannister" :: Nil into Root;

    query rules, related, root select (d, c) from DegreeCount(d, c) |>
        List.foreach(match (d, c) -> println("Separated by degree ${d}: ${c}"))
```
And, for completeness, here's what it outputs:

```flix
Separated by degree 6: 2
Separated by degree 5: 14
Separated by degree 4: 60
Separated by degree 3: 104
Separated by degree 2: 56
Separated by degree 1: 6
Separated by degree 0: 1
```

## Conclusion

That's it for our journey through Datalog and Flix. Please do experiment with other problems which can leverage Datalog: I'd love to see what you come up with!

\[[Part 1](datalog1.html) | [Part 2](datalog2.html) | [Part 3](datalog3.html) | Part 4\]
