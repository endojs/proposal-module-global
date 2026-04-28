# Module global

Stage: 1

Champions:

- Zbyszek Tenerowicz (ZTZ), Consensys, @naugtur
- Kris Kowal (KKL), Agoric, @kriskowal
- Richard Gibson (RGN), Agoric, @gibson042
- Mark S. Miller (MM), Agoric, @erights

## Problem statement 

A way to evaluate a module and its dependencies in the context of a new global scope within the same Realm



> This proposal picks up from the previous proposal for
> [Evaluators](https://github.com/tc39/proposal-compartments/blob/7e60fdbce66ef2d97370007afeb807192c653333/3-evaluator.md)
> from the [HardenedJS](https://hardenedjs.org) [`Compartment` proposal][proposal-compartments] and depends upon [proposal-import-hook][],
> [proposal-esm-phase-imports][], and [proposal-source-phase-imports][].

## Motivation

### Domain Specific Languages

Tools like Mocha, Jest, and Jasmine install the verbs and nouns of their
domain-specific-language in global scope.

Isolating these changes currently requires creation of a new realm,
and creating new realms comes with the hazard of identity discontinuity.
For example, `array instanceof Array` is not as reliable as `Array.isArray`,
and the hazard is not limited to intrinsics that have anticipated this
problem with work-arounds like `Array.isArray` or thenable `Promise` adoption.

Some of these tools work around this problem by using the platforms existing
facility for creating a new global context, albeit an iframe or the Node.js `vm`
module.
Then, they are obliged to graft the intrinsics of one realm over the other,
which leaks for the cases of syntactically undeniable Realm-specific intrinsics
like the `AsyncFunction` constructor and prototype, and requires the
implementer to be vigilant to the extent that they graft every intrinsic from
one realm to another.
We have found such arrangements to be fragile and leaky. Also costly in memory efficiency and developer time.

This proposal provides an alternate solution: evaluate modules or scripts in a
separate global scope with shared intrinsics.

### Enforcing the principle of least authority

On the web, the same origin policy has become sufficiently effective at
preventing cross-site scripting attacks that attackers have been forced to
attack from within the same origin.
Conveniently for attackers, the richness of the JavaScript library ecosystem
has produced ample vectors to enter the same origin.
The vast bulk of a modern web application is its supply chain, including code
that will be eventually incorporated into the scripts that will run in the same
origin, but also the tools that generate those scripts, and the tools that
prepare the developer environment.

The same-origin-policy protects the rapidly deteriorating fiction that
web browsers mediate an interaction between just two parties: the service and
the user.
For modern applications, particularly platforms that mediate interactions among
many parties or simply have a deep supply chain, web application developers
need a mechanism to isolate third-party dependencies and minimize their access
to powerful objects like high resolution timers or network, compute, or storage
capability bearing interfaces.

Some hosts, including a community of embedded systems represented at [ECMA
TC53][tc53], do not have an origin on which to build a same-origin-policy, and
have elected to build their security model on isolated evaluators, through the
high-level Compartment interface.

### Isolating/encapsulating unreliable code

The way modern software is composed has already undermined the validity of the assumption that every author participating has their intentions well aligned for the benefit of the software working correctly. A whole new level of unreliability is now added with the popularity of _coding agents_ and _vibe coding_ where creating syntactically valid but effectively unpredictable JavaScript and integrating it into existing software to check whether it seems to implement the desired functionality is becoming a popular way of building software.

It is not an entirely new concern, as test runners have been concerned with isolating test cases to avoid them relying on global side-effects of other test cases. It is now a concern for a much wider audience with more at stake.

With `Global` constructor comes the ability to isolate fragments of the application in a way that unreliable code cannot rely on shared global state without the maintainer of the software knowing about it.
AI generated sources from independently working agents can come with colliding names for global variables to use and may need separate global scopes to collaborate or coexist. Similarly a misguided attempt at an inline polyfill by an AI or a package author could be prevented by freezing the parts of the new global in which the unreliable code subsequently runs.
Using a new global instead of a new Realm avoids the issues like identity discontinuity impeding the composition of software where function calls need to happen across the isolated and non-isolated code.


### Incremental or in-context execution

There are tools that currently use much more complex and costly mechanisms (similar to the ones described in [Domain Specific Languages](#domain-specific-languages) among other) to provide the ability to execute fragments of JavaScript code in a very specific context of the tool.

That includes REPLs, inline code execution results in editors (eg. [Quokka.js](https://quokkajs.com/)) and various use cases of IDEs in the browser.

Maintaining the global state between executions of user-provided code snippets would benefit from the ability to control scope 

## Intersection Semantics

TBD

## Design Questions



[proposal-source-phase-imports]: https://github.com/tc39/proposal-source-phase-imports
[proposal-esm-phase-imports]: https://github.com/tc39/proposal-esm-phase-imports
[proposal-compartments]: https://github.com/tc39/proposal-compartments
[proposal-import-hook]: https://github.com/endojs/proposal-import-hook
