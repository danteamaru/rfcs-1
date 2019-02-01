- Start Date: 2019-01-29
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Handlebars Strict Mode

## Summary

In this RFC, we propose a set of changes to Ember's variant of Handlebars that
are aimed at improving clarity and simplifying the language. Together, these
changes are bundled into a "strict mode" that developers can opt-into. In
contrast, the non-strict mode (i.e. what developers are using today) will be
referred to as "sloppy mode" in this RFC.

## Motivation

* easier to learn
* easier to understand
* easier to implement
* better foundation for tooling

1. No implicit globals
2. No implicit `this` fallback
3. No dynamic resolution
4. No evals (no partials)
5. HTML attributes by default

### 1. No implicit globals

Today, Ember implicitly introduce a set of implicit globals into a template's
scope, such as built-in helpers, components, modifiers. Apps and addons also
have the ability to introduce additional implicit globals by placing files into
the `app` folder or broccoli tree. It is also possible to further influence
this behavior by using the intimate resolver API (such as the alternative
"pods" layout).

This adds a fair amount of dynamism, ambiguity and confusion when reading
templates. When an identifier is encountered, it's not always clear where this
value comes from or what kind of value it may be. This problem is especially
acute for the "ambigious content" position, i.e. `<div>{{foo-bar}}</div>`,
which could be a local variable `foo-bar`, a global component or a helper named
`foo-bar` provided by the app or an addon, or `{{this.foo-bar}}` (see the next
section). This problem is even worse when there is a custom resolver involved,
as the resolver may return a component or helper not found in the "expected"
location at runtime.

Not only is this confusing for the human reader, it also makes it difficult for
the Glimmer VM implementation as well as other ecosystem tooling. For example,
if the developer made a typo, `{{food-bar}}`, it would be impossible to issue
a static error (build time error, inline error in IDEs) because the value may
be resolvable at runtime. It is also difficult, and in some cases impossible,
to implement IDE features such as "Jump to definition" without running code.

[RFC #432](https://github.com/emberjs/rfcs/blob/contextual-helpers/text/0432-contextual-helpers.md#relationship-with-globals)
described some additional issues with the current implicit globals semantics.

We propose to remove support for implicit globals in strict mode. All values
must be explicitly brought into scope, either through [template imports](todo://template-import-rfc),
variables in the [JavaScript scope](todo://gbs-rfc) or from block params. Any
other undefined references will be treated as syntax errors.

### 2. No implicit `this` fallback

Today, when Ember sees a path like `{{foo}}` in the template, after exhausting
the possibilities of implicit globals, it falls back to `{{this.foo}}`. This
adds to the same confusion outlined above. More details about the motivation
can be found in the accepted [RFC #308](https://github.com/emberjs/rfcs/pull/308).

We propose to remove support for implicit `this` fallback in strict mode. The
explicit form `{{this.foo}}` must be used to refer to instance state, otherwise
it will trigger the same errors mentioned in the previous section.

It is worth mentioning that [RFC #432](https://github.com/emberjs/rfcs/pull/432)
laid out a transition path towards a world where "everything is a value".
Between this and the previous restriction, we essentially have completed that
transition. To recap, here is a list of the
[outstanding issues](https://github.com/emberjs/rfcs/blob/contextual-helpers/text/0432-contextual-helpers.md#relationship-with-globals):

1. Not possible to reference globals outside of invocation positions
2. Invocation of global helpers in angle bracket named arguments positions
3. Naming collisions between global components, helpers and element modifiers

All of these problems are all related to implicit globals and/or implicit
`this` fallback. Since neither of these features are supported in strict mode,
they are no longer a concern for us.

### 3. No dynamic resolution

Today, Ember supports passing strings to the `component` helper (as well as the
`helper` and `modifier` helpers proposed in [RFC #432](https://github.com/emberjs/rfcs/pull/432)).
This can either be passed as a literal `{{component "foo-bar"}}` or passed as a
runtime value `{{component this.someString}}`. In either case, Ember will
attempt to _resolve_ the string as a component.

It shares some of the same problems with implicit globals (where did this come
from?), but the dynamic form makes the problem more acute, as it is difficult
or impossible to tell which components a given template is dependent on. As
usual, if it is difficult for the human reader, the same is true for tools as
well. Specifically, this is hostile to "tree shaking" and other forms of static
(build time) dependency-graph analysis, since the dynamic form of the component
helper can invoke _any_ component available to the app.

We propose to remove support for these forms of dynamic resolutions in strict
mode. Specifically, passing a string to the `component` helper (as well as the
`helper` and `modifier` helpers), whether as a literal or at runtime, will
result in an error.

In practice, it is almost always the case that these dynamic resolutions are
switching between a small and bounded number of known components. For this
purpose, they can be replaced by patterns similar to this:

```
--- js ---
import { component, get } from '@ember/template/helpers'; // TODO: imports RFC
import { First, Second, Third } form './contextual-components';

const COMPONENTS = {
  first: First,
  second: Second,
  third: Third
};

--- hbs ---
{{yield (component (get COMPONENTS this.selected) ...)}}
```

This will both make it clear to the human reader and enables tools to perform
optimizations, such as tree-shaking, by following the explict dependency graph.

### 4. No evals (no partials)

Ember current supports the `partial` feature. It takes a template name, which
can either be passed as a literal `{{partial "foo-bar"}}` or passed as a
runtime value `{{partial this.someString}}`. In either case, Ember will resolve
the template with the given name (with a prefix dash, like `-foo-bar.hbs`) and
render its content _as if_ they were copy and pasted into the same position.

In either case, the rendered partials have full access to anything that is "in
scope" in the original template. This includes any local variables, instance
variables (via implicit `this` fallback or explicitly), implicit globals, named
arguments, blocks, etc.

This feature has all of the same problems as above, but worse. In addition to
the usual sources (globals, `this` fallback etc), each variable found in a
partial template could also be coming from the "outer scope" from the caller
template. Conversely, on the caller side, "unused" variables may not be safe
to refactor away, because they may be consumed in a nested partial template.

Not only do these make it difficult for humans to follow, the same is true for
tools as well. For example, linters cannot provide accurate undefined/unused
variables warning. Whenever the Glimmer VM encounter partials, it has to emit
a large amount of extra metadata just so they can be "wired up" correctly at
runtime.

We propose to remove support for partials completely in strict mode. Invoking
the `{{partial ...}}` keyword in strict mode will be a static (build time)
error.

The use case of extracting pieces of a template into smaller chunks can be
replaced by [template-only components](https://github.com/emberjs/rfcs/pull/278).
While this requires any variables to be passed explicitly as arguments, it also
removes the ambiguity and confusions.

It should also be mentioned that the `{{debugger}}` keyword also falls into the
category of "eval" in today's implementation, since it will be able to access
any variable available to the current scope, including `this` fallback and when
nested inside a partial template. However, with the other changes proposed in
this RFC, we will be able to statically determine which variables the debugger
will have access to. Therefore we would still be able to support debugger usage
in strict mode without a very high performance penalty.

### 5. HTML attributes by default

...?

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
