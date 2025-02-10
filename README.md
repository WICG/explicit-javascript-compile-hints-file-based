# Explainer for Explicit JavaScript Compile Hints

This proposal is an early design sketch by the V8 team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- V8 team / Google

## Participate
- https://github.com/explainers-by-googlers/explicit-javascript-compile-hints-file-based/issues

## Spec draft

[Spec draft](https://wicg.github.io/explicit-javascript-compile-hints-file-based/)

## Table of contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
  - [On JavaScript parsing and compilation](#on-javascript-parsing-and-compilation)
  - [The PIFE heuristic](#the-pife-heuristic)
- [Goals](#goals)
- [Use cases](#use-cases)
  - [Use case 1: loading websites](#use-case-1-loading-websites)
  - [Use case 2: interaction](#use-case-2-interaction)
- [Potential solution: Magic comment in JavaScript files](#potential-solution-magic-comment-in-javascript-files)
  - [Possible browser implementations](#possible-browser-implementations)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Per-file or per-function?](#per-file-or-per-function)
- [Alternatives considered](#alternatives-considered)
  - [Alternative: Top-level magic comment with per-function data as payload](#alternative-top-level-magic-comment-with-per-function-data-as-payload)
  - [Alternative: Top-level "use eager" directive and per-function "use eager" directive](#alternative-top-level-use-eager-directive-and-per-function-use-eager-directive)
  - [Alternative: per-function magic comment](#alternative-per-function-magic-comment)
  - [Alternative: compile hint data in the script tag](#alternative-compile-hint-data-in-the-script-tag)
  - [Alternative: compile hint data in an HTTP header](#alternative-compile-hint-data-in-an-http-header)
  - [Alternative: do nothing / recommend using the PIFE heuristic for triggering eager compilation](#alternative-do-nothing--recommend-using-the-pife-heuristic-for-triggering-eager-compilation)
  - [Alternative: do nothing / no solution for web developers to control eager compilation](#alternative-do-nothing--no-solution-for-web-developers-to-control-eager-compilation)
- [Risks and mitigations](#risks-and-mitigations)
- [Stakeholder feedback / opposition](#stakeholder-feedback--opposition)
- [FAQ](#faq)
  - [Q: Why are you not pursuing standardizing the feature via TC39?](#q-why-are-you-not-pursuing-standardizing-the-feature-via-tc39)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This proposal introduces a new magic comment that signals to browsers that the functions in a JavaScript file are likely to be needed by the website. This allows the browser to parse and compile them eagerly, which can improve page load times.

In this example, the magic comment is used for triggering the eager compilation of the two JavaScript functions in the file:
```JavaScript
//# eagerCompilation=all

function foo() { ... } // will now be eagerly parsed and compiled
function bar() { ... } // will now be eagerly parsed and compiled
```

### On JavaScript parsing and compilation

Knowing which JavaScript functions to parse and compile during the initial script compilation can speed up web page loading.

When processing a script we are loading from the network, we have a choice for each function; either we parse and compile it right away ("eagerly"), or we don't. If the function is later called and it was not compiled yet, we need to parse and compile it at that point. The main thread is waiting for the function to be called and cannot proceed until the function is compiled.

If a JavaScript function ends up being called during page load, parsing and compiling eagerly is beneficial, because:
- During the initial parsing, we anyway need to do at least a lightweight parse to find the function end. In JavaScript, finding the function end requires parsing the full syntax (there are no shortcuts where we could count the curly braces - the grammar is too complex for them to work). Doing the lightweight parsing and after that the actual parsing is duplicate work.
- The eager parsing might happen on a background thread instead of the main thread. When we need to compile the function because it's being called, it's too late to parallelize work.

Based on initial experiments, Google Workspace products (such as Google Docs) report 5-7% improvement in their userland page load metrics with our prototype implementation, when selecting the core JS file for eager compilation.

### The PIFE heuristic

Currently, Chromium and Firefox use the [PIFE heuristic](https://v8.dev/blog/preparser#pife) to direct which functions to compile. Safari doesn't follow the heuristic.

The PIFE heuristic has existed for a long time and is well known to web developers - some websites (e.g., Facebook) use it or have used it for triggering eager compilation.

Using PIFEs for triggering eager compilation has downsides, though. Especially:
- using it forces using function expressions instead of function declarations. The semantics of function expressions mandate doing the assignment, so they're generally less performant than function declarations. For browsers which don't follow the PIFE hint there's no upside.
- it cannot be applied to ES6 class methods

Thus, we'd like to specify a  more elegant way for triggering eager compilation.

## Goals

The goal of this proposal is to improve intial web page load speed and reduce interaction delays by allowing web developers to control which JavaScript functions are parsed and compiled eagerly.

## Use cases

### Use case 1: loading websites

When users load websites, they often encounter delays as the browser parses and compiles necessary scripts. A part of the delays are due to "lazy function compilation", where we haven't compiled a JavaScript function before it is called, and we need to compile it now.

### Use case 2: interaction

When users interact with websites, there are delays in how quickly the website responds to the interaction. Likewise, a part of these delays are due to "lazy function compilation".

## Potential solution: Magic comment in JavaScript files

We propse adding the following magic comment to trigger eager compilation of all functions in the JavaScript file:

```JavaScript
//# eagerCompilation=all
```

The magic comment is intended as a hint to the browser. It signals the functions in this JS file should be treated as "high priority" - for example, compile them immediately when processing the script, as opposed to when a function is called.

The magic comment doesn't change the semantics of the JavaScript file. The browser is allowed to ignore the hint.

The format for the magic comment is similar to the [Source Map magic comment](https://sourcemaps.info/spec.html).

The magic comment should be at the top of the file, preceeded only by other single-line or multiline comments.

Web developers should consider using explicit compile hints for files that contain important functions which are likely to be needed by the website early on. For example, they might use explicit compile hints for a file that contains the main entry point for the application, or for a file that contains a critical library.

### Possible browser implementations

Different browsers may handle the magic comment differently, based on their design and available resources.

The following examples describe possible and valid actions when encountering a file with the magic comment:

Implementation 1: Background-parse the JavaScript file while downloading it. Mark all functions for compilation, potentially by separate background compilation tasks (possibly up to a quota).

Implementation 2: Parse the JavaScript file and compile all functions eagerly on the main thread (possibly up to a quota).

Implementation 3: Ignore the hint when initially compiling the file. When a code cache for the file is created (e.g., when a user visits the same website often enough that cache creation is deemed useful), create a code cache containing all the functions in the file.

Implementation 4: Like Implementation 1/2 but compile the functions with a higher tier compiler right away.

Implementation 5: Ignore the hint.

Chromium is currently experimenting with the feature (options 1 and 3). 

### How this solution would solve the use cases

The solution of explicit compile hints addresses the identified use cases by providing developers with a mechanism to prioritize the parsing and compilation of critical JavaScript functions, thereby speeding up the initial page load. This results in smoother browsing experiences, especially on mobile devices with limited resources.

## Detailed design discussion

### Per-file or per-function?

If compile hints apply to the full file, web developers can manually select a "core file" of their website for eager compilation. Selecting individual functions for eager compilation would need to be done in an automatic fashion - modern websites have tens of thousands of JavaScript functions, and improving the page load time requires selecting thoudands of them for eager compilation.

Selecting a whole file for eager compilation might overshoot: if some functions are not needed, compiling them takes CPU time, and storing the compiled code takes up memory (although, unused code may eventually be garbage collected).

In this proposal, we're proposing a magic comment for marking the whole file for eager compilation. We'd like to make it easy to extend the feature to be able to mark individual functions in the future.

## Alternatives considered

### Alternative: Top-level magic comment with per-function data as payload

Example:
```JavaScript
//# eagerCompilationData=<payload>
```

The payload would describe the function positions of the functions to be eagerly compiled. Designing a suitable payload format is non-trivial.

We'd also need to make sure that web development toolchains can generate the per-function data and incorporate it in an automatic fashion. For finding the functions, we suggest a profile-guided optimization (PGO) approach: launching a web server, loading the website and logging which functions were called, and generating the per-function annotation based on the data. This area needs more experimentation.

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

We could also transmit compile hint data in an HTTP header. This alternative also has the same downside as the previous solution; it would require modifying the web servers, not only the JavaScript source files.

### Alternative: do nothing / recommend using the PIFE heuristic for triggering eager compilation

The PIFE heuristic forces using assignment expressions instead of function declarations and cannot be added to ES6 class methods.

### Alternative: do nothing / no solution for web developers to control eager compilation

Even if we don't provide a way for web developers to control eager compilation, we can still observe which JavaScript functions were called and use that information for speeding up subsequent page loads of the same version of the website. However, this does not work optimally for the following types of websites:
- Websites which update frequently  (e.g., daily); when a new version ships, the browser hasn't seen the new scripts yet and cannot make an optimal compilation decision for it.
- Websites which generate the JavaScript files dynamically (e.g., decide which experiment groups are active for each page load individually). Those websites tend to create unique JavaScript files for every page load, and if compile hint information is not part of those files, we won't be able to infer the optimal compilation decision from outside sources.

## Risks and mitigations

- Web developers might overuse compile hints, potentially slowing down their web sites. Browsers can mitigate this risk by limiting resource usage, such as CPU and memory, up to a certain quota. This risk also exists with using the PIFE heuristic for triggering eager compilation, and browsers following the PIFE heuristic don't try to mitigate it.

- Compile hints might get stale, if the web site is refactored. Likewise, this risk also exists with using the PIFE heuristic for triggering eager compilation.

- If multiple browsers implement this feature, it's possible that for a particular web site, only some of the browsers exhibit desirable behavior (performance improvements) and other browsers show a regression. There's no way for a web site to make the compile hints only apply to one browser.

- Removing this feature is easy; if a browser decides to no longer implement the feature, it will simply start ignoring the magic comment. All web site will still function normally. If no browsers implement the feature, eventually web sites will drop the magic comment.

## Stakeholder feedback / opposition

Initially, the feature will probably be implemented only by Chromium, but it is designed to be general-purpose, so that other browsers can implement it in the future.

Concerns brought up by other browser implementors:
- Web developers might overuse this feature, selecting too many JavaScript files (or functions, in the future per-function version) for eager compilation.
- The optimal set of files / functions to eager compile might be different for different browsers.
- Compile hints are only relevant for the "cold load" (the initial, non-cached load of a website). After the initial web page load, the browser might be in a better position to decide which functions should be eagerly compiled than the web developers.

## FAQ

### Q: Why are you not pursuing standardizing the feature via TC39?

This feature only impacts performance. It doesn't change observable behavior, and thus it avoids the kind of interoperability challenges that TC39 is designed to address. Currently, other browser implementors have not show interest in implementing performance optimizations based on this feature.

Incubating in WICG allows us to gather feedback from stakeholders and iterate on the spec in a faster and leaner way. We'll also keep the format as generic as possible - to make it as easy as possible for other browsers to implement performance optimisations based on this feature later - and incorporate feedback from other browsers into the spec.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Shu-Yu Guo, Toon Verwaest and Leszek Swirski from V8 (Google)
- Philip Weiss, Adam Giacobbe and Quade Jones from Workspace (Google)
- Noam Rosenthal and other spec experts (Google)
