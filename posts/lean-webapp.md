Title: A complete production webapp in Lean
Date: 2026-09-01
Tags: lean
Description: Not just a demo: Passwordless sign-in, SQL migrations, telemetry, an LLM assistant panel via Bedrock, an MCP endpoint your own agent can use, and IaC deployment to AWS Lambda.

A few weeks ago, I published a [Lean implementation of the popular TodoMVC app](./lean2.html). That was a nice proof of existence, but every software engineer knows that the gap from a cute example to a fully working application is huge.

So I'm pleased to be able to show a [complete, production-ready web app in Lean](https://github.com/paulbutcher/lean-todomvc-max); TodoMVC plus everything a production application needs around it: passwordless sign-in, SQL migrations, telemetry, an LLM assistant panel via Bedrock, an MCP endpoint your own agent can use, and IaC deployment to AWS Lambda:

<img src="/assets/Screenshot.png" alt="My image" style="width:100%; height:auto;">

## Why Lean?

Lean is a strongly typed functional language with a built-in theorem prover. This allows us to make some very strong guarantees, including:

<dl>
  <dt>**Totality**</dt>
  <dd>Nothing in this application is `partial` and nothing in it can panic, and the same holds of every library it's built upon. A loop that reads until its input runs out carries a bound and a proof that it decreases.</dd>
  <dt>**Security properties are theorems**</dt>
  <dd>Nothing in this application is `partial` and nothing in it can panic, and the same holds of every library it's built upon. A loop that reads until its input runs out carries a bound and a proof that it decreases.</dd>
  <dt>**Markup is typed and formally verified**</dt>
  <dd>A `<div>` inside a `<p>` is a type error, text content is escaped on the way in, and `Node.render_wellFormed` proves that what comes out is well-formed HTML.</dd>
  <dt>**Routes are strongly typed**</dt>
  <dd>A handler of the wrong arity or the wrong type does not compile, and a link cannot drift from the route that serves it.</dd>
  <dt>**Markdown is formally verified**</dt>
  <dd>[lean-markdown](lean-markdown.html) is total, never panicking or looping on any input including adversarial input. `renderHtmlSafe` is proved to emit well-formed HTML in which no string from the document can produce markup or break out of an attribute.</dd>
  <dt>**What an agent was granted bounds what it can reach**</dt>
  <dd>A token that was not granted `todos:write` reaches no tool that changes anything (`nothing_mutates_without_write`).</dd>
  <dt>**Encodings are proved to round-trip**</dt>
  <dd>What is written to a chat row is what is read back from it (`toMsg_ofMsg`), which matters because the conversation is replayed to the model in full on every turn. Underneath, leancrypto proves `decode (encode bytes) = some bytes` for hex, base64, base64url and Crockford base32, that its modular exponentiation agrees with `base ^ exponent % modulus`, and that its early-exit-free comparison is equality.</dd>
</dl>

## Benefits for AI-assisted software engineering

The above benefits count for a lot by themselves, but they buy a great deal more than just confidence that the software won't crash, infinite loop, or generate malformed output. Lean's combination of strong type safety and formal guarantees provides _really_ strong guardrails for an AI coding agent. My experience has been that the results are much higher quality, take less time, require fewer tokens, and require far (far!) fewer debugging round-trips. It's a cliché (but like all clichés, it exists for a reason) but in Lean, broadly speaking, "if it compiles it works".

If you can, I'd encourage you to try Lean. It's much easier than it might appear, and the benefits are huge.