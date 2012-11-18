---
layout: post
title: Fixing CoffeeScript
tags:
- javascript
- coffeescript
- math
status: draft
type: post
published: true
meta:
  _edit_last: "1"
---

Ashkenas quote from language panel

it appears to be popular for it's terse syntax

has been criticised as _too_ flexible in some cases

## CoffeeScript Confusion

```coffeescript
deSomething = () -> true

# and

doSomething () ->  false

# vs

doSomething() ->  false
```

```javascript
var doSomething = function(){ return true };

// and

doSomething(function(){ return false; });

// vs

doSomething()(function(){ return false; });

// !!! => true(function(){ return false; });
```

The goal then is to apply some formalism to CoffeeScript in the hopes of finding a way to prevent these types of issues for language designers. With that in mind, a good place to would be the way in which the language evaluates to produce expressions like the second one that "go wrong" in some fashion.

## Operational Semantics

Operational Semantics is one way [!!] to formalise the meaning of a programming language constructs by defining the way terms evaluate or reduce to values. We'll build a basic understanding of how it works by borrowing examples from Pierce's book Types and Programming Lanaguages. This includes a basic language dealing with boolean values to start:

boolean grammar

The grammar definition is made of up of two "meta variables" `t` and `v`. The first, `t`, represents all of the possible ways to construct terms in the language. Make sure to note that the term `if t then t else` has sub terms represented with the meta variable `t`. This captures the ability to use `true`, `false`, or another `if t then t else t` in place of each sub term `t`. The second meta variable `v` represents the set of terms that are acceptable as the final result of evaluation. That means if evaluation "gets stuck" at an `if t then t else t` term something has gone wrong. Some sample terms:

```haskell
-- simple values
true
false

-- a simple compound term
if true then false else true

-- a more complex compund term
if false then false else if true then false else true
```

With the building blocks of the example language in place the next step is to establish a set of rules that will define the way terms are evaluated. In spite of the fact that the language defined here is exceptionally simple there is some subtlety that might not otherwise be obvious without a complete definition.

evaluation rules

These equations are collectively referred to as the _evaluation relation_ and individually as _inference rules_. Each of them plays an important role in the _evaluation strategy_ [!!] of the example language. Equations 1 and 2 are fairly simple in that they represent the expected evaluation results for the different guard values in an `if t then t else t` term. With `true` you get the first subterm and with `false` you get the second subterm. Equation 3 is more interesting in its construction and how captures an important subtlety of the evaluation strategy.

equation 3

There are two parts to this rule. Above the line is the _premise_ and below is the _conclusion_. The premise establishes a requirement or precondition that's required for the application of the conclusion to a given term. In this case the premise says that if the first term `t` can be evaluated then the parent term, `if t then t else t` should evaluate to `if t' then t else t`. The importance of this rule is

!! deeper discusion of grammar required. eg, why the ::== and what each line means. important for eval/type rule understanding !!

show basic type information applied, including type rules

## CoffeeScript Grammar

You might consider this problem with the grammar/parser and not the evaluation result. That is, maybe it shouldn't allow both lambda forms as method/function arguments, but because both forms are valid in the current parser implementation it's only an issue when evaluation takes place. Moreover it's only an *obvious* issue when the return type of `doSomething` doesn't allow for invocation. In an ideal world the author would arrive at these conclusions early during the process of creating the language. Creating a subset of CoffeeScript including the grammar, evaluation rules, and some simple types should help in clarifying the issue and will hopefully present a way to avoid it all together.

In the interest of keeping this post a semi-digestable length the subset will be the minimum necessary to reproduce the aformentioned ambiguity. In the original example, the overhead of assignment and identifiers can be avoided by using lamda expressions directly:

```coffeescript
(-> true) () ->  false
# => true

# vs

(-> true)() ->  false
# !!! => true(function(){ return false; });
```

Which of course translates into JavaScript similarly to the original example:

```javascript
(function(){ return true; })(function(){ return false; });

// vs

(function(){ return true; })()(function(){ return false; });

// !!! => true(function(){ return false; });
```

Here the use of atomic boolean values alleviates the need for arguments in the lambda syntax. Again, simplicity in reproducing the issue is preferred for the sake of brevity. Having identified the subset of CoffeeScript necessary for reproducing the original issue, we can begin with a precise definiton of terms in the form of a language grammar.

<div class="center">
  <img src="/assets/images/diagrams/cs-grammar.png"></img>
</div>

If you've never seen a language grammar before it might appear a bit confusing. On the left hand side of each `::=` assignment is a meta variable that can be used in other parts of the grammar. In the case of `\t` it was easier to create a meta term in place of repeating each possible lambda form in `t`. `v` on the other hand has special meaning. It represents the set of acceptable final results of evaluating expressions which will be covered shortly. Finally `t` is total set of terms and possible compound terms. Notable among them is the invocation and application of lambda terms (`\t`).

The monospace font portions of terms like `true`, `false`, `->`, `() ->`, and `()` are parts of the language that cannot be represented by meta variables and will sometimes be replaced when forms are expressed using the grammar. Some translated examples should help to clarify:

```coffeescript
true
```

<div class="center">
  <img src="/assets/images/diagrams/cs-grammar-true.png"></img>
</div>

```coffeescript
(-> false)()
```

<div class="center">
  <img src="/assets/images/diagrams/cs-grammar-lambda-invocation.png"></img>
</div>


```coffeescript
(() -> true) true
```

<div class="center">
  <img src="/assets/images/diagrams/cs-grammar-lambda-application.png"></img>
</div>

```coffeescript
(-> (() -> true))() (-> true)
```

<div class="center">
  <img src="/assets/images/diagrams/cs-grammar-lambda-complex.png"></img>
</div>

The most important part to note that may not have been clear from the grammar definition is that the grammar representation of a lambda term captures the subterm be it `true`, `false` or another lambda. Equations 2, 3, and 4 are examples.

As a result of including that subterm information it's reasonable to ask whether there's value in providing a grammar that, in translation, looks a lot like it's own language. First, maps the two different lambda forms to one form in the grammar, which makes talking about evaluation and types easier. Second, differentiating values (`true`, `false`, and `-> t`) from other terms by calling them values is important for knowing when evaluation has finished or stalled ("gotten stuck").

## Inference Rules

With the grammar in place the next step is to define both the inference rules and strategy so that our terms can perform some computation. To keep things simple the language subset will use the call-by-value, left to right strategy [!!] employed by CoffeeScript, JavaScript and numerous other languages (C, Java, Python etc). There are three inference rules.

<div class="center">
  <img src="/assets/images/diagrams/cs-inference-rules.png"></img>
</div>

The impact of the evaluation strategy (call-by-value l-to-r) for this language is that the arguments applied to lambda terms must be as "evaluated" as possible, before application can take place. For example `(-> true) (-> false)()` would first evaluate to `(-> true) false` as a result of the `()` operator.

<div class="center">
  <img src="/assets/images/diagrams/cs-inference-rules-application-argument.png"></img>
</div>

Equation 7 stipulates in the premise that if the term that a lambda would otherwise be applied to can take a step of evaluation it should. Taken together these three rules define how the CoffeeScript subset performs computation. Unfortunately it's not obvious thusfar that there might be an issue.

So far defining lambdas with a `-> t` is appealing, and preparing for arguments with a null tuple preceding the arrow makes sense. The application of type rules to our language is the last step to having a complete picture of how it works.

## Type Rules

Type rules are similar in construction to evaluation rules consisting of a premise and conclusion. As with evaluation rules the premise establishes the preconditions for the conclusion.

<div class="center">
  <img src="/assets/images/diagrams/cs-type-rules.png"></img>
</div>

The type rules for this subset of CoffeeScript are fairly intuitive. For the type rules without a premise like `true : Bool` and `false : Bool`, we take them to be true out of hand. That is, the terms `true` and `false` both have the type `Bool`. The others are more complicated.

<div class="center">
  <img src="/assets/images/diagrams/cs-type-rules-lambda-term.png"></img>
</div>

Equation 9 illustrates how to determine the type of a lambda term like `(-> true)`. The premise above the line states that if the subterm `t` has the type `T`, then the conclusion on the bottom, `\t`, has the type `\top \to T`. Translated to our mini example `(-> true)`, because the subterm `true` has the type `Bool`, the lambda term has the type `\top \to Bool`. One extremely important note: because neither of the lambda expressions make use of arguments, Top (`Top`) is used to represent the acceptance of any type. So when `t` in `\t` has the type `T` it means that `\t` will, under invocation or application, result in an expresion of typ `T` regardless of what it's applied to or invoked with. As an example:

<div class="center">
  <img src="/assets/images/diagrams/cs-type-rules-lambda-invocation.png"></img>
</div>

Equation 10 shows how to determine the type of lambda invocation like `(-> true)()`. The premise states if the lambda term has the type `\top \to T` the term `\t()` has the type `T`. This fits with the CoffeeScript `(-> true)()` which obviously evaluates to `true` which has the type `Bool`. Again, `\top` is used to denote the fact that the argument type is unimportant and in this case non existent.

<div class="center">
  <img src="/assets/images/diagrams/cs-type-rules-lambda-application.png"></img>
</div>

Equation 11 is the type rule for lambda applications such as `(-> true) false`. The premise says that if the lambda term `\t` on the left has the type `\top \to T_1` and the term on the right as the type `T_2` the conclusion is that the application of the lambda term will have the type `T_1`. Again, the type of an application, like invocation, is only concerned with the type of the *first* lambda's subterm `t`.

## Type Rule Stacking

This notation makes it easy to establish the type of a term by stacking the type rules on one another. Taking a very timple example, some diagrams will illustrate how this works:

```coffeescript
(-> true)
```

<div class="center">
  <img src="/assets/images/diagrams/cs-type-derivation-simple.png"></img>
</div>

Equation 13 is the stack, a derivation tree. It explains how the reader can derive the type at the bottom from it's subterms. Starting with equation 14, the subterm `true` is typed by the type rule *true*. That can then be "stacked" by using it to replace the premise of type rule *lambda* in equation 15. The type derivation expands from the subterm to establish each subsequent parent term's type. Make sure to note that type derivations are never really accompanied by the full form of the type rules in equations (just the labels like *lambda* and *true*) but it's helpful here for illustration here. Taking the more complex term from earlier it's clear why they are ommited:

```coffeescript
(-> true) (-> (-> false))()
```

<div class="center">
  <img src="/assets/images/diagrams/cs-type-derivation-complex.png"></img>
</div>

Since there are two subterms involved in application, each branch extends upward until it reaches the atomic value types `true : Bool` and `false : Bool`. The subterm on the left is identical to equation 13. On the right, the nested lambdas and invocation make for a taller stack of type rules to reach the atomic `false`.

## Not Quite There

At this point the CoffeeScript subset has enough definition to describe the original issue in terms of evaluation or typing. That is, using the inference rules it's clear that the evaluation of `(-> true)() -> false` "gets stuck" from the following derivation tree:

<div class="center">
  <img src="/assets/images/diagrams/cs-eval-derivation-original-issue.png"></img>
</div>

After the invocation of the left lambda term the evaluation gets stuck because the right lambda term can't be evaluated any further and no rule exists to handle the application of the term `true` to `(-> false)`. Additionally, a derivation tree based on the typing rules highlight that the term is untypable:

<div class="center">
  <img src="/assets/images/diagrams/cs-type-derivation-original-issue.png"></img>
</div>

Once the derivation tree reaches the outermost term it breaks because once again we have no type definition for the application of `true` to a lambda term like `(-> false)`.

So what's been gained thusfar by applying the formalism of operational semantics and type derivations? It's clear that types, inferred or otherwise, would prevent at least the original case `(-> true)() -> false` as you can see from the type derivation. Unfortunately type derivation or type inference would only alert the end user and not an author in the middle of creating a language and then only if the result of the leftmost lambda term is not a lambda term. That is, if the leftmost lambda term evaluates to a lambda term the the whole thing is well typed. This means type information/derivations alone can't help identify this syntactic ambiguity.

## Detecting Ambiguity

Still, there is clearly an important difference here. Even if the two terms will *not* result in a type error under evaluation (eg, see the snippet below), they have different type derivations. The fact that the syntactic difference is seemingly very small in spite of the fact that the type derivation is different may suggest a way to detect ambiguities.

```coffeescript
# fails
(-> true)() -> false

# works fine
(-> (-> true))() -> false

# also works fine
(-> (-> true)) () -> false
```

First, some formulation of syntactic difference is necessary so that it can be measured. Since this particular problem is small the definition can start out simple. Taking the Levenshtein Distance as a function of string size should work for a start. For the syntax/string representation of two terms `a` and `b`:

insert image

*For two terms with inconsistent types, we say they are more or less ambigous based on the calculation Levenshtein Distance/max(Length<sub>1</sub>, Length<sub>2</sub>)*.

# footnotes

!! Tested at http://coffeescript.org/.
!! Technically JavaScript uses a strategy known as Call by Sharing, which differs from Call by Value in how deals with objects. More information at http://dmitrysoshnikov.com/ecmascript/chapter-8-evaluation-strategy/ courtesy of [@raganwald](https://twitter.com/raganwald).