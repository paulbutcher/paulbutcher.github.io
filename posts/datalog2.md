Title: An introduction to Datalog in Flix: Part 2
Date: 2022-10-23
Tags: flix, datalog, logic-programming

This is part 2 of a series. \[[Part 1](datalog1.html) | Part 2 | [Part 3](datalog3.html) | [Part 4](datalog4.html)]

The code to accompany this series is available [here](https://github.com/paulbutcher/datalog-flix).

## Injecting facts

In part 1 of this series, we used Datalog rules to infer new facts about characters in Game of Thrones.

But where do our initial facts come from? In part 1 we simply included them as part of our Datalog, but this isn't a scalable approach. We don't want to have to manually re-enter all the details of all of the characters in Game of Thrones!

What's more likely is that we'll read these facts from some external source; perhaps a database or a JSON file. This is where the power of Flix integration comes into play; we can use Flix, a powerful general purpose language, to read our data and get it into the right format. We then *inject* it into Datalog facts.

So instead of writing this (from part 1):

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
We can instead write:

```flix
def main(): Unit \ IO =
    let parentList = ("Tywin Lannister", "Cersei Lannister") ::
        ("Tywin Lannister", "Jaime Lannister") ::
        ("Cersei Lannister", "Joffrey Baratheon") ::
        ("Cersei Lannister", "Myrcella Baratheon") :: Nil;

    let parents = inject parentList into ParentOf;

    let got = #{
        SiblingOf(x, y) :- ParentOf(p, x), ParentOf(p, y), if x != y.
    };

    let siblings = query got, parents select (x, y) from SiblingOf(x, y);

    println(siblings)
```
Or, more realistically, we read our data from some external file (in this case, Jeffrey Lancaster's amazingly detailed [Game of Thrones dataset](https://github.com/jeffreylancaster/game-of-thrones)):

```flix
def main(): Unit \ IO =

    let parents = inject Json.getParents() into ParentOf;

    let got = #{
        SiblingOf(x, y) :- ParentOf(p, x), ParentOf(p, y), if x != y.
    };

    let siblings = query got, parents select (x, y) from SiblingOf(x, y);

    println(siblings)
```
Which gives us the following (truncated!) output:

```flix
(Aegon Targaryen, Jon Snow) :: (Aegon Targaryen, Rhaenys Targaryen) ::
(Arya Stark, Bran Stark) :: (Arya Stark, Rickon Stark) ::
(Arya Stark, Robb Stark) :: (Arya Stark, Sansa Stark) ::
(Benjen Stark, Brandon Stark) :: (Benjen Stark, Eddard Stark) :: ...
```
## Chaining Datalog

The ability to move data back and forth between Datalog and Flix makes it very easy to use Datalog as part of solving a larger problem. To illustrate this, let's consider working out whether Game of Thrones obeys the ["six degrees of separation"](https://en.wikipedia.org/wiki/Six_degrees_of_separation) rule (the idea that all people are six or fewer connections away from each other; you might have come across it from the "Six Degrees of Kevin Bacon" game, or a mathematician's Erdős number).

As well as parentage, the dataset we've been using includes all kinds of other relationships between characters such as alliances, marriages and engagements, who serves whom, who killed whom, etc. Taking all of these relationships together, how many degrees of separation does it take from a major character (say Tyrion) to cover the entire cast of characters?

(You can find the complete source [here](https://github.com/paulbutcher/datalog-flix/tree/master/part2-3)).

First, we extract our long list of relationships from our dataset:

```flix
    let relationshipTypes = "parents" :: "parentOf" :: "killed" :: "killedBy" ::
        "serves" :: "servedBy" :: "guardianOf" :: "guardedBy" :: "siblings" ::
        "marriedEngaged" :: "allies" :: Nil;
    let relationships = Json.getRelationships(relationshipTypes);
```
We then inject that list into Datalog as a series of `Related` facts, just like we've seen above:

```flix
    let related = inject relationships into Related;
```
So now we have a set of facts like `Related("Tyrion Lannister", "Shae")` because Tyrion killed Shae, and `Related("Arya Stark", "Nymeria")` because Arya is guarded by Nymeria, and so on.

Next, here is a Datalog rule which, given a set of characters we've already found, finds the characters with the next degree of separation:

```flix
    let rules = #{
        NextDegree(x) :- AlreadyFound(y), Related(y, x), not AlreadyFound(x).
    };
```
So a character `x` is in the next degree of separation if there is at least one character `y` who we have already and where `y` is related to `x`. Finally we exclude characters that we've already found.

Here's a function which uses the above to repeatedly calculate degrees of separation until we've exhausted all our characters:

```flix
    def degreesOfSeparation(deg: Int32, cs: List[String]): Unit \ IO = {
        let alreadyFound = inject cs into AlreadyFound;
        let nextDegree = query rules, alreadyFound, related select (x) from NextDegree(x);
        match List.length(nextDegree) {
            case count if count > 0 =>
                println("Separated by degree ${deg}: ${count}");
                degreesOfSeparation(deg + 1, cs ::: nextDegree)
            case _ =>
                println("Nobody separated by degree ${deg}")
        }
    };
```
This function takes `deg` (a number representing the current degree of separation we're looking at) and `cs` a list of characters we've already found.

We start by creating our `AlreadyFound` facts from this list of characters, and then use the Datalog `rules` we defined above, along with our `AlreadyFound` and `Related` facts to generate `NextDegree` characters.

`List.length(nextDegree)` returns the length of this list, and we use `match` to determine whether this number is greater than zero.

If it is, then we output the number of characters we just found, and recursively call `degreesOfSeparation` with `deg` increased by one, and a new set of characters which includes both our original root set and the characters we just found.

Finally, we kick the whole thing off by calling `degreesOfSeparation` with our initial root character, Tyrion:

```flix
    degreesOfSeparation(1, "Tyrion Lannister" :: Nil)
```
Here's what we get when we run it:

```flix
Separated by degree 1: 6
Separated by degree 2: 56
Separated by degree 3: 104
Separated by degree 4: 60
Separated by degree 5: 14
Separated by degree 6: 2
Nobody separated by degree 7
```

So yes, it does look like Game of Thrones does obey the six degrees of separation rule.

Here's the whole thing. I suggest trying to implement this in your favourite language to see just how easy the combination of Flix and Datalog makes this:

```flix
def main(): Unit \ IO =

    let relationshipTypes = "parents" :: "parentOf" :: "killed" :: "killedBy" ::
        "serves" :: "servedBy" :: "guardianOf" :: "guardedBy" :: "siblings" ::
        "marriedEngaged" :: "allies" :: Nil;
    let relationships = Json.getRelationships(relationshipTypes);

    let related = inject relationships into Related;
    let rules = #{
        NextDegree(x) :- AlreadyFound(y), Related(y, x), not AlreadyFound(x).
    };

    def degreesOfSeparation(deg: Int32, cs: List[String]): Unit \ IO = {
        let alreadyFound = inject cs into AlreadyFound;
        let nextDegree = query rules, alreadyFound, related select (x) from NextDegree(x);
        match List.length(nextDegree) {
            case count if count > 0 =>
                println("Separated by degree ${deg}: ${count}");
                degreesOfSeparation(deg + 1, cs ::: nextDegree)
            case _ =>
                println("Nobody separated by degree ${deg}")
        }
    };

    degreesOfSeparation(1, "Tyrion Lannister" :: Nil)
```
## Conclusion

Flix allows us to seamlessly move data back and forth between Flix and Datalog by `inject`ing facts from Flix to Datalog and `query`ing data from Datalog to Flix. This allows us to use the strengths of a general purpose language like Flix for things it's good at (e.g. reading JSON files) and Datalog for things that it's good at (e.g. inferring new facts from existing ones).

In the next part of this series, we'll look at one of the most powerful aspects of Flix's implementation of Datalog: lattice semantics.

\[[Part 1](datalog1.html) | Part 2 | [Part 3](datalog3.html) | [Part 4](datalog4.html)\]
