# AEM Component Development & HTL

## HTL (Sightly) Essentials

HTL is AEM's server-side templating language. It replaces JSP and enforces XSS-safe output by default.

```html
<!-- Use statement — adapts a Sling Model -->
<sly data-sly-use.model="com.mysite.core.models.Author">
    <div class="author">
        <!-- Output is XSS-escaped automatically -->
        <h1>${model.firstName} ${model.lastName}</h1>

        <!-- Explicit context for HTML output (not escaped) -->
        <div>${model.richTextBody @ context='html'}</div>

        <!-- Attribute context -->
        <a href="${model.email @ context='uri'}">Email</a>

        <!-- Test / conditional -->
        <p data-sly-test="${model.featured}">Featured Author</p>

        <!-- List iteration -->
        <ul data-sly-list.item="${model.tags}">
            <li>${item}</li>
        </ul>

        <!-- Include another component -->
        <sly data-sly-include="header.html"/>

        <!-- Resource inclusion -->
        <sly data-sly-resource="${'footer' @ resourceType='mysite/components/footer'}"/>
    </div>
</sly>
```

## HTL Context Options (XSS)

| Context | Use for |
|---|---|
| (default) | HTML text — escapes `<`, `>`, `&`, `"` |
| `html` | Trust the value as raw HTML (only safe content) |
| `uri` | URL attribute values — encodes unsafe URL characters |
| `scriptString` | Inside JavaScript string literals |
| `styleContext` | CSS property values |
| `attribute` | HTML attribute name (dynamic attributes) |
| `text` | Explicit text context (same as default) |
| `unsafe` | No escaping at all — **never use for user input** |

## HTL Rules to Remember

| Rule | Wrong | Right |
|---|---|---|
| Arithmetic | `${model.page + 1}` | Add getter to model, use `${model.displayPage}` |
| Boolean attribute | `${x ? 'selected' : ''}` | `data-sly-attribute.selected="${x}"` |
| String comparison | `${'literal' == model.val}` | `${model.val == 'literal'}` |
| List contains check | Works as-is | `${model.list contains 'value'}` |
| XSS in href | `href="${model.path}"` | `href="${model.path @ context='uri'}"` |
| XSS in attribute | `value="${model.val}"` | `value="${model.val @ context='attribute'}"` |

## Component Dialog — Key Touch UI Resource Types

| Resource Type | Purpose |
|---|---|
| `granite/ui/components/coral/foundation/form/textfield` | Single-line text input |
| `granite/ui/components/coral/foundation/form/textarea` | Multi-line text |
| `granite/ui/components/coral/foundation/form/checkbox` | Boolean toggle |
| `granite/ui/components/coral/foundation/form/select` | Dropdown |
| `granite/ui/components/coral/foundation/form/pathfield` | Path picker (page/asset) |
| `granite/ui/components/coral/foundation/form/multifield` | Repeating field group |
| `granite/ui/components/coral/foundation/form/numberfield` | Numeric input |
| `cq/gui/components/authoring/dialog/richtext` | Rich text editor |
| `granite/ui/components/coral/foundation/include` | Include another dialog fragment |

## Editable Templates vs Static Templates

| | Static Templates | Editable Templates |
|---|---|---|
| Stored in | `/apps/mysite/templates/` | `/conf/mysite/settings/wcm/templates/` |
| Modifiable by authors? | No — developer only | Yes — template authors via Template Editor |
| Component allowed list | `allowedChildren` property on template | Configured per-container in Template Editor |
| Layout mode | N/A | Authors can set column widths in Layout Mode |
| Recommended? | Legacy — avoid | Yes — standard in all modern AEM projects |

## Common Interview Questions

**Q: How do you prevent XSS in HTL?**
HTL escapes output by default using HTML context. For URLs use `@ context='uri'`, for HTML use `@ context='html'` only with trusted content. Never use `@ context='unsafe'` with user-supplied data.

**Q: What is the difference between `data-sly-include` and `data-sly-resource`?**
`data-sly-include` renders another HTL script in the same component context — the included script shares the same model and bindings. `data-sly-resource` renders a JCR resource (or virtual resource) as a new component — it starts a new Sling request dispatch cycle with its own model.

**Q: How do you make a component inherit from a Core Component?**
Set `sling:resourceSuperType` on your component node to the Core Component path (e.g. `core/wcm/components/text/v2/text`). Your component inherits all HTL scripts and dialog fields, and you only override what you need.

**Q: What is a policy in editable templates?**
A policy is a reusable set of component design settings stored in `/conf`. Multiple template pages can reference the same policy. For example, a "default text" policy might restrict the RTE toolbar to bold/italic/link only. Authors cannot override policy settings — they are design-time constraints.
