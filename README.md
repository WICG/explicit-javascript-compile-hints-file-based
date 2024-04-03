# Explainer for Explicit JavaScript Compile Hints

This proposal is an early design sketch by the V8 team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- V8 team / Google

## Participate
- https://github.com/explainers-by-googlers/explicit-javascript-compile-hints-file-based/issues
- [Discussion forum] FIXME

## Table of contents [if the explainer is longer than one printed page] FIXME

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

### On JavaScript parsing and compilation

Knowing which JavaScript functions to parse & compile during the initial script compilation can speed up web page loading.

When processing a script we are loading from the network, we have a choice for each function; either we parse and compile it right away ("eagerly"), or we don't. If the function is later called and it was not compiled yet, we need to parse and compile it at that moment - and that always happens in the main thread.

If a JavaScript function ends up being called during page load, doing the parsing & compile work upfront is beneficial, because:
- During the initial parsing, we anyway need to do at least a lightweight parse to find the function end. In JavaScript, finding the function end requires parsing the full syntax (there are no shortcuts where we could count the curly braces - the grammar is too complex for them to work). Doing the lightweight parsing and after that the actual parsing is duplicate work.
- The initial parse might happen on a background thread instead of the main thread. When we need to compile the function because it's being called, it's too late to parallelize work.

Based on initial experiments, Google Docs report 5-7% improvement in their userland page load metrics with our prototype implementation, when selecting the core JS file for eager compilation.

Currently, Chromium and Firefox use the [PIFE heuristic](https://v8.dev/blog/preparser#pife) to direct which functions to compile. Safari doesn't follow the heuristic.

The PIFE heuristic has existed for a long time and is well known to web developers - some web pages (e.g., Facebook) use it or have used it for triggering eager compilation.

Using PIFEs for triggering eager compilation has downsides, though. Especially:
- using it forces using function expressions instead of function declarations. The semantics of function expressions mandate doing the assignment, so they're generally less performant than function declarations. For browsers which don't follow the PIFE hint there's no upside
- it cannot be applied to ES6 class methods

Thus, we'd like to specify a  more elegant way for triggering eager compilation.

## Goals

The goal of this proposal is to improve intial web page load speed and reduce interaction delays.

## Use cases

When users access web pages, they often experience delays as the browser parses and compiles necessary scripts. By utilizing explicit compile hints, developers can indicate which JavaScript files are crucial for rendering the initial page. This enables browsers to prioritize parsing and compiling the functions in these files ahead of time, potentially resulting in significantly faster page load times.

## Potential solution: Magic comment in JavaScript files

Explicit compile hints are triggered by inserting the following magic comment into JavaScript files:

```JavaScript
//# eagerCompilation=all
```

The magic comment is intended as a hint to the browser. It signals the functions in this JS file should be treated as "high priority", e.g., compile them upfront.

The magic comment doesn't change the semantics of the JavaScript file. The browser is allowed to ignore the hint.

The format for the magic comment is similar to the [Source Map magic comment](https://sourcemaps.info/spec.html).

The magic comment can appear anywhere in a JavaScript file, in any syntactic position where a comment can appear. The comment is intended to affect only JavaScript functions which occur after it.

Web developers should consider using explicit compile hints for files that contain important functions that are likely to be needed by the web page early on. For example, they might use explicit compile hints for a file that contains the main entry point for the application, or for a file that contains a critical library.

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

### Per-file or per-function?

If compile hints apply to the full file, web developers can manually select a "core file" of their web page for eager compilation. Selecting individual functions for eager compilation would need to be done in an automatic fashion - modern pages have tens of thousands of JavaScript functions, and improving the page load time requires selecting thoudands of them for eager compilation.

Selecting a whole file for eager compilation might overshoot: if some functions are not needed, compiling them takes CPU time, and storing the compiled code takes up memory (although, unused code can eventually be garbage collected).

In this proposal, we're proposing a magic comment for marking the whole file for eager compilation. We'd like to make it easy to extend the feature to be able to mark individual functions in the future.

## Considered alternatives

### Alternative: Top-level magic comment with per-function data as payload

Example:
```JavaScript
//# eagerCompilationData=<payload>
```

The payload would describe the function positions of the functions to be eagerly compiled. Designing a suitable payload format is non-trivial.

We'd also need to make sure that web development toolchains can generate the per-function data and incorporate it in an automatic fashion. For finding the functions, we suggest a profile-guided optimization (PGO) approach: first running the web page and logging which functions were called, and generating the per-function annotation based on the data. This area needs more experimentation.

We'd like to keep this alternative as a future extension, and propose the per-file magic comment solution first.

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

The top-level "use eager" directive is very similar to the top level magic comment we're proposing. The downside is that the per-function "use eager" directive would bloat the source code size. This is unoptimal; although compression would alleviate the transmitted file size overhead, parsing the magic comments still adds processing overhead which we think is unnecessary.

In addition, the per-function compile hint data should be automatically generated by the web development toolchains, so it's unnecessary to have it in the source code next to the function. We don't expect humans to look at or edit the data when working with the source code.

We'd like to propose a solution which we can later extend with per-function information in a lean way, and thus decided against this alternative.

### Alternative: per-function magic comment

Example:
```
/*eagerCompilation*/ function foo() { ... }
class C {
  /*eagerCompilation*/ m() { ... }
}
```

Similarily to per-functon "use eager" directives, this would bloat the source code size and introduce unnecessary parsing overhead.

### Alternative: compile hint data in the script tag

We could also add compile hint data (either the per-file version, or per-function data) into the script tag:

Example / per-file compile hint in script tag:
```
<script src="..." eager-compilation>
```

Example / per-function compile hint data in script tag:
```
<script src="..." eager-compilation-data="<payload>">
```

Example / per-function compile hint data in script tag:
```
<script src="..." eager-compilation-data-file="metadata-filepath">
```

Downsides:
- These alternatives separate the source code and the compile hints, making it harder to keep the compile hints up to date. They also require more modifications either to the HTML page or the parts of the pipeline generating or serving the HTML page to transmit the compile hint.
- Files loaded via other means than adding a script tag would require a separate solution.

### Alternative: compile hint data in an HTTP header

We could also transmit compile hint data in an HTTP header. This alternative also has the same downside than the previous solution; it would require modifying the web servers, not only the JavaScript source files.

## Risks and mitigations

- Web developers might overuse compile hints, making their web page slower. Browsers can try to mitigate that, e.g., by only increasing resource use (CPU for compiling the functions, memory for storing the compilation results) up to a quota. This risk also exists with using the PIFE heuristic for triggering eager compilation, and browsers following the PIFE heuristic don't try to mitigate it.

- Compile hints might get stale, if the web site is refactored. Likewise, this risk also exists with using the PIFE heuristic for triggering eager compilation.

- If multiple browsers implement this feature, it's possible that for a particular web site, only some of the browsers exhibit desirable behavior (performance improvements) and other browsers show a regression. There's no way for a web site to make the compile hints only apply to one browser.

- Removing this feature is easy; if a browser decides to no longer implement the feature, it will simply start ignoring the magic comment. All web sites will still function normally. If no browsers implement the feature, eventually web sites will drop the magic comment.

## Stakeholder feedback / opposition

Initially, the feature will probably be implemented only by Chromium, but it is designed to be general-purpose, so that other browsers can implement it in the future.

Concerns brought up by other browser implementors:
- Web developers might overuse this feature, selecting too many JavaScript files for eager compilation.
- The optimal set of functions to eager compile might be different for different browsers.
- Focus on warm loads: After the initial web page load, the browser might be in a better position to decide which functions should be eagerly compiled, than the web developers.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Shu-Yu Guo, Toon Verwaest and Leszek Swirski from V8 (Google)
- Philip Weiss and Adam Giacobbe from Workspace (Google)
