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
Style parameters are stored as components, and serve to control some element of its presentation, such as the font, background color or text alignment.
Style parameters can be reused across disparate widget types, with functionality achieved through systems that query for the relevant components.
The type of the style parameter components determines which behavior it controls,
while the value returned by `.get()` controls the final behavior of the widget.

Every style parameter has both a `MyStyle<Base>` and a `MyStyle<Final>` variant, stored together on each entity.

When working on complex games or applications, you're likely to want to group your styles into **themes**,
automatically applying them to large groups of widgets at once.
In Bevy, themes are applied by adding a generic system that corresponds to that theme to your app's schedule.
Select a marker component type `W`, then add a theme system to your app with `W` as a type parameter:
`app.add_startup_system(solarized_theme::<Button>.system())` will then create an entity that stores the solarized theme
style parameters to your world, and adds the appropriate `Entity` reference to the end of each of the `Styles` components
on all entities with `Widget` that have the `Button` marker component on them.

Generally, you'll want to add these systems to a startup stage, but you may also find it useful to add them to various `State`s.
This can work well for quickly toggling between various themes, like you might see in a light-dark mode.

### Example: Building a Simple Widget

TODO: add example

### Example: Building a Compound Widget

TODO: add example

### Example: Stacking Styles

TODO: add example

### Example: Setting a Theme

TODO: add example

### Example: Toggling Light vs. Dark Mode

TODO: add example

## Reference-level explanation

### Style data flow

At the heart of the styling design is a data flow that propagates style parameters from the style to the end widget.
Naively, you'd like to just overwrite the parameter in question, applying the first style's value if any, then the next and so on.
Unfortunately, this causes issues with dynamically applying and reverting styles, because the original value is lost completely.

In order to get around this, we need to somehow duplicate our data, storing both the original and final values.
This is done by creating two variants of each style parameter component: `MyStyleParameter<Base>` and `MyStyleParameter<Final>`,
where `Base` and `Final` are simple unit structs used by systems to disambiguate which version of the data are being referred.

In order for the data to be propagated from our styles to our widgets, we need a set of **style propagation systems**, that work like so:

```rust
/// Automatically updates the styling for all style parameter components of type `S`
///
/// Styles are rebuilt from scratch, working from S.base() each time the styles are changed
/// End users should register this system in your app using `app.add_style::<S>()
pub fn propagate_style<S: Component>(widget_query: Query<(&S<Base>, &mut S<Final>, &Styles), 
  (With<Widget, Without<Style>, Changed<Styles>>)>,
  style_query: Query<Option<&S>, With<Style>>){

 for (style_param, styles) in widget_query.iter_mut(){
  for style in style.iter(){
   // Grab the corresponding style_param off of each style in order
   // using style_query.get()

   // If the style is set in that style, use style_param.set() to apply it
   // This replaces any previous values for the `final` field
  }
 }
}
```

Style propagation systems run every loop;
the `Changed<Styles>` filter will prevent us from doing unnecessary work.

When we call `app.add_style::<S>()`, the following steps occur:

1. We add a `maintain_style::<S>` system to `CoreStage::PreUpdate`, which adds and removes the `Base` and `Final` variants of `S` to entities as needed.
2. We add the `propagate_style::<S>` system to `CoreStage::Update`.

This is done under the hood for all of the built-in style parameters as part of `DefaultPlugins`.

### Theming systems

Themes are applied to the app using **theming system**, which, when run, adds the appropriate style to the widgets with the correct component.
Users can control the priority of the themes by controlling the relative run order of their theming systems in the usual fashion.

These should be constructed ad hoc following a pair of simple examples.

```rust
pub MyTheme {
 pub style_entity: Entity
}

/// Adds an instantiated style to all widgets with the component `M`
pub fn my_theme<M: Component>(widget_query: Query<&mut Styles, (With<M>, With<Widget>)>, theme: Res<MyTheme>){
 for styles in widget_query.iter_mut(){
  styles.push(theme.style_entity);
 }
}

/// Adds the style stored in the `style_scene` to all widgets with the component `M`
pub fn my_stored_theme<M: Component>(mut commands: Commands, 
 widget_query: Query<Entity, (With<Styles>, With<M>, With<Widget>)>, 
 style_scene: Handle<DynamicScene>){

 let widget_entities = widget_query.iter().collect();
 
 // Loads in the style scene and create an entity out of it
 // Then, pushes that entity onto the end of the `Styles` component on each of those entities 
 commands.apply_stored_theme(style_scene, entities);
}
```

## Drawbacks

We clearly need *some* standard, but the devil is in the details.
Leave your complaints and we'll list them here.

## Rationale and alternatives

TODO: discuss why it should be part of the ECS. Discuss integration with scenes.

TODO: discuss why we want this local styling.

TODO: Explain why we need the Style marker component (avoid widget inheritance)

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

TODO: dunk on CSS some more.

> So in CSS you have rules which define the properties to be applied and selectors which determine which elements to apply those rules to. Selectors can be made up of a few different parts. There's the simple selectors like element, id, and class name. There's pseudo-selectors, for selecting elements based on a state they are in like hovered. And there's also a way to select elements based on relationships with other elements (such as direct children and descendants) as well as some other more specific selectors. Anyway, each of these different kinds of selectors have an associated specificity. This specificity determines which rules should apply to an element if more than one could apply. If the specificities are the same then it defaults to order declared.

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

1. How does this relate to the event-passing that the UI must perform? Does it play nicely?
2. How does this connect into widget layout?
3. Can / should we use trait queries to reduce the boilerplate involved in adding new style parameters?

## Future work

Eventually, we can use the `Widget` and `Style` marker components to enforce appropriate `Entity` pointers in both engine and user code using [kinded entities](https://github.com/bevyengine/bevy/issues/1634).
