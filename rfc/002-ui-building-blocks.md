# Feature Name: `ui_building_blocks`

## Summary

This RFC establishes a common vocabulary, and basic data structures for UI.
It is combined with a design pattern to manipulate UI styles ergonomically using Bevy's ECS, to tangibly demonstrate the feasibility of the approach.

## Motivation

UI is the next big challenge for Bevy, with a complex set of data structures that need to be stored and manipulated.
In particular, the ability to style elements (by setting) in a reusable fashion is central to a polished, easy-to UI, but there's no immediately obvious approach to handling them.
What makes a "style" in Bevy? How do we represent the individual style parameters? How are widgets represented? How are styles composed? How are they applied?

None of these questions are terribly challenging to implement on their own, but taking the first-seen path is particularly dangerous here, due to the risk of proliferation of non-local configuration and similar-but-not-identical style parameters, as seen in CSS.

We need to settle on a common standard, in order to enable interop between and establish basic building blocks that more complex UI decisions can build off of.

## Guide-level explanation

User interfaces in Bevy are made out of **widgets**, which are modified by the **styles** that are applied to them, and the aesthetic and functional behavior of is ultimately implemented through **UI systems**.

A **widget** is a distinct part of the UI that needs its own data: be that its position, local state or any style parameters.
Widgets in Bevy are represented by entities with a `Widget` marker component.

Each widget has an associated `Styles` component, which contains a `Vec<Entity>` which points to some (possibly 0) number of **styles**, represented as entities with a `Style` marker component.

Each style in that vector is applied, in order, to the widget, overriding the **style parameters** that already exist.
Style parameters components on UI widgets that implement the `StyleParam` trait, serve to control some element of its presentation, such as the font, background color or text alignment.
Style parameters can be reused across disparate widget types, with functionality achieved through systems that query for the relevant style components.
The type of the style parameter components determines which behavior it controlling, while the value returned by `.get()` controls the final behavior of the widget.

### Example: Building a Simple Widget

TODO: add example

### Example: Building a Compound Widget

TODO: add example

### Example: Working with Styles

TODO: add example

## Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

We clearly need *some* standard, but the devil is in the details.
Leave your complaints and we'll list them here.

## Rationale and alternatives

TODO: discuss why it should be part of the ECS.

TODO: discuss why we want this local styling. 


- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

TODO: dunk on CSS some more.

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

1. How does this relate to the event-passing that the UI must perform? Does it play nicely?
2. How does this connect into widget layout?
