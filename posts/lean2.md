Title: Formally verified CRUD
Date: 2026-07-18
Tags: lean
Description: A couple of weeks ago, I hypothesized that the combination of the Lean Programming Language and AI-assisted coding meant that we were very close to the point where formal verification was realistic for everyday software engineering. Since then I've been experimenting further, and I'm now convinced that we're not close: we're already there.

A couple of weeks ago, I published [Lean-ing into Software Engineering](/lean1.html) in which I hypothesized that the combination of the [Lean Programming Language](https://lean-lang.org) and AI-assisted coding meant that we were very close to the point where formal verification was realistic for everyday software engineering.

Since then I've been experimenting further, and I'm now convinced that we're not close: we're already there.

In this article, I'm going to show you a web application implemented in Lean. As you'll see, it's no more difficult to create a web app this way than it would be in Node, Rails, or any other of the stacks that are popular at the moment. But, crucially, by using Lean we can make hard, mathematically guaranteed, statements about our apps behaviour. We can prove (not demonstrate through testing, but mathematically prove), for example, that certain types of XSS vulnerabilites aren't present.

## tl;dr

Writing a webapp in Lean is just as easy as in any popular framework, but by doing so we get some very valuable benefits:

- Many common errors are impossible: they will be caught immediately at compile time (or even earlier) with no need to write tests.
- Important invariants can be mathematically proven to hold, completely eliminating some classes of both bugs and security vulnerabilities.

[Here](https://github.com/paulbutcher/lean-todomvc) is an implementation of the popular [TodoMVC](https://todomvc.com) web application in Lean, and [here](https://github.com/paulbutcher/lean-webapp) is a minimal Lean webapp which demonstrates the bare minimum necessary.

## A minimal Lean webapp

I'm going to dive straight in and show a minimal but complete Lean webapp to give you the flavour:

<div class="code-block">
<details>
<summary>imports ...</summary>

```lean
import Std.Http
import Html
import Routing

open Std Async
open Std Http Server
open Html
open Routing
```

</details>

```lean
routeTable! App
  [ index := "/",
    greet := "/greet/:name:String" ]

def homePage :=
  document [
    head [ title "Lean Webapp" ],
    body [
      h1 [ "Welcome to lean-webapp" ],
      p [ "A minimal example webapp in Lean." ],
      p [ a { href := App.links.greet "world" } [ "Say hello" ] ],
    ]
  ]

def greetPage (name : String) :=
  document [
    head [ title s!"Hello, {name}!" ],
    body [
      h1 [ s!"Hello, {name}!" ],
      p [ a { href := App.links.index } [ "Back home" ] ],
    ]
  ]

def app := [
    .get App.patterns.index (fun _request => Response.ok.html homePage),
    .get App.patterns.greet (fun name _request => Response.ok.html (greetPage name))
  ] |> toHandler

def main := Async.block do
  let addr := .v4 ⟨.ofParts 127 0 0 1, 0⟩
  let server ← serve addr app
  IO.println s!"Listening on http://{server.localAddr.get!}"
  server.waitShutdown
```

</div>

If you've done any web programming at all it should be immediately obvious what this app does and how it works. The complete project is available [here](https://github.com/paulbutcher/lean-webapp) if you want to play with it yourself.

## Typesafe HTML

Let's see some of the benefits that we gain. Firstly, the DSL that we're using to generate HTML enforces correct HTML structure. So if (say) I change one of the paragraphs in `homePage` to contain a `<div>`, I immediately get an error in the IDE (no need to compile, I see this immediately after I make the change):

<img src="/assets/Screenshot%202026-07-18%20at%2016.49.16.png" width="100%"> 

HTML does not allow a `<div>` to appear within a `<p>`, and the error explains exactly why: paragraphs expect [phrasing content](https://html.spec.whatwg.org/dev/dom.html#phrasing-content) (which doesn't include `<div>`), not [flow content](https://html.spec.whatwg.org/dev/dom.html#flow-content). Lean's type system tells us this immediately in the IDE: no need to compile, no need to run any tests or validate generated HTML.

## Typesafe Routing

Here's another example: let's imagine that we change the type of the parameter passed to `/greet` from `String` to `Nat` (`Nat` is Lean's natural number type). 

```lean
routeTable! App
  [ index := "/",
    greet := "/greet/:name:Nat" ]
```

If I do that, then I immediately see the following error in the IDE:

<img src="/assets/Screenshot 2026-07-18 at 17.01.40.png" width="100%"> 

Lean's type system has worked out that generating a "greet" link with a String is (now) illegal. If you're following along, you'll see that it also flags a second error lower down the file telling us that the type of the handler function is also, now, out of sync with the route it's handling.

Both the above come as benefits of Lean's type system. But Lean also includes a theorem prover which can prove _propositions_ about the code.

## Guaranteed XSS Protection

The HTML library used by our application includes functionality to escape any text embedded within the output. We can see that this is working by putting a `>` character into one of the strings and using Lean's `#eval` command directly within the source file to see the result of the function (this is Lean's equivalent of a REPL):

<img src="/assets/Screenshot 2026-07-18 at 17.13.22.png" width="100%"> 

Of course, there are tests in the library confirming that this escaping does what we expect, but it goes further. It also proves a number of theorems about the code that does the escaping. Here's one of them:

```lean
theorem escape_safe (s : String) : 
  ∀ c ∈ (escape s).toList, c ≠ '<' ∧ c ≠ '>' ∧ c ≠ '"' := _
```

The proposition that this theorem is proving is:

- For any string `s`
  - For all characters `c` which are in the result of calling `escape s` (`escape` is the function that performs the escaping)
    - `c` is not equal to `<`, `>`, or `"`

You can see the whole theorem, along with its proof, [here](https://github.com/paulbutcher/lean-html/blob/main/HtmlTests/Escape.lean#L50). 

In another language, we would might convince ourselves that our escaping is working by providing tests with examples (and indeed, there are some such [tests in the Lean code](https://github.com/paulbutcher/lean-html/blob/main/HtmlTests/Escape.lean#L7)). And that's a great approach, but it's not a guarantee. Lean allows us to go beyond testing and create code that we can rely upon because we have mathematically proven its behaviour.

The above theorem is not sufficient by itself to prove that our generated HTML isn't vulnerable, we also need to make sure that `escape` is used correctly within the rest of the code, but this can also be proven with similar theorems (take a look at the code to convince yourself that every loophole is covered, and let me know if you think I've missed anything).

## Guaranteed Symmetric Routing

The other library used by our example is the routing library which implements `routeTable!`. This provides what's commonly called either "named routes" or "reverse routing" which allows a single route definition to be used both for handling incoming requests and to generate links included within generated HTML.

It's important that the forward and reverse portions of such a routing library agree with each other; we don't want to get into situations where we generate links which our router can't then handle. There have, indeed, been cases of several such bugs in high-profile web frameworks, e.g. [Play](https://github.com/playframework/playframework/issues/3050), [Rails](https://github.com/rails/rails/issues/4164). Here's a theorem within the Lean routing library which guarantees the round-trip; that `parsePattern` (the function that takes a string and converts it into a sequence of path segments) and `renderPattern` (the function that takes a sequence of path segments and returns a link) are perfect inverses of each other:

```lean
theorem parsePattern_renderPattern (segs : List PathSeg) (h : ∀ seg ∈ segs, seg.WellFormed) :
    parsePattern (renderPattern segs) = some segs := _
```

This theorem says:

- For any list of path segments `segs`:
  - Assuming that each segment is well formed
  - Applying `parsePattern` to the result of calling `renderPattern` on `segs` gives us back exactly the segments we started with (the `some` is there because `parsePattern` returns an `Option`).

## TodoMVC

As a more realistic example, [here](https://github.com/paulbutcher/lean-todomvc) is an implementation of the popular [TodoMVC](https://todomvc.com) web application in Lean. As well as the HTML and routing libraries we've already seen, this makes use of an [HTMX](https://htmx.org) [library](https://github.com/paulbutcher/lean-htmx) (built on top of the HTML library) and a [forms library](https://github.com/paulbutcher/lean-forms).

## 