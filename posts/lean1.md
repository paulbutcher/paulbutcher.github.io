Title: Lean-ing into Software Engineering
Date: 2026-07-08
Tags: lean

The [Lean](https://lean-lang.org) programming language is revolutionising mathematics. More and more mathematical results have been formalised in Lean and it's increasingly becoming a standard part of the mathematician's toolbox.

Lean didn't start out as a maths tool though; it was originally created to help with software verification: proving that software was correct.

This is not a new idea: software verification was all the rage when I was doing my PhD back in the early 1990s, but it never really took off. It was just too difficult and (outside of a few safety critical projects) nobody really used it in anger.

But, in just the same way that we recently crossed a threshold meaning that it's now realistic for mathematicians to automatically verify their proofs, that same threshold is rapidly approaching for software engineering. Combined with AI-assisted software engineering, it might already have been crossed.

In this article, I'm going to show you why I think that.

## tl;dr

This is quite a long article, and the code behind it is also quite long. But (and this is the whole point) that doesn't matter because **it's now cheap**.

In just a few minutes with Claude Code, I got not just five different implementations of an algorithm, but I also got **cast-iron proofs** that they work as expected and are equivalent to each other (despite taking very different approaches). Not tests which _suggest_ that they work, but a 100% guarantee.

Today, code isn't as important as it used to be. The real value is in the machinery **around** the code which allows you to confirm that it's correct. As Chad Fowler says in [Evaluations Are the Real Codebase](https://aicoding.leaflet.pub/3mb526js42k26):

> The durable asset is the thing that lets you regenerate with confidence: evaluations that encode what the system must do, independent of how any particular implementation does it.

Tests are one way to do that, but they're far from perfect. Lean allows us to create **incontrovertable mathematically verified proofs** that our code is correct, and AI assisted coding allows us to do so quickly and easily.

## A complicated enough problem

For this article, I'm going to use a little programming problem that I used to use when interviewing candidates. I wouldn't use it today, because AI agents have invalidated that approach to interviewing, but it's useful for our purposes because it's both simple enough to explain here, but also complicated enough to have a number of subtle pitfalls and edge cases. It can be solved in multiple different ways, each with their own tradeoffs. It's easy to state, but surprisingly difficult to get right.

We're going to create a function called `partitionWhen` which takes a list and a predicate (a function which takes an element of the list and returns either `true` or `false`). It partitions the list, returning a list of lists, where each sub-list starts whenever the predicate returns `true`.

Here's how we're going to test it:

```lean
#guard [1, 2, 0, 3, 4, 0, 5].partitionWhen (· == 0) == [[1, 2], [0, 3, 4], [0, 5]]
#guard ([] : List Nat).partitionWhen (· == 0) == []
#guard [0, 1, 2].partitionWhen (· == 0) == [[0, 1, 2]]
#guard [1, 2, 3].partitionWhen (· == 0) == [[1, 2, 3]]
#guard [0, 0, 0].partitionWhen (· == 0) == [[0], [0], [0]]
```

* `#guard` is a Lean command which succeeds if the expression it's given is `true` and fails if it's `false`.
  * This makes it a great way to write simple tests like this.
  * **`#guard` runs at compile time**. So these tests will run (and succeed or fail) without having to be compiled and then run; just compiling is enough.
* `(· == 0)` is an _anonymous function_ with the dot "`·`" showing where the argument should go. This is the same as `x => x === 0` in JavaScript or `lambda x: x == 0` in Python.

Here's one way to implement `partitionWhen` in Lean:

```lean
def partitionWhen {α : Type _} (p : α → Bool) : List α → List (List α)
  | [] => []
  | [x] => [[x]]
  | x :: y :: rest =>
    match partitionWhen p (y :: rest) with
    | g :: gs => if p y then [x] :: g :: gs else (x :: g) :: gs
    | [] => [[x]]
```

* This is a polymorphic function which can take lists of any type `α`.
* `p` is our predicate function, which takes an `α` and returns a `Bool`.
* Once we've given `partitionWhen` its predicate, what's left is a function which takes "a list of `α` (`List α`)" and returns "a list of lists of `α` (`List (List α)`)".

I encourage you to try implementing it in your own favourite language: it's slightly trickier than it initially seems. Here's the same approach implemented in a few other languages for comparison:

<div class="code-tabs">

<div class="code-tabs__nav">
<button class="code-tabs__btn is-active" data-lang="javascript">JavaScript</button>
<button class="code-tabs__btn" data-lang="python">Python</button>
<button class="code-tabs__btn" data-lang="clojure">Clojure</button>
<button class="code-tabs__btn" data-lang="elixir">Elixir</button>
<button class="code-tabs__btn" data-lang="haskell">Haskell</button>
<button class="code-tabs__btn" data-lang="fsharp">F#</button>
</div>

<div class="code-tabs__panel is-active" data-lang="javascript">

```javascript
const partitionWhen = (p, xs) => {
  if (xs.length === 0) return [];
  if (xs.length === 1) return [[xs[0]]];

  const [x, y, ...rest] = xs;
  const result = partitionWhen(p, [y, ...rest]);

  if (result.length === 0) return [[x]];

  const [g, ...gs] = result;
  return p(y) ? [[x], g, ...gs] : [[x, ...g], ...gs];
};
```

</div>

<div class="code-tabs__panel" data-lang="python">

```python
def partition_when(p, xs):
    match xs:
        case []:
            return []
        case [x]:
            return [[x]]
        case [x, y, *rest]:
            match partition_when(p, [y, *rest]):
                case []:
                    return [[x]]
                case [g, *gs]:
                    return [[x], g, *gs] if p(y) else [[x, *g], *gs]
```

</div>

<div class="code-tabs__panel" data-lang="clojure">

```clojure
(defn partition-when [p xs]
  (cond
    (empty? xs) []
    (empty? (rest xs)) [(list (first xs))]

    :else
    (let [[x y & more] xs
          result (partition-when p (cons y more))]
      (if (empty? result)
        [(list x)]
        (let [[g & gs] result]
          (if (p y)
            (cons (list x) (cons g gs))
            (cons (cons x g) gs)))))))
```

</div>

<div class="code-tabs__panel" data-lang="elixir">

```elixir
defmodule PartitionWhen do
  def partition_when(_p, []), do: []
  def partition_when(_p, [x]), do: [[x]]

  def partition_when(p, [x, y | rest]) do
    case partition_when(p, [y | rest]) do
      [g | gs] -> if p.(y), do: [[x], g | gs], else: [[x | g] | gs]
      []       -> [[x]]
    end
  end
end
```

</div>

<div class="code-tabs__panel" data-lang="haskell">

```haskell
partitionWhen :: (a -> Bool) -> [a] -> [[a]]
partitionWhen _ []  = []
partitionWhen _ [x] = [[x]]
partitionWhen p (x : y : rest) =
  case partitionWhen p (y : rest) of
    []       -> [[x]]
    (g : gs) -> if p y then [x] : g : gs else (x : g) : gs
```

</div>

<div class="code-tabs__panel" data-lang="fsharp">

```fsharp
let rec partitionWhen (p: 'a -> bool) : 'a list -> 'a list list =
    function
    | [] -> []
    | [x] -> [[x]]
    | x :: y :: rest ->
        match partitionWhen p (y :: rest) with
        | g :: gs -> if p y then [x] :: g :: gs else (x :: g) :: gs
        | [] -> [[x]]
```

</div>

</div>

In just about any language other than Lean, we would now be done. We have our function, we have our tests, time to move on to the next problem.

## The limitations of tests

Tests are great. The widespread adoption of automated testing is one of the best things to happen to Software Engineering over the last several decades.

But tests aren't perfect. However carefully we test, we're just checking a few _examples_ and assuming that because our code works with them, it will work with whatever is thrown at it. How do we know that our tests have covered all the edge cases? Will `partitionWhen` work with any predicate? Will it works with lists of something other than natural numbers?

Sure, we can add more tests, but we have to stop somewhere. As the ISTQB [says](https://astqb.org/istqb-foundation-level-seven-testing-principles/): "Testing shows the presence of defects, not their absence" and "Exhaustive testing is impossible".

## Proving `partitionWhen` correct

Here's a theorem which will be true of any correct implementation of `partitionWhen`:

```lean
theorem partitionWhen_flatten {α : Type _} (p : α → Bool) :
    ∀ (xs : List α), (partitionWhen p xs).flatten = xs := by _
```

As in maths, the upside-down A "`∀`" means "for all". So this is saying "For any list `xs` of any type `α`, if we call `partitionWhen` on that list and then flatten the result, we should get our original list `xs` back".

Flatten takes a list of lists and turns it into a simple list by concatenating all sub-lists. So given `[[1, 2], [0, 3, 4], [0, 5]]` it will return `[1, 2, 0, 3, 4, 0, 5]`

So `partitionWhen_flatten` proves that no element is lost, duplicated, or reordered.

This isn't enough to guarantee that `partitionWhen` is correct on its own, but it is in combination with the following:

```lean
theorem partitionWhen_ne_nil_of_mem {α : Type _} (p : α → Bool) :
    ∀ (xs : List α), ∀ g ∈ partitionWhen p xs, g ≠ [] := by _

theorem partitionWhen_tail_false {α : Type _} (p : α → Bool) :
    ∀ (xs : List α), ∀ g ∈ partitionWhen p xs, ∀ z ∈ g.tail, p z = false := by _

theorem partitionWhen_head_true {α : Type _} (p : α → Bool) :
    ∀ (xs : List α), ∀ g ∈ (partitionWhen p xs).tail, ∀ z, g.head? = some z → p z = true := by _
```

In turn, these say:

* no sub-list is spuriously empty.
* sub-lists are maximal: nothing inside a sub-list (past its own head) should have split off.
* sub-lists are complete: every sub-list after the first genuinely starts at a trigger.

It's worth taking a moment to convince yourself that these theorems, taken together, guarantee that `partitionWhen` is doing what we expect.

So how do we go about proving that these theorems are true?

## How Lean proves theorems

[There is a whole book on how Lean proves theorems](https://lean-lang.org/theorem_proving_in_lean4/) and I'm not going to try to cover everything it says here. But it boils down to two things:

* **Lean's type system:** A theorem is a proof of a _proposition_, and for many years we've known that (given a sufficiently powerful type system) propositions can be represented as types. This is the [Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry–Howard_correspondence) commonly known as "propositions as types".
  * Proving a proposition is exactly the same as proving that the type that represents that proposition is _inhabited_, which just means that we can find one example of a value of that type.
  * It's really that simple: create a single value of that type and the fact that you've done so is also a proof that the proposition represented by that type is true.
* **Lean's Theorem Prover:** Lean provides a whole array of _tactics_ who's entire job is finding these values.

And that's it. Find an instance of the type and you're done.

Of course, that doesn't mean that finding that instance is easy 😜.

## How AI changes things

A couple of years ago, our only choice would have been to construct these proofs by hand. Lean's theorem prover is a great help, but it's still a laborious process for anything other than the simplest of proofs. So Lean would have been nothing more than an interesting footnote for most software engineers.

Today, however, AI agents are becoming very good indeed at generating proofs.

[Here](https://github.com/paulbutcher/partition-when/blob/main/PartitionWhenTests/Correctness.lean#L5) are complete proofs of all four of the theorems above. They run to around 160 lines of code, so I'm not going to reproduce them here. And as you can see they aren't the easiest to read or to follow. But:

* They were created entirely automatically: I did nothing beyond setting Claude Code going (and paying for the tokens 🤷).
* We don't need to understand them in depth. We just need to know that the theorems we care about are true.

In this, we're different from mathematicians. In most cases, a mathematician won't be happy with a just knowing that something is true, they will also want to know _why_ it's true. Ideally they don't just want a proof of whatever it is they're working on, they want a simple, elegant proof.

But in our case, we just want to know that the code we've written (or, more likely, that our AI agent has written on our behalf) is correct. An ugly proof that it's correct is just fine. The theorem is the important bit, the proof an implementation detail (think of it like you think of the machine code that comes out of your compiler).

## There's more than one way

Our implementation of `partitionWhen` doesn't just work, but it provably works, which puts it ahead of 99.999% of all code ever written (which is nice). But it's not perfect.

Like most functional langauges Lean supports [tail recursion](https://lean-lang.org/functional_programming_in_lean/Programming___-Proving___-and-Performance/Tail-Recursion/) meaning that recursion won't blow up the stack. But this only works if calling itself is the last thing that a function does, which isn't true of `partitionWhen`.

No problem, it's not too difficult to create a tail recursive version:

```lean
def partitionWhenTR {α : Type _} (p : α → Bool) (xs : List α) : List (List α) :=
  go xs [] []
where
  go : List α → List α → List (List α) → List (List α)
    | [], cur, acc => if cur.isEmpty then acc.reverse else (cur.reverse :: acc).reverse
    | x :: xs, cur, acc =>
      if p x && !cur.isEmpty then
        go xs [x] (cur.reverse :: acc)
      else
        go xs (x :: cur) acc
```

Or, alternatively, we can make use of the `foldl` function provided by the Lean standard library (some languages call this `reduce`):

```lean
def partitionWhenFold {α : Type _} (p : α → Bool) (xs : List α) : List (List α) :=
  let (groups, cur) := xs.foldl
    (fun (acc : List (List α) × List α) x =>
      let (groups, cur) := acc
      if p x && !cur.isEmpty then
        (cur.reverse :: groups, [x])
      else
        (groups, x :: cur))
    ([], [])
  (if cur.isEmpty then groups else cur.reverse :: groups).reverse
```

There are another couple of implementations in the GitHub repository, one [using `takeWhile` and `dropWhile`](https://github.com/paulbutcher/partition-when/blob/main/PartitionWhen/Basic.lean#L48), and another [using `span`](https://github.com/paulbutcher/partition-when/blob/main/PartitionWhen/Basic.lean#L84).

I'm going to assert that all the above give exactly the same result as our first implementation, but it's not immediately obvious from looking at them.

In most languages, all we could do would be to test our new implementations and hope. In Lean, we could prove the same theorems as we proved for `partitionWith`. But we have another choice: we can prove that these new implementations are _equivalent_ to our first implementation.

## Proving equivalence

Here's an innocent looking little theorem:

```lean
theorem partitionWhenTR_eq_partitionWhen {α : Type _} (p : α → Bool) (xs : List α) :
    partitionWhenTR p xs = partitionWhen p xs := _
```

It literally just says "`partitionWhenTR` is equal to `partitionWhen`". Easy to say, but how on Earth do you go about proving it? In general, it's not possible for an arbitrary program (this is very closely related to the famous [Halting Problem](https://en.wikipedia.org/wiki/Halting_problem)), but it is possible for two programs that are guaranteed to terminate. And one of the nice things we haven't yet mentioned about Lean is that it is able to prove whether or not a function terminates (in fact every Lean function is guaranteed to terminate unless it's marked [`partial`](https://lean-lang.org/doc/reference/latest/Definitions/Recursive-Definitions/#partial-functions))

[Here](https://github.com/paulbutcher/partition-when/blob/main/PartitionWhenTests/Correctness.lean#L162) are proofs that all of the different implementations are equivalent. In this case, the proofs runs to over 500 lines of code, but again, these are 500 cheap lines, automatically generated by AI.

Which means that if we have some code which uses one of our `partitionWhen` implementations and we decide to swap to another, we can **guarantee** that its behaviour will be unchanged.

## Is Lean ready for production?

Lean is a full featured programming language capable of doing anything you can do in just about any other language. It's not as fast as some languages, but much faster than others (Python, we're looking at you).

What might stop you from using Lean in production is that all the focus has been on maths. [Mathlib](https://mathlib-initiative.org), the repository of reusable mathematical proofs is large and growing quickly; many of the things a working mathematician would expect to find are already there. The same is not true from the point of general software engineering. [Reservoir](https://reservoir.lean-lang.org), the repository of Lean packages, isn't especially well populated and a software engineer hoping to find a pre-built package which addresses their use case is likely to be disapointed. Although there are some gems in there (a [fully verified HTTPS server](https://reservoir.lean-lang.org/@AfonsoBitoque/LeanServer), for example).

But the benefit of knowing (not guessing, not hoping) that your code works is huge. With that in mind, I can only imagine that Lean will be of increasing importance to software engineers over time.

## Notes

All the code in this article was created in VSCode configured as described [here](https://lean-lang.org/install/), Claude Code Pro, [lean-lsp-mcp](https://github.com/oOo0oOo/lean-lsp-mcp) and [Lean 4 Skills](https://github.com/cameronfreer/lean4-skills/).
