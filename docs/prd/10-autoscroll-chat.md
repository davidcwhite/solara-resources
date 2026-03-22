# Auto-scrolling Chat in Solara -- Implementation Guide

Build a ChatGPT / WhatsApp-style chat UI in Solara where messages flow top-to-bottom
(oldest at top, newest at bottom) and the viewport automatically scrolls to the
**start** of the latest assistant reply whenever new messages are added. The user can
scroll up to review history; when they send a new message the view snaps back to the
fresh assistant response.

---

## File structure

```
your-project/
  app.py               # Solara application (Python)
  AutoScroll.vue        # Vue component that drives the scroll behavior
  requirements.txt      # solara
```

| File | Role |
|---|---|
| `app.py` | Defines CSS constants, message state, the `Page` component, and registers the Vue component. |
| `AutoScroll.vue` | A zero-height hidden `<div>` that installs a `MutationObserver` on the scroll container and scrolls to the last assistant message when the DOM changes. |

---

## Step 1 -- Lock the viewport (CSS)

### The problem

Solara renders your `Page` inside a deeply nested Vuetify container hierarchy:

```
html > body
  > div.v-application
    > div.v-application--wrap
      > main.v-content.solara-content-main
        > div.v-content__wrap
          > div                          <-- default page-scroll container
            > div.solara-autorouter-content
              > div.v-sheet              <-- your Page's outermost element
```

Every layer defaults to `overflow: visible` (or `auto`) with no fixed height.
Your chat container sits several levels deep, so even if you set `height` and
`overflow-y: auto` on it directly, the parent chain still allows unbounded growth
and the **page-level scrollbar** takes over. The chat never gets its own scrollbar
and auto-scroll has nothing to scroll.

### The fix

Force every ancestor in the chain to `height: 100vh` (or `100%`) and
`overflow: hidden` so that the chat container becomes the **only** scrollable
element on the page.

```python
# In app.py -- inject via solara.Style(VIEWPORT_LOCK_CSS) inside Page()

VIEWPORT_LOCK_CSS = """
/* --- 1. Lock every Vuetify ancestor to the viewport --- */
html, body {
    height: 100vh !important;
    max-height: 100vh !important;
    overflow: hidden !important;
}

.v-application,
.v-application--wrap {
    height: 100vh !important;
    max-height: 100vh !important;
    overflow: hidden !important;
}

.v-content.solara-content-main {
    height: 100vh !important;
    max-height: 100vh !important;
    overflow: hidden !important;
    padding: 0 !important;
}

.v-content__wrap {
    height: 100% !important;
    overflow: hidden !important;
}

/* This anonymous div is the default page-scroll container.
   Locking it is the single most important rule. */
.v-content__wrap > div {
    height: 100% !important;
    overflow: hidden !important;
}

.solara-autorouter-content {
    height: 100% !important;
    overflow: hidden !important;
}

.solara-autorouter-content > .v-sheet {
    height: 100% !important;
    overflow: hidden !important;
}

/* --- 2. Your app layout --- */
.app-shell {
    height: 100% !important;
    max-height: 100% !important;
    overflow: hidden !important;
    box-sizing: border-box;
}

.chat-panel {
    flex: 1 1 0 !important;
    min-height: 0 !important;
    overflow: hidden !important;
}

/* Pin the ChatInput at the bottom of the panel */
.chat-panel > .v-sheet:last-child {
    flex-shrink: 0 !important;
}
"""
```

> **Why `!important` everywhere?** Vuetify injects its own utility classes
> (`d-flex`, `ma-0`, etc.) and inline styles at high specificity. Without
> `!important`, your rules lose the specificity war and have no effect.

---

## Step 2 -- Create the scroll container (CSS + Python)

We do **not** use `solara.lab.ChatBox` for the auto-scroll layout. `ChatBox` uses
`flex-direction: column-reverse` internally (newest messages anchored at the
bottom, growing upward). We want a normal top-to-bottom chronological flow with
programmatic scroll control, so we use a plain `solara.Column` instead.

### CSS

```python
SCROLL_CONTAINER_CSS = """
.chat-history-autoscroll {
    flex: 1 1 0 !important;
    min-height: 0 !important;
    overflow-y: auto !important;
    padding: 16px;
    row-gap: 12px;
}
"""
```

Three rules are doing all the work:

| Rule | Why it matters |
|---|---|
| `flex: 1 1 0` | Tells the container to grow and fill all remaining space inside `.chat-panel`. |
| `min-height: 0` | Overrides the flexbox default (`min-height: auto`) which would let the container grow to fit its content instead of shrinking and scrolling. **This is the most commonly missed rule.** |
| `overflow-y: auto` | Gives the container its own scrollbar. |

### Python

```python
with solara.Column(classes=["chat-panel"], gap="0"):
    with solara.Column(classes=["chat-history-autoscroll"]):
        for message in messages.value:
            MessageBubble(message)
        AutoScroll(trigger=len(messages.value))

    solara.lab.ChatInput(send_callback=send_message)
```

Key points:

- Messages are rendered in their **natural list order** (oldest first).
- `AutoScroll` is placed **after** the messages, at the bottom of the Column.
- `trigger=len(messages.value)` changes whenever the message list grows, which
  is used as a signal by the Vue component.

---

## Step 3 -- The AutoScroll Vue component

Create `AutoScroll.vue` in the same directory as `app.py`:

```vue
<template>
  <div style="display: none;"></div>
</template>

<script>
export default {
  props: ["trigger"],
  data() {
    return {
      observer: null,
      scrollTimer: null,
    };
  },
  mounted() {
    this.installObserver();
    this.doScroll();
  },
  beforeDestroy() {
    if (this.observer) this.observer.disconnect();
    if (this.scrollTimer) clearTimeout(this.scrollTimer);
  },
  methods: {
    doScroll() {
      if (this.scrollTimer) clearTimeout(this.scrollTimer);
      this.scrollTimer = setTimeout(function () {
        var container = document.querySelector(".chat-history-autoscroll");
        if (!container) return;
        var msgs = container.querySelectorAll(".assistant-msg");
        if (msgs.length === 0) return;
        var last = msgs[msgs.length - 1];
        var containerTop = container.getBoundingClientRect().top;
        var msgTop = last.getBoundingClientRect().top;
        container.scrollTop += msgTop - containerTop;
      }, 50);
    },
    installObserver() {
      if (this.observer) this.observer.disconnect();
      var target = document.querySelector(".chat-history-autoscroll");
      if (!target) return;
      var self = this;
      var debounce = null;
      this.observer = new MutationObserver(function () {
        if (debounce) clearTimeout(debounce);
        debounce = setTimeout(function () {
          self.doScroll();
        }, 50);
      });
      this.observer.observe(target, { childList: true, subtree: true });
    },
  },
  watch: {
    trigger: {
      handler: function () {
        var self = this;
        this.$nextTick(function () {
          self.installObserver();
          self.doScroll();
        });
      },
      immediate: false,
    },
  },
};
</script>
```

### How it works

The component uses two independent mechanisms to trigger scrolling, because
neither one is reliable on its own inside Solara's widget lifecycle:

1. **`MutationObserver`** -- installed on the scroll container, fires whenever
   child elements are added or changed. This catches the DOM mutations caused
   by new messages being rendered. A 50ms debounce collapses rapid-fire
   mutations into a single scroll.

2. **`trigger` prop watcher** -- fires when Solara updates the widget with a new
   `trigger` value. On each fire it reinstalls the MutationObserver (in case the
   container element was replaced) and calls `doScroll()`.

### Scroll target

`doScroll()` does **not** scroll to the bottom of the container. Instead it:

1. Finds all elements with the `.assistant-msg` class.
2. Takes the last one (the newest assistant reply).
3. Calculates the offset between the message's top edge and the container's top
   edge.
4. Adjusts `scrollTop` by that offset so the **start** of the assistant message
   aligns with the top of the visible area.

This means long assistant replies are readable from the first line -- the user
scrolls down at their own pace to finish reading.

---

## Step 4 -- Python wiring

### Register the Vue component

```python
@solara.component_vue("AutoScroll.vue")
def AutoScroll(trigger=0):
    pass
```

The `trigger` parameter becomes a Vue prop. Solara passes it to the widget
whenever the component re-renders.

### Mark assistant messages with a CSS class

The Vue component locates the scroll target using `.assistant-msg`. Add this
class to assistant message bubbles:

```python
@solara.component
def MessageBubble(message):
    is_user = message["role"] == "user"
    classes = ["chat-message"] if is_user else ["chat-message", "assistant-msg"]

    with solara.lab.ChatMessage(
        user=is_user,
        name=message["name"],
        color="#dbeafe" if is_user else "#f3f4f6",
        notch=True,
        classes=classes,
    ):
        solara.Markdown(message["body"])
```

### Inject CSS and render the layout

```python
@solara.component
def Page():
    solara.Style(VIEWPORT_LOCK_CSS)
    solara.Style(SCROLL_CONTAINER_CSS)

    with solara.Column(classes=["app-shell"]):
        # Optional: header, title, toolbar, etc.

        with solara.Column(classes=["chat-panel"], gap="0"):
            with solara.Column(classes=["chat-history-autoscroll"]):
                for message in messages.value:
                    MessageBubble(message)
                AutoScroll(trigger=len(messages.value))

            solara.lab.ChatInput(send_callback=send_message)
```

---

## Gotchas

These are the problems we hit during development. Each one cost significant
debugging time. Read all of them before you start.

### 1. Stale DOM references in Vue components

**Symptom:** You put a `ref="anchor"` on the Vue template element and use
`this.$refs.anchor.closest(".chat-history-autoscroll")` to find the scroll
container. Setting `scrollTop` on the result has no effect.

**Cause:** Solara's reconciliation engine may destroy and recreate the Vue
widget on every re-render. When the old widget's watcher fires, `$refs.anchor`
points to a **detached** DOM node that is no longer in the document. `.closest()`
returns the old (detached) parent. Setting `scrollTop` on a detached element is
a no-op.

**Fix:** Never rely on `$refs` to navigate upward. Use
`document.querySelector(".chat-history-autoscroll")` instead -- it always
searches the **live** DOM tree.

---

### 2. CSS `scroll-behavior: smooth` blocks instant scroll

**Symptom:** You set `scroll-behavior: smooth` on the scroll container via CSS.
You then assign `el.scrollTop = 0` in JavaScript expecting it to jump instantly.
Instead the scroll animates slowly over several seconds and may not finish before
the next re-render resets it.

**Cause:** When `scroll-behavior: smooth` is set **in CSS**, it applies to
**all** scroll operations on that element -- including direct `scrollTop`
assignments and `scrollTo()` calls, even those without `behavior: "smooth"`.

**Fix:** Do not put `scroll-behavior: smooth` in CSS on the scroll container.
If you want smooth scrolling for specific operations, use it only in individual
`scrollTo()` calls:

```javascript
// Smooth only when you explicitly ask for it:
container.scrollTo({ top: 0, behavior: "smooth" });

// This will be instant if CSS scroll-behavior is NOT set:
container.scrollTop = 0;
```

---

### 3. `!important` is required on every CSS rule

**Symptom:** Your CSS rules for `height`, `overflow`, or `flex` have no visible
effect even though they appear correct.

**Cause:** Vuetify injects its own styles via utility classes (`d-flex`, `ma-0`,
`elevation-0`) and inline style attributes. These have higher specificity than
plain class selectors in a `<style>` block.

**Fix:** Use `!important` on all layout-critical properties. This is
heavy-handed but unavoidable when overriding Vuetify from outside its component
system.

---

### 4. `min-height: 0` is essential on flex children

**Symptom:** The scroll container grows to fit all its messages instead of
showing a scrollbar. The page extends below the viewport.

**Cause:** By default, a flex child's minimum size is its content size
(`min-height: auto`). This prevents it from shrinking below its content, which
defeats `overflow-y: auto` -- the container never overflows because it just
keeps growing.

**Fix:** Add `min-height: 0 !important` to every flex child in the chain from
`.app-shell` down to `.chat-history-autoscroll`. This allows flex children to
shrink smaller than their content, which is what triggers overflow scrolling.

---

### 5. Vue component is recreated on every render (watcher never fires)

**Symptom:** You add a `watch` on the `trigger` prop expecting it to fire when
`trigger` changes from e.g. 4 to 6. Console logs in the watcher never appear.

**Cause:** Solara may destroy the old Vue widget instance and create a new one
with the updated prop value. The new instance starts fresh -- the watcher sees
the prop as its initial value, not a change from the old value. So the watcher
never fires.

**Fix:** Do not rely solely on the `trigger` watcher. Use a `MutationObserver`
as the primary scroll mechanism (it fires on DOM changes regardless of component
lifecycle). Keep the watcher as a backup that reinstalls the observer on each
render.

---

### 6. `.vue` file changes require a server restart

**Symptom:** You edit `AutoScroll.vue`, save, and refresh the browser. The old
behavior persists.

**Cause:** Solara's hot-reload watches `.py` files but does not reliably
hot-reload `.vue` files used by `solara.component_vue`. The old compiled Vue
component remains cached.

**Fix:** Kill the Solara server process and restart it after any `.vue` file
change. A hard browser refresh (`Cmd+Shift+R` / `Ctrl+Shift+R`) is also
recommended to clear any browser-side caching.

---

### 7. `solara.component_vue` must use decorator syntax

**Symptom:** `TypeError: decorator() got an unexpected keyword argument 'trigger'`
when calling `AutoScroll(trigger=len(messages.value))`.

**Cause:** Using `solara.component_vue` as a direct assignment
(`AutoScroll = solara.component_vue("AutoScroll.vue")`) returns a decorator
function, not a component.

**Fix:** Always use the decorator form:

```python
# Correct:
@solara.component_vue("AutoScroll.vue")
def AutoScroll(trigger=0):
    pass

# Wrong -- returns a decorator, not a component:
AutoScroll = solara.component_vue("AutoScroll.vue")
```

The function body is intentionally `pass`. The parameter names and defaults
define the Vue props.

---

## Integrating with your own CSS

If you are adding auto-scroll to an existing Solara app that already applies
custom CSS to chat components, watch out for these conflicts.

### Your CSS sets `height` or `overflow` on the chat container

If your codebase already has rules like:

```css
.my-chat-container {
    height: 600px;
    overflow-y: scroll;
}
```

These will conflict with the viewport-lock approach. The viewport lock needs the
container to use `flex: 1 1 0` (fill remaining space) rather than a fixed pixel
height, because a fixed height won't adapt to different screen sizes or to
headers/toolbars above the chat.

**Resolution:** Replace fixed `height` values with the flex approach. If you
need a maximum height, use `max-height` alongside `flex`:

```css
.chat-history-autoscroll {
    flex: 1 1 0 !important;
    min-height: 0 !important;
    max-height: 800px;            /* optional cap */
    overflow-y: auto !important;
}
```

### Your CSS applies `scroll-behavior: smooth` anywhere in the ancestor chain

Even if `scroll-behavior: smooth` is set on a parent element, it can be
inherited by the scroll container. Check with browser DevTools (Computed tab)
that `scroll-behavior` on `.chat-history-autoscroll` is `auto`, not `smooth`.

**Resolution:** Explicitly override it on the scroll container:

```css
.chat-history-autoscroll {
    scroll-behavior: auto !important;
}
```

### Your CSS uses custom classes on `ChatMessage`

The auto-scroll Vue component finds the latest assistant message using:

```javascript
container.querySelectorAll(".assistant-msg")
```

If your `ChatMessage` components already use a `classes` prop with your own
class names, make sure you **add** `assistant-msg` to the list rather than
replacing it:

```python
# If you already have custom classes on assistant messages:
existing_classes = ["my-bubble", "my-assistant-style"]

# Add the scroll-target marker alongside your existing classes:
classes = existing_classes + ["assistant-msg"]

with solara.lab.ChatMessage(classes=classes, ...):
    ...
```

If `assistant-msg` conflicts with an existing class name in your project,
rename it in both the Python code and the Vue component. The name itself does
not matter -- it just needs to match in both places. Use something specific to
avoid collisions:

```python
# Python
classes = [..., "autoscroll-target"]
```

```javascript
// AutoScroll.vue -- update the selector to match
var msgs = container.querySelectorAll(".autoscroll-target");
```

### Your CSS overrides the `.chat-panel > .v-sheet:last-child` selector

This selector pins `ChatInput` at the bottom of the panel. If your codebase
wraps `ChatInput` in extra containers, the `:last-child` selector may not
target the right element.

**Resolution:** Inspect the DOM with browser DevTools to find the actual
element that wraps `ChatInput`, and update the selector accordingly. The
key property is `flex-shrink: 0` -- whatever element wraps the input needs
this to prevent it from being squashed when the message list grows.

### Your CSS conflicts with the viewport lock selectors

The viewport lock targets generic Vuetify class names (`.v-application`,
`.v-content`, etc.). If your app already styles these for other purposes
(e.g. custom padding, background colors), the lock rules will override them.

**Resolution:** Add your cosmetic styles **after** the viewport lock
`solara.Style()` call so they come later in the cascade. For properties that
the viewport lock sets (like `padding: 0 !important` on `.v-content`), you
will need to move your padding to a child element instead:

```python
@solara.component
def Page():
    # Viewport lock first (sets padding: 0 on .v-content)
    solara.Style(VIEWPORT_LOCK_CSS)

    # Your cosmetic overrides second
    solara.Style("""
        .app-shell {
            padding: 24px;          /* add padding here instead */
            background: #f0f0f0;
        }
    """)
```

### Summary checklist for existing codebases

- [ ] No fixed `height` on the scroll container -- use `flex: 1 1 0` instead
- [ ] `min-height: 0 !important` on every flex child from shell to scroll container
- [ ] No `scroll-behavior: smooth` on or inherited by the scroll container
- [ ] `assistant-msg` class (or your renamed equivalent) added to assistant bubbles
- [ ] Cosmetic CSS loaded **after** the viewport lock CSS
- [ ] `ChatInput` wrapper has `flex-shrink: 0` so it stays pinned at the bottom
- [ ] `.vue` file is in the same directory as the Python file that registers it
- [ ] Server restarted after any `.vue` file change
