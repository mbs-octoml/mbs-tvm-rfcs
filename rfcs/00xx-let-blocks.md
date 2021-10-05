- Feature Name: let-blocks
- Start Date: 2021-10-05
- RFC PR: [apache/tvm-rfcs#0000](https://github.com/apache/tvm-rfcs/pull/0000)
- GitHub Issue: [apache/tvm#0000](https://github.com/apache/tvm/issues/0000)

# Summary
[summary]: #summary

Currently Relay supports:
 - Implicitly shared AST nodes:
   ```
     %0 = add(%y, %z);
     subtract(%0, %0)
   ```
   These are typically constructed directly using the Python AST bindings:
   ```
     a = relay.add(y, z)
     relay.subtract(a, a)
   ```
   This sharing is via the generic ObjectRef machinery.

 - Let-bound variables, which can be shared:
   ```
     let %x = relay.add(%y, %z);
     relay.subtract(%x, %x)
   ```
   All referential occurrences of %x share the same node as the definitional occurrence
   (again using the generic ObjectRef machinery). However, we say the sharing is explicit
   in this case.

   The `ToANormalForm` pass will let-bind all non-atomic expressions (at the inner-most
   possible scope compatible with all the expression's uses), thus converting implicit
   to explicit sharing.

 - Implicit let-rec bound variables:
   ```
     let %f = fn(%x, %y) { if (%x == 0) { %y } else { %f(%x - 1, add(%y, %y)) } };
     %f(3, ...)
   ```
   Currently only function literals can be let-rec-bound.

   Both let and let-rec forms share the same AST:
   ```
     class LetNode { Var var; Expr value; Expr body; }
   ```

We've identified 4 issues with the above:
1. ML models tend to have extremely deep expression trees:
   ```
     %0 = ...
     %1 = ...%0...
     %2 = ...%1...
     ...
     %1001 = ...%1000...
   ```
   A naive recursive AST visitor could blow the stack on these expression.
   The `MixedModeVisitor` both explicitly unwinds these expression to avoid
   stack overflow, and memoizes visitor results to hide implicit sharing.
   However no equivalent handling is available for post-ToANormalFrom programs,
   and it is the responsibility of each pass author to explicitly iterate
   when visiting `LetNode`s. Better would be to collect individual let-bindings
   into an `Array` so that this optimization is built into the visitors themselves.
   Also note that `MixedModeVisitor` is not used uniformly.

2. Most passes attempt to work with the program in implicit sharing, ie 'graph' form.
   However, it is generally clearer to work with programs in their explicit sharing
   form since it is straightforward to maintain a `Map<Var, T>` for all in-scope variables
   while visiting. Thus we would prefer to make some form of `ToANormalForm` a standard
   part of the early compilation flow.

3. There's nothing to distinguish the let and let-rec form, requiring pass authors to
   check for the syntactic form of the let-bound expression and whether the let-bound
   var is free within it. Better would be to distinguish these forms by a bool or enum.

4. Relay is not explicit about which expressions have side-effects vs which are pure.
   Many optimizations are unsound in the presence of side-effects, such as moving the binding-site
   of an expression, eliminating an expression who's result is not used, and inlining an expression
   into its multiple usage sites. Better would be to explicitly indicate which let-bound expression
   are pure. (This obviously requires purity annotations on primitive operators and a purity
   propagation pass -- at this point we're just focusing on being able to capture the result of that
   analysis.) We'd like to make it easy for engineers to write optimizations which assume purity
   throughout, and rely on our 'driver' to ensure soundness in the presence of side effects.
   (Again we'll leave the design of the driver out of scope and just focus on the 'is pure'
   annotation.)

Our proposal is to replace `LetNode` with a new `LetBlockNode`:
```
  class LetBindingNode : public Object {
   public:
    Var var;
    Expr value;
    Span span;
  };

  enum LetBlockFlavor {
    // The let-bound expressions are pure. Scope is strictly top-to-bottom.
    kPure,
    // The let-bound expression may be impure. Scope is strictly top-to-bottom.
    kImpure,
    // The let-bound expression are pure but may be mutually recursive.
    // (Currently only singleton bindings are permitted.)
    kLetRec
  };

  class LetBlockNode : public ExprNode {
   public:
    Array<LetBinding> bindings;
    LetBlockFlavor flavor;
    Expr body;
  };
```

All passes will need to be changed to use this form atomically. However if the standard
visitors can be shifted first the remaining changes should not be too onerous. See
(device_aware_visitors.h)[https://github.com/apache/tvm/blob/37cd9837ff302e4490696ca57a9fbba6404c7046/src/relay/transforms/device_aware_visitors.h#L129]
for a possible approach for structuring those visitors (which would subsume most of the code
in that file.)

Here's how this proposal tackles the 4 issues:
1. We include `ToANormalForm` (or a less pedantic version which will leave 'small', non-shared sub-expressions in place) as
   a standard early pass, after which `LetBlockNode` will be the main node building up function bodies. The standard
   visitors will work both on `LetBlockNode`s as well as `LetBindingNode`s. There should not be deep chains of `LetBlockNodes`,
   thus naive recursion should be sufficient on the AST in this form.
2. Implicit sharing is removed under this proposal.
3. We make let and let-rec distinguishable by the `LetBlockFlavor`.
4. We make pure and possibly impure bindings distinguishable by the `LetBlockFlavor`.

An (alternative proposal)[https://www.notion.so/octoml/Relax-AST-Design-ca92c5623ad44984baa4f6047d8c239e] is
(slightly simplified):
```
  // LetNode retained.
  // LetBindingNode added as per above.

  class BindingBlockNode : public Object {
    Array<LetBinding> bindings;
  };

  class SeqExprNode : public ExprNode {
   public:
    Array<BindingBlock> blocks;
    Expr body;
  };
```
Under this proposal `BindingBlockNodes` are always considered pure, with
the traditional `LetNode` always considered impure. The motivation for the
'double nesting' of bindings is unclear. Perhaps it is trying to compactly represent
programs such as:
```
  let %x0 = ...pure...
  let %x1 = ...pure...
  let %y = ...impure...
  let %z0 = ...pure...
  let %z1 = ...pure...
  ...result...
```
However that would look like:
```
  SeqExprNode {
    blocks = [
      BindingBlockNode {
        bindings = [%x0=...; %x1=...]
      }
    body = SeqExprNode {
      blocks = [
        BindingBlockNode {
          bindings = [%y=...]
        }
      ]
      body = SeqExprNode {
        blocks = [
          BindingBlockNode {
            bindings = [%z0=...; %z1=...]
          }
        ]
        body = ...result...
      }
    }
  }
```

Compare that with the main proposal:
```
  LetBlockNode {
    bindings = [%x0=...; %x1=...]
    flavor = kPure
    body =
      LetBlockNode {
        bindings = [%y=...]
        flavor = kImpure
        body =
          LetBlockNode {
            bindings = [%z0=...; %z1=...]
            flavor = kPure
            body = ...result...
          }
      }
  }
```
which the author finds much more straightforward.

The alternative proposal does not address issue 3.

The alternative proposal suggest specializing `FunctionNode` body as
a `SeqExpr`. The author feels doing so would require most passes to
handle function bodies as a special case but for uncertain benefit of
the invariant.

----------------------------------------------------------------------

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to a TVM user. 

That generally means:

- Introducing new named concepts.
- Explaining what the feature enables (hint: think in terms of examples).
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.

For internal RFCs (e.g. for compiler internals), this section should focus on how core contributors s
hould think about the change, and give examples of its concrete impact. 

For policy RFCs, this section should provide an example-driven introduction to the policy, 
  and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, 
and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other ML compilers or languages and discuss the experince their community has had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? 
  If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

If there is no prior art, that is fine - your ideas are interesting to us whether they are 
  brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that TVM intentionally diverges from other compilers.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future 
  independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
