# Custom Widget, Content Fragment & Adobe Launch — Applied Scenario

Extending the Property Listing component with three previously-missing patterns, all in one connected scenario.

## Granite UI Custom Widget

A custom dialog field needs: a component extending the base form field, HTL markup with a real hidden `<input>` that Coral's submit logic actually reads, and JS that calls `.trigger("change")` on every value update — without it, Coral's dirty-tracking never notices the field changed, and the value silently fails to save despite looking correct on screen.

## Content Fragment API

```java
ContentFragment fragment = resource.adaptTo(ContentFragment.class);
Integer score = fragment.getElement("walkabilityScore").getValue(Integer.class);
```

Structured fields (number/boolean/date/multi-value) use `getValue(Class)`; plain text uses `getContent()`. Reference fields (a property page pointing at a separately-authored fragment) are resolved in two steps: get the `Resource` at the stored path, then adapt/construct the model from THAT resource — not from the calling component's own resource.

## Adobe Launch / Client Context

Page-load facts are rendered server-side via a Sling Model that composes with (not duplicates) the existing component model. Interaction events are fired client-side in JS, but always read their payload from a `data-*` attribute the Java model already rendered — keeping the model the single source of truth for what the data IS, while JS only decides WHEN to send it.

```html
<script>window.adobeDataLayer.push(${dataLayer.dataLayerJson @ context='unsafe'});</script>
<button data-property-id="${dataLayer.propertyId}">RSVP</button>
```
