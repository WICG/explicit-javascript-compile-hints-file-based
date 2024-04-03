# Explainer for Explicit JavaScript Compile Hints

This proposal is an early design sketch by the V8 team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- V8 team / Google

## Participate
- https://github.com/explainers-by-googlers/explicit-javascript-compile-hints-file-based/issues
- [Discussion forum] FIXME

## Table of Contents [if the explainer is longer than one printed page] FIXME

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1](#use-case-1)
  - [Use case 2](#use-case-2)
- [[Potential Solution]](#potential-solution)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
- [Detailed design discussion](#detailed-design-discussion)
  - [[Tricky design choice #1]](#tricky-design-choice-1)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [[Alternative 1]](#alternative-1)
  - [[Alternative 2]](#alternative-2)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This proposal introduces a new magic comment that signals to browsers that the functions in a JavaScript file are likely to be needed by the web page. This allows the browser to parse and compile the correct set of JavaScript functions, which can improve page load times.

## Goals

The goal of this proposal is to improve intial web page load speed and reduce interaction delays.

## Non-goals

[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]

## Use cases

When users access web pages, they often experience delays as the browser parses and compiles necessary scripts. By utilizing explicit compile hints, developers can indicate which JavaScript files are crucial for rendering the initial page. This enables browsers to prioritize parsing and compiling the functions in these files ahead of time, potentially resulting in significantly faster page load times.


## Potential Solution: Magic comment in JavaScript files

Explicit compile hints are triggered by inserting the following magic comment into JavaScript files:

```JavaScript
//# eagerCompilation=all
```

The magic comment is intended as a hint to the browser. It signals the functions in this JS file should be treated as "high priority", e.g., compile them upfront.

The magic comment doesn't change the semantics of the JavaScript file. The browser is allowed to ignore the hint.

The format for the magic comment is similar to the [Source Map magic comment](https://sourcemaps.info/spec.html).

The magic comment can appear anywhere in a JavaScript file, in any syntactic position where a comment can appear. The comment is intended to affect only JavaScript functions which occur after it.

Web developers should consider using explicit compile hints for files that contain important functions that are likely to be needed by the web page early on. For example, they might use explicit compile hints for a file that contains the main entry point for your application, or for a file that contains a critical library.

### Possible browser implementations

Different browsers might handle the magic comment differently, based on what makes the most sense given their design and the available resources.

The following examples describe possible and valid actions when encountering a file with the magic comment:

Implementation 1: Background-parse the JavaScript file while downloading it. Kick off background compilation tasks for all the functions (possibly up to a quota).

Implementation 2: Parse the JavaScript file and compile all functions eagerly on the main thread (possibly up to a quota).

Implementation 3: Ignore the hint when initially compiling the file. When a code cache for the file is created (e.g., when a user visits the same web page often enough that cache creation is deemed useful), create a code cache containing all the functions in the file.

Implementation 4: Like Implementation 1/2 but compile the functions with a higher tier compiler right away.

Implementation 5: Ignore the hint.

Chromium is currently experimenting with the feature (options 1 and 3). 

### How this solution would solve the use case

The solution of explicit compile hints addresses the identified use cases by providing developers with a mechanism to prioritize the parsing and compilation of critical JavaScript functions, thereby expediting the initial page load. This results in smoother browsing experiences, especially on mobile devices with limited resources.

## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

### Alternative: Top-level "use eager" directive and per-function "use eager" directive

Example / top level "use eager" directive:
```
"use eager"; // All functions in this file will be eager

function foo() { ... }
function bar() { ... }
class C {
  m() { ... }
}
```

Example / per function "use eager" directive:

```
function foo() { "use eager"; ... }
class C {
  m() { "use eager"; ... }
}
```

The top-level "use eager" directive is very similar to the top level magic comment we're proposing. The downside is that the per-function "use eager" directive would bloat the source code size. This is unoptimal; although compression would alleviate the tramitted file size overhead, parsing the magic comments still adds overhead which we think is unnecessary.

We'd like to propose a solution which we can later extend with per-function information in a lean way, and thus decided against this alternative.

### Alternative: per-function magic comment

Example:
```
/*compile-hint*/ function foo() { ... }
class C {
  /*compile-hint*/ m() { ... }
}
```

Similarily to per-functon "use eager" directives, this would bloat the source code size and introduce unnecessary parsing overhead.

### Alternative: compile hint data in the script tag

We could also add compile hint data (either the per-file version, or per-function data) into the script tag:

Example / per-file compile hint in script tag:
```
<script src="..." compile-hint>
```

Example / per-function compile hint data in script tag:
```
<script src="..." compile-hint-data="...">
```

Downsides:
- This alternative separates the source code and the compile hint, making it harder to keep the compile hint up to date, and requiring more modifications /either to the HTML page or the parts of the pipeline generating or serving the HTML page) to transmit the compile hint.
- Files loaded via other means than adding a script tag would require a separate solution.

### Alternative: compile hint data in a HTTP header

We could also transmit compile hint data in a HTTP header. This alternative also has the same downside than the previous solution; it would require more extensive modifications to the web server, not only the JavaScript source file.


## Stakeholder Feedback / Opposition

Initially, the feature will probably be implemented only by Chromium, but it is designed to be general-purpose, so that other browsers can implement it in the future.

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
