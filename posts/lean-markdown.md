Title: A (somewhat) formally verified implementation of Markdown
Date: 2026-08-08
Tags: lean
Description: Further evidence that formally verifying software as a matter of course is feasible.

Over the last couple of weeks, I've been working on a personal project in [Lean](https://lean-lang.org). Six months ago I would have used Clojure but the combination of Lean and AI assisted software engineering is making formal verification so easy, why wouldn't I choose the additional safety it brings?

As part of this work, I needed a Markdown parser/renderer. The obvious route would be to wrap something like [cmark-gfm](https://github.com/github/cmark-gfm/) using Lean's FFI but as an experiment I decided to see how far I could get with a pure Lean implementation, and as part of that how far I could go to formally verifying the code's behaviour.

## tl;dr

[lean-markdown](https://github.com/paulbutcher/lean-markdown) is a (somewhat) formally verified implementation of both
[CommonMark 0.31.2](https://spec.commonmark.org/0.31.2/) and 
[GitHub Flavored Markdown (GFM)](https://github.com/github/cmark-gfm/).

The formally proven parts are:

- **Total**: never panics or loops on any input, including adversarial input.
- **Safe**: proved to never let an AST leaf's string content produce unescaped HTML 
  markup, or break out of an attribute.
- **Well-formed**: for input with no embedded raw HTML, output is proved well-formed
  HTML.

Both CommonMark and GFM pass raw HTML through verbatim by design. For untrusted input, lean-markdown provides `renderHtmlSafe` which guarantees the output is both safe and well formed, including adversarial input.

What is not formally verified is conformance with the CommonMark and GFM specifications; it passes every test in the official CommonMark and cmark-gfm suites, but these are tests, not formal verification.

## Could we go further?

I'd love to explore whether there's some way to go further and prove that the implementation not only passes the test suites, but extract propositions from the specification and prove them. Perhaps something like [Allium](https://allium-lang.org) could be helpful here?

But still, it seems clear that Lean is bringing real value: the ability to use a library like this and be certain (not confident, certain) that it won't crash or generate ill-formed output is a real step forwards.
