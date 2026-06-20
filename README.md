# GTM Browser Event Listener Recipe — onChange (and Beyond)

> **Track any native DOM event in Google Tag Manager with a single-line change — no custom trigger type needed.**

A plug-and-play Google Tag Manager container that wires a native `document.addEventListener` call into GTM's dataLayer via a reusable JavaScript variable callback, then forwards each interaction to GA4 as a custom event — complete with a full DOM tree walk to identify the element even when class names live on parent or sibling nodes.

The pattern is shipped pre-configured for the `change` event, but swapping to `focus`, `blur`, `input`, `keydown`, `copy`, `touchstart`, or any other DOM event requires changing exactly one string.

---

## ⬇ Download

**[GTM Container JSON — Google Drive](https://drive.google.com/file/d/1dAn_-tCqAJ2aIsZSoZdKXyR9OSPWFLe5/view?usp=drivesdk)**

---

## Credits

This recipe is a modern adaptation of a technique first published by **[Simo Ahava](https://www.simoahava.com)** in 2013. Simo demonstrated how to use a Custom HTML tag + Custom JavaScript Variable to attach any native browser event listener to GTM's dataLayer — a pattern that remains the cleanest solution for DOM events GTM doesn't natively support. This container updates that approach for GA4, environment-aware Stream ID routing, and a richer element-identification variable stack.

---

## Contents

```
on-change-browser-user-interaction-listener-recipe.json   ← GTM container export (import this)
README.md                                                  ← this file
```

---

## Architecture

```
GTM Initializes
      │
      ▼
[Trigger] gtm.js load  (Custom Event: gtm.js)
      │
      ▼
[Tag] onChange Listener  (Custom HTML)
      │  document.addEventListener('change', {{generic event handler}}, false)
      │  ↳ legacy IE: document.attachEvent('onchange', {{generic event handler}})
      │
      │  [User changes a form field / interacts with tracked element]
      │
      ▼
[Variable] generic event handler  (Custom JavaScript Variable)
      │  Callback pushes to dataLayer:
      │  {
      │    event:              'gtm.change',
      │    gtm.element:        e.target,
      │    gtm.elementClasses: e.target.className,
      │    gtm.elementId:      e.target.id,
      │    gtm.elementUrl:     e.target.href || e.target.action,
      │    gtm.originalEvent:  e
      │  }
      │
      ▼
[Trigger] gtm.js change  (Custom Event: gtm.change)
      │
      ▼
[Tag] onChange Event Push  (GA4 Event: "onChange")
           Parameters:
             onChange_class   → {{Coalesce Click Class Name}}
             onChange_element → {{Coalesce Click Elements}}
             onChange_path    → {{Page Path}}
```

---

## Container Assets

### Tags

| Name | Type | Firing Trigger | Consent |
|---|---|---|---|
| onChange Listener | Custom HTML | gtm.js load | `analytics_storage` required |
| onChange Event Push | GA4 Event | gtm.js change | Not needed |
| Google Analytics Configuration | Google Tag | All Pages | Not needed |

### Triggers

| Name | Type | Custom Event Name | What It Fires |
|---|---|---|---|
| gtm.js load | Custom Event | `gtm.js` | onChange Listener tag |
| gtm.js change | Custom Event | `gtm.change` | onChange Event Push GA4 tag |

### Variables

**Folder: Custom Events**

| Name | Type | Description |
|---|---|---|
| generic event handler | Custom JavaScript | Returns a dataLayer-push callback keyed on `'gtm.' + e.type` |

**Folder: Element Hierarchy**

| Name | Type | dataLayer Key / Source |
|---|---|---|
| Click Element Parent Class Name | Data Layer | `gtm.element.parentElement.className` |
| Click Element Uncle Class Name | Data Layer | `gtm.element.parentElement.previousSibling.className` |
| Click Element Grandparent Class Name | Data Layer | `gtm.element.parentElement.parentElement.className` |
| Click Element Great Uncle Class Name | Data Layer | `gtm.element.parentElement.parentElement.previousSibling.className` |
| Click Element Great Grandparent Class Name | Data Layer | `gtm.element.parentElement.parentElement.parentElement.className` |
| Click Element Great Grandparent thru Great Uncle Class Name | Data Layer | Combined great-grandparent + sibling walk |
| Element Name | Auto Event Variable | `name` attribute of the event target |
| Element Value | Auto Event Variable | `value` attribute of the event target |
| Element Title | Auto Event Variable | `title` attribute of the event target |
| Coalesce Click Class Name | Custom Template (If Else If) | First non-empty class name walking up the DOM tree |
| Coalesce Click Elements | Custom Template (If Else If) | First non-empty label: Click Text → Title → Name → Value |

**Folder: Analytics**

| Name | Type | Description |
|---|---|---|
| Measurement Stream ID Development | Constant | Your GA4 dev stream Measurement ID (`G-XXXXXXXXXX`) |
| Measurement Stream ID Production | Constant | Your GA4 prod stream Measurement ID |
| Environment Stream ID | Custom Template (If Else If) | Routes dev vs prod based on GTM Environment Name |
| Measurement Stream RegEx Lookup | Regex Match Table | Final Stream ID selector |
| Google Tag Shared Configuration Settings | Google Tag Config Variable | Shared config for all Google Tags |
| Google Tag Shared Event Settings | Google Tag Event Settings Variable | Shared event settings |

**Built-in Variables used**

`Page URL`, `Page Path`, `Click Element`, `Click Classes`, `Click ID`, `Click Target`, `Click URL`, `Click Text`, `Environment Name`

### Custom Templates

| Template | Purpose |
|---|---|
| If Else If – Advanced Lookup Table | Powers Coalesce variables and environment Stream ID routing with ordered conditional fallback |

---

## The Core Tag — onChange Listener

```html
<script>
  var eventType = 'change'; // ← Change this string to track any other DOM event

  if (document.addEventListener) {
    document.addEventListener(eventType, {{generic event handler}}, false);
  }
  else if (document.attachEvent) {
    // Legacy IE8 and below
    document.attachEvent('on' + eventType, {{generic event handler}});
  }
</script>
```

**Key design decisions:**

- Attached to `document`, not to individual elements — captures events from all current and future elements through bubbling.
- Fires on `gtm.js` (the GTM library load event) — the earliest safe moment to register a listener.
- The `false` third argument to `addEventListener` means the handler runs in the **bubbling phase**, which is correct for delegated event handling.
- Requires `analytics_storage` consent before the listener is attached at all.

---

## The Generic Event Handler — JavaScript Variable

```javascript
function() {
  return function(e) {
    window.dataLayer.push({
      'event':              'gtm.' + e.type,     // 'gtm.change', 'gtm.focus', 'gtm.blur', etc.
      'gtm.element':        e.target,            // DOM element reference
      'gtm.elementClasses': e.target.className || '',
      'gtm.elementId':      e.target.id || '',
      'gtm.elementTarget':  e.target.target || '',
      'gtm.elementUrl':     e.target.href || e.target.action || '',
      'gtm.originalEvent':  e                   // full native Event object
    });
  }
}
```

The outer function is the GTM Custom JavaScript Variable wrapper (must return a value). The inner function is the actual event callback passed to `addEventListener`. Because the event name is derived from `e.type` rather than hard-coded, this same variable works for every event type — the dataLayer key is always `'gtm.' + [whatever event you're listening to]`.

---

## Coalesce Variable Logic

### Coalesce Click Class Name

Evaluates each entry in order using the **If Else If – Advanced Lookup Table** template. Returns the first value that does **not** match the garbage-value regex `^(undefined|null|0|true|false|NaN|)$|\[object`:

1. `{{Click Classes}}` — the element itself
2. `{{Click Element Parent Class Name}}`
3. `{{Click Element Uncle Class Name}}`
4. `{{Click Element Grandparent Class Name}}`
5. `{{Click Element Great Uncle Class Name}}`
6. `{{Click Element Great Grandparent Class Name}}`
7. `{{Click Element Great Grandparent thru Great Uncle Class Name}}`

This walk handles custom component libraries (Material UI, Tailwind component wrappers, Angular Material, etc.) where the semantically meaningful class name sits on a wrapper element rather than the native input.

### Coalesce Click Elements

Returns the first non-garbage value from:

1. `{{Click Text}}` — visible text content (best for buttons and labels)
2. `{{Element Title}}` — `title` attribute
3. `{{Element Name}}` — `name` attribute (most reliable for form inputs)
4. `{{Element Value}}` — `value` attribute (fallback for inputs without a name)

---

## GA4 Event Parameters

Register these as **Event-scoped Custom Dimensions** in GA4 (`Admin → Custom Definitions → Custom Dimensions`):

| GA4 Parameter | GTM Variable | What It Contains |
|---|---|---|
| `onChange_class` | `{{Coalesce Click Class Name}}` | First meaningful CSS class name found walking up the DOM from the changed element |
| `onChange_element` | `{{Coalesce Click Elements}}` | First human-readable label: text, title, name, or value |
| `onChange_path` | `{{Page Path}}` | URL path where the interaction occurred |

---

## Every DOM Event You Can Track — One String Change

Replace `'change'` in the Custom HTML tag with any of the following. Update the downstream Custom Event trigger's filter from `gtm.change` to `gtm.[your event type]`.

### Form & Input Events

| Event | Fires When | Use Case |
|---|---|---|
| `change` | Committed value change (on blur for text; immediate for select/checkbox) | Dropdown selections, checkbox toggles, file chooser |
| `input` | Value changes on every keystroke or paste | Real-time search, character count milestones |
| `focus` | Element receives focus | Which field a user started filling; funnel entry |
| `blur` | Element loses focus | Field abandonment detection |
| `submit` | Form is submitted | SPA forms that don't bubble to GTM's built-in trigger |
| `reset` | Form is reset to defaults | Frustration / restart signal |
| `invalid` | HTML5 native validation fails | Which fields are left empty or invalid |
| `select` | Text inside input/textarea is selected | Copy-intent on product codes, addresses, phone numbers |

### Mouse & Pointer Events

| Event | Fires When | Use Case |
|---|---|---|
| `mouseenter` | Pointer enters element bounding box (no bubble) | Hover intent on CTAs, sticky nav |
| `mouseleave` | Pointer leaves element bounding box (no bubble) | Exit intent on modals, cart panels |
| `mouseover` | Pointer enters element or children (bubbles) | Tooltip impressions, image hover |
| `mouseout` | Pointer leaves element or children (bubbles) | Navigation panel close detection |
| `mousemove` | Pointer moves — fires very frequently, throttle before use | Attention heatmaps, rage-move detection |
| `mousedown` | Mouse button pressed | Long-press detection, drag start |
| `mouseup` | Mouse button released | Drag end, text selection complete |
| `contextmenu` | Right-click / long-press context menu | Intent to copy or save images and links |
| `dblclick` | Double-click | Double-tap on mobile, edit-in-place activation |
| `wheel` | Mouse wheel or trackpad scroll gesture | Direction-aware scroll complement to GTM's scroll trigger |

### Keyboard Events

| Event | Fires When | Use Case |
|---|---|---|
| `keydown` | Key pressed (fires before character appears) | Shortcut detection, Escape to dismiss |
| `keyup` | Key released | Enter-to-submit in search boxes |
| `keypress` | Character-producing key pressed (deprecated but supported) | Legacy form character tracking |

### Touch Events

| Event | Fires When | Use Case |
|---|---|---|
| `touchstart` | Touch point placed on screen | Mobile interaction start, swipe initiation |
| `touchend` | Touch point lifted | Tap vs. swipe disambiguation |
| `touchmove` | Touch point moves | Gesture classification |
| `touchcancel` | Touch interrupted (e.g. incoming call) | Session interruption detection on mobile |

### Clipboard Events

| Event | Fires When | Use Case |
|---|---|---|
| `copy` | Content copied to clipboard | Track sharing of codes, addresses, phone numbers |
| `cut` | Content cut to clipboard | Rich text editor behaviour |
| `paste` | Content pasted from clipboard | Auto-fill detection, document editor usage |

### Drag & Drop Events

| Event | Fires When | Use Case |
|---|---|---|
| `dragstart` | Drag begins | File upload widgets, sortable lists |
| `dragend` | Drag ends (drop or cancel) | Completion rate of drag-and-drop flows |
| `drop` | Dragged element dropped on valid target | File drop zone tracking |

### Media Events

> Media events do not bubble — attach the listener to the specific media element rather than `document`.

| Event | Fires When | Use Case |
|---|---|---|
| `play` | Playback starts | HTML5 video/audio (non-YouTube) |
| `pause` | Playback paused | Engagement depth |
| `ended` | Media reaches the end | Completion rate |
| `timeupdate` | Playback position changes — fires very frequently, throttle | Quartile progress milestones |

> ⚠️ **High-frequency events** (`mousemove`, `input`, `wheel`, `touchmove`, `timeupdate`) fire dozens of times per second. Add a debounce or throttle to the generic event handler before using these in production to avoid exhausting GA4 collection limits.

---

## How to Import

1. Download `on-change-browser-user-interaction-listener-recipe.json` from the [Google Drive link](https://drive.google.com/file/d/1dAn_-tCqAJ2aIsZSoZdKXyR9OSPWFLe5/view?usp=drivesdk).
2. In GTM → **Admin → Import Container**.
3. Upload the JSON file and choose **Merge**.
4. Set your GA4 Measurement IDs in the **Measurement Stream ID Development** and **Measurement Stream ID Production** constant variables.
5. In GA4 → **Admin → Custom Definitions → Custom Dimensions**, register `onChange_class`, `onChange_element`, and `onChange_path` as event-scoped dimensions.
6. Open **GTM Preview mode**, interact with a form field, and confirm:
   - `onChange Listener` fires on page load (triggered by `gtm.js`).
   - A `gtm.change` event appears in the dataLayer after a field interaction.
   - `onChange Event Push` fires and a GA4 hit is sent.
7. To track a different event:
   - Open the **onChange Listener** tag and change `'change'` to your target event type.
   - Open the **gtm.js change** trigger and update the Custom Event filter value from `gtm.change` to `gtm.[your event type]`.
   - Optionally rename both the tag and trigger to reflect the new event.
8. Publish the container.

---

## Extending the Recipe

### Track multiple events simultaneously

Duplicate the Custom HTML tag, change the `eventType` string, and add a corresponding Custom Event trigger. Each tag registers its own listener and the generic event handler produces a distinct dataLayer event name for each one — `gtm.focus`, `gtm.blur`, `gtm.input` — so their downstream tags don't interfere.

### Add a debounce for high-frequency events

Wrap the inner function in the generic event handler with a throttle:

```javascript
function() {
  var timer;
  return function(e) {
    clearTimeout(timer);
    timer = setTimeout(function() {
      window.dataLayer.push({
        'event':              'gtm.' + e.type,
        'gtm.element':        e.target,
        'gtm.elementClasses': e.target.className || '',
        'gtm.elementId':      e.target.id || '',
        'gtm.elementUrl':     e.target.href || e.target.action || '',
        'gtm.originalEvent':  e
      });
    }, 300); // 300ms debounce
  };
}
```

### Add more GA4 parameters

Add rows to the `eventSettingsTable` in the **onChange Event Push** tag. Any GTM variable is valid as a value — for example, `{{Click ID}}`, `{{Element Value}}`, or `{{Click URL}}`.

---

## Requirements

- GTM Web container (not server-side)
- GA4 property with a configured Data Stream
- `analytics_storage` consent must be granted for the listener to attach
- Modern browser with `addEventListener` support; IE8 fallback via `attachEvent` is included

---

## License

MIT — use freely, attribution appreciated.

---

## Credits

- Original DOM event listener technique: **[Simo Ahava](https://www.simoahava.com)**, 2013
- If Else If – Advanced Lookup Table custom template: bundled in container
- Container design and GA4 integration: see blog post for full walkthrough
