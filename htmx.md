# htmx Comprehensive Reference

> htmx ~14KB gzipped. No build step, no dependencies. Current version: **2.0.8**

## 1. Installation

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.8/dist/htmx.min.js"></script>
```

**npm:**
```bash
npm install htmx.org@2.0.8
```

**Webpack/ESM:**
```javascript
import 'htmx.org';
window.htmx = require('htmx.org');
```

**Self-hosted (recommended for production):** download `htmx.min.js` and serve locally.

**Meta config (optional):**
```html
<meta name="htmx-config" content='{"defaultSwapStyle":"outerHTML"}'>
```

---

## 2. Core Attributes

### HTTP Method Attributes

| Attribute | Description |
|-----------|-------------|
| `hx-get` | Issues GET to the specified URL |
| `hx-post` | Issues POST to the specified URL |
| `hx-put` | Issues PUT to the specified URL |
| `hx-patch` | Issues PATCH to the specified URL |
| `hx-delete` | Issues DELETE to the specified URL |

```html
<button hx-get="/api/items" hx-target="#list">Load Items</button>
<button hx-delete="/api/items/5" hx-target="closest tr" hx-swap="outerHTML">Delete</button>
<form hx-put="/api/items/3" hx-target="#item-3" hx-swap="outerHTML">
  <input name="name" value="Updated">
  <button type="submit">Save</button>
</form>
```

### Targeting & Swapping

| Attribute | Description |
|-----------|-------------|
| `hx-target` | CSS selector for the element to swap content into |
| `hx-swap` | Controls how response content is swapped in (default: `innerHTML`) |
| `hx-select` | CSS selector to pick a subset of the response to swap |
| `hx-select-oob` | Select content from response for out-of-band swap |
| `hx-swap-oob` | Mark response elements for out-of-band swapping |

```html
<!-- Target a different element -->
<input hx-get="/search" hx-target="#results" hx-swap="innerHTML">

<!-- Select only part of response -->
<button hx-get="/page" hx-select="#content" hx-target="#main">Load</button>

<!-- Out-of-band swap: server response updates multiple elements -->
<!-- In the response HTML: -->
<div id="notifications" hx-swap-oob="true">3 new messages</div>
```

### Triggers

| Attribute | Description |
|-----------|-------------|
| `hx-trigger` | Event(s) that trigger the request. Defaults: click (most), change (inputs), submit (forms) |

```html
<!-- Multiple triggers -->
<input hx-get="/search" hx-trigger="input changed delay:500ms, keyup[key=='Enter']">

<!-- Polling -->
<div hx-get="/status" hx-trigger="every 5s">Checking...</div>

<!-- Load trigger -->
<div hx-get="/lazy-content" hx-trigger="load">Loading...</div>

<!-- Revealed (scroll into view) -->
<div hx-get="/more" hx-trigger="revealed">Load when visible</div>

<!-- Intersection observer -->
<img hx-get="/analytics" hx-trigger="intersect threshold:0.5">

<!-- Trigger filter -->
<div hx-get="/detail" hx-trigger="click[ctrlKey]">Ctrl+Click me</div>
```

### URL & History

| Attribute | Description |
|-----------|-------------|
| `hx-push-url` | Push URL into browser history bar (`true`, `false`, or a URL) |
| `hx-replace-url` | Replace current URL in history without adding entry |

```html
<a hx-get="/page/2" hx-push-url="true" hx-target="#content">Page 2</a>
<a hx-get="/page/2" hx-replace-url="/custom-url" hx-target="#content">Page 2</a>
```

### UI Feedback

| Attribute | Description |
|-----------|-------------|
| `hx-indicator` | Element to add `htmx-request` class during request (shows loading) |
| `hx-disabled-elt` | Elements to disable during request |
| `hx-confirm` | Shows `confirm()` dialog before issuing request |
| `hx-prompt` | Shows `prompt()` dialog; value sent in `HX-Prompt` header |

```html
<button hx-delete="/item/5" hx-confirm="Are you sure?">Delete</button>

<button hx-post="/submit" hx-indicator="#spinner" hx-disabled-elt="this">
  Submit
</button>
<img id="spinner" class="htmx-indicator" src="/spinner.svg">

<button hx-patch="/rename" hx-prompt="Enter new name:">Rename</button>
```

### Request Configuration

| Attribute | Description |
|-----------|-------------|
| `hx-vals` | JSON values to include in the request |
| `hx-headers` | JSON headers to add to the request |
| `hx-include` | CSS selector for additional elements to include in request data |
| `hx-params` | Filter which params are submitted (`*`, `none`, `not <list>`, or `<list>`) |
| `hx-encoding` | Changes encoding (e.g., `multipart/form-data` for file upload) |
| `hx-request` | Configure request options (timeout, credentials, noHeaders) |

```html
<!-- Extra JSON values -->
<button hx-post="/action" hx-vals='{"key": "value"}'>Go</button>

<!-- Dynamic values via JS (prefix js:) -->
<button hx-post="/action" hx-vals='js:{ts: Date.now()}'>Go</button>

<!-- Custom headers -->
<button hx-get="/api" hx-headers='{"X-Custom": "value"}'>Fetch</button>

<!-- Include other inputs -->
<input id="token" name="token" type="hidden" value="abc">
<button hx-post="/submit" hx-include="#token">Submit</button>

<!-- File upload -->
<form hx-post="/upload" hx-encoding="multipart/form-data">
  <input type="file" name="file">
  <button>Upload</button>
</form>
```

### Behavior & Inheritance

| Attribute | Description |
|-----------|-------------|
| `hx-boost` | Progressive enhancement: converts links/forms to AJAX |
| `hx-disable` | Disables htmx processing for element and children |
| `hx-disinherit` | Prevents attribute inheritance for children (`*` or specific attrs) |
| `hx-inherit` | Re-enables attribute inheritance when disabled |
| `hx-preserve` | Keeps element unchanged between requests (by ID) |
| `hx-sync` | Controls request synchronization between elements |
| `hx-validate` | Forces validation before request |
| `hx-ext` | Activates extensions for this element |
| `hx-history` | Set to `false` to prevent sensitive pages from caching |

```html
<!-- Boost all links in a nav -->
<nav hx-boost="true">
  <a href="/page1">Page 1</a>  <!-- becomes AJAX -->
  <a href="/page2">Page 2</a>
</nav>

<!-- Preserve a video element across swaps -->
<video id="player" hx-preserve>...</video>

<!-- Sync: cancel in-flight requests when new one fires -->
<input hx-get="/search" hx-trigger="keyup" hx-sync="closest form:abort">

<!-- Sync: queue only the last request -->
<form hx-sync="this:queue last">...</form>

<!-- Disable inheritance -->
<div hx-target="#main" hx-disinherit="hx-target">
  <button hx-get="/other">Uses own default target</button>
</div>
```

### Inline Event Handling

| Attribute | Description |
|-----------|-------------|
| `hx-on*` | Handle any htmx or DOM event with inline JS. Format: `hx-on:event-name` |

```html
<!-- Modify request before sending -->
<form hx-post="/submit"
      hx-on::config-request="event.detail.headers['X-Token'] = getToken()">
  ...
</form>

<!-- Handle after swap -->
<div hx-get="/content" hx-on::after-swap="console.log('swapped!')">Load</div>

<!-- Standard DOM events -->
<button hx-on:click="alert('clicked')">Click me</button>
```

---

## 3. Swap Strategies

Set via `hx-swap` attribute. Default is `innerHTML`.

| Strategy | Description |
|----------|-------------|
| `innerHTML` | Replace inner content of target (default) |
| `outerHTML` | Replace the entire target element |
| `beforebegin` | Insert before the target element (sibling) |
| `afterbegin` | Insert before the first child of target |
| `beforeend` | Insert after the last child of target |
| `afterend` | Insert after the target element (sibling) |
| `delete` | Delete the target element, ignore response |
| `none` | No swap; still processes response headers and OOB swaps |

### Swap Modifiers

Append modifiers after the strategy:

```html
<!-- Transition API -->
<div hx-get="/new" hx-swap="innerHTML transition:true">

<!-- Timing control -->
<div hx-get="/new" hx-swap="innerHTML swap:100ms settle:200ms">

<!-- Scroll control -->
<div hx-get="/new" hx-swap="innerHTML scroll:top">
<div hx-get="/new" hx-swap="innerHTML show:#element:top">

<!-- Ignore title tag in response -->
<div hx-get="/new" hx-swap="innerHTML ignoreTitle:true">

<!-- Focus scroll: keep focus on active element -->
<div hx-get="/new" hx-swap="innerHTML focus-scroll:true">
```

| Modifier | Values | Description |
|----------|--------|-------------|
| `transition` | `true` | Use View Transitions API |
| `swap` | `<time>` | Delay before old content is removed |
| `settle` | `<time>` | Delay before new content settles |
| `scroll` | `top`, `bottom` | Scroll target after swap |
| `show` | `top`, `bottom` | Scroll element into viewport |
| `ignoreTitle` | `true` | Don't update document title from response |
| `focus-scroll` | `true`, `false` | Control focus scrolling behavior |

---

## 4. Trigger Modifiers

Full syntax: `hx-trigger="event[filter] modifier1 modifier2, event2"`

| Modifier | Syntax | Description |
|----------|--------|-------------|
| `once` | `click once` | Fire only once |
| `changed` | `input changed` | Fire only if value changed |
| `delay` | `keyup delay:500ms` | Debounce; resets on repeat events |
| `throttle` | `scroll throttle:200ms` | Throttle; ignores events during window |
| `from` | `click from:#other` | Listen on a different element |
| `target` | `click target:.child` | Filter by event.target CSS selector |
| `consume` | `click consume` | Stop event propagation to parent htmx elements |
| `queue` | `click queue:last` | Queue strategy: `first`, `last` (default), `all`, `none` |

### Special Triggers (non-standard events)

| Trigger | Description |
|---------|-------------|
| `load` | Fires when element is loaded into DOM |
| `revealed` | Fires when element scrolls into viewport |
| `intersect` | Fires on intersection; options: `root:<sel>`, `threshold:<0.0-1.0>` |
| `every <time>` | Polling; e.g., `every 2s`. Server returns HTTP 286 to stop. |

### Extended `from` Selectors

```
from:document        — listen on document
from:window          — listen on window
from:closest div     — closest ancestor matching selector
from:find .child     — first child matching selector
from:next            — next sibling
from:previous        — previous sibling
```

### Trigger Filters

JavaScript expressions in square brackets evaluated against the event:

```html
<div hx-get="/data" hx-trigger="click[ctrlKey && shiftKey]">Ctrl+Shift+Click</div>
<input hx-get="/search" hx-trigger="keyup[key=='Enter']">
<div hx-get="/data" hx-trigger="every 1s [isActive()]">Conditional poll</div>
```

---

## 5. CSS Transitions

htmx applies CSS classes during the request lifecycle:

| Class | When Applied | When Removed |
|-------|-------------|--------------|
| `htmx-request` | On requesting element (or `hx-indicator` target) when request starts | When request completes |
| `htmx-indicator` | Permanent class. Elements with this are hidden (opacity:0) until `htmx-request` is on ancestor | N/A |
| `htmx-swapping` | On target before content is swapped out | After swap (default: immediately) |
| `htmx-settling` | On target after new content is swapped in | After settle (default: 20ms) |
| `htmx-added` | On new content before it is swapped in | After settle |

### Loading Indicator Pattern

```html
<button hx-get="/data" hx-indicator="#spinner">Load</button>
<img id="spinner" class="htmx-indicator" src="/spinner.svg">

<style>
  .htmx-indicator { opacity: 0; transition: opacity 200ms ease-in; }
  .htmx-request .htmx-indicator { opacity: 1; }
  .htmx-request.htmx-indicator { opacity: 1; }
</style>
```

### Swap Transition Pattern

```css
/* Fade out old content */
.htmx-swapping { opacity: 0; transition: opacity 300ms ease-out; }

/* Fade in new content */
.htmx-added { opacity: 0; }
.htmx-settling { opacity: 1; transition: opacity 300ms ease-in; }
```

### CSS Transition with ID Matching

htmx transitions work when old and new elements share the same `id`:

```html
<!-- Response changes the class, triggering CSS transition -->
<div id="color-demo" class="red">Red</div>
<!-- After swap, htmx copies old attributes, swaps content, then applies new attributes -->
<div id="color-demo" class="blue">Blue</div>

<style>
#color-demo { transition: all 1s ease-in; }
.red { color: red; } .blue { color: blue; }
</style>
```

---

## 6. Events

### Key Lifecycle Events

```javascript
// Modify request before sending (add headers, params)
document.body.addEventListener("htmx:configRequest", (e) => {
  e.detail.headers["X-CSRF-Token"] = getCsrfToken();
  e.detail.parameters["extra"] = "value";
});

// Before request fires (can cancel with preventDefault)
document.body.addEventListener("htmx:beforeRequest", (e) => {
  if (!isReady()) e.preventDefault(); // cancel request
});

// After successful swap
document.body.addEventListener("htmx:afterSwap", (e) => {
  initializeComponents(e.detail.target);
});

// After DOM settling (final state)
document.body.addEventListener("htmx:afterSettle", (e) => {
  console.log("Content settled in", e.detail.target);
});

// Error handling
document.body.addEventListener("htmx:responseError", (e) => {
  console.error("HTTP error:", e.detail.xhr.status);
});

// Network error
document.body.addEventListener("htmx:sendError", (e) => {
  alert("Network error. Check connection.");
});

// Customize swap behavior
document.body.addEventListener("htmx:beforeSwap", (e) => {
  if (e.detail.xhr.status === 404) {
    e.detail.shouldSwap = true;        // swap even on error
    e.detail.isError = false;          // don't treat as error
  }
  if (e.detail.xhr.status === 422) {
    e.detail.shouldSwap = true;
    e.detail.target = htmx.find("#errors"); // retarget
  }
});

// Abort a request
htmx.trigger(document.getElementById("my-element"), "htmx:abort");

// New content loaded (like htmx's DOMContentLoaded)
htmx.onLoad((elt) => {
  initThirdPartyLibrary(elt);
});
```

### Complete Event List

**Request lifecycle:** `htmx:configRequest` > `htmx:beforeRequest` > `htmx:beforeSend` > `htmx:xhr:loadstart` > `htmx:xhr:progress` > `htmx:beforeOnLoad` > `htmx:beforeSwap` > `htmx:afterSwap` > `htmx:afterSettle` > `htmx:afterOnLoad` > `htmx:afterRequest`

**Error events:** `htmx:responseError`, `htmx:sendError`, `htmx:swapError`, `htmx:targetError`, `htmx:timeout`

**History events:** `htmx:beforeHistorySave`, `htmx:pushedIntoHistory`, `htmx:replacedInHistory`, `htmx:historyRestore`, `htmx:historyCacheHit`, `htmx:historyCacheMiss`, `htmx:historyCacheError`

**OOB events:** `htmx:oobBeforeSwap`, `htmx:oobAfterSwap`, `htmx:oobErrorNoTarget`

**Other:** `htmx:load`, `htmx:abort`, `htmx:confirm`, `htmx:prompt`, `htmx:beforeProcessNode`, `htmx:afterProcessNode`, `htmx:beforeCleanupElement`, `htmx:beforeTransition`, `htmx:validation:validate`, `htmx:validation:failed`, `htmx:validation:halted`

---

## 7. Headers

### Request Headers (sent by htmx)

| Header | Value |
|--------|-------|
| `HX-Request` | Always `"true"` |
| `HX-Trigger` | `id` of the triggering element |
| `HX-Trigger-Name` | `name` of the triggering element |
| `HX-Target` | `id` of the target element |
| `HX-Current-URL` | Current browser URL |
| `HX-Boosted` | `"true"` if request is from a boosted element |
| `HX-Prompt` | User's response to `hx-prompt` |
| `HX-History-Restore-Request` | `"true"` if restoring from history cache |

### Response Headers (sent by server)

| Header | Description |
|--------|-------------|
| `HX-Location` | Client-side redirect without full page reload (JSON: `{"path": "/new", "target": "#main"}`) |
| `HX-Push-Url` | Push URL into browser history |
| `HX-Redirect` | Full client-side redirect |
| `HX-Refresh` | `"true"` forces full page refresh |
| `HX-Replace-Url` | Replace current URL in history |
| `HX-Reswap` | Override the `hx-swap` value from the server |
| `HX-Retarget` | CSS selector to override `hx-target` |
| `HX-Reselect` | CSS selector to pick response subset (overrides `hx-select`) |
| `HX-Trigger` | Trigger client-side events (JSON for event data) |
| `HX-Trigger-After-Settle` | Trigger events after settle phase |
| `HX-Trigger-After-Swap` | Trigger events after swap phase |

```python
# Server-side example (any language): trigger client events
# Single event:
response.headers["HX-Trigger"] = "itemAdded"

# Multiple events with data:
response.headers["HX-Trigger"] = '{"itemAdded": {"id": 5}, "showToast": {"message": "Saved!"}}'
```

---

## 8. Extensions

Install extensions separately. For htmx 2.x, use the `htmx-ext-*` packages.

### json-enc

Encodes request body as JSON instead of form-urlencoded.

```html
<script src="https://unpkg.com/htmx-ext-json-enc@2.0.3/json-enc.js"></script>

<form hx-post="/api/items" hx-ext="json-enc">
  <input name="title" value="New Item">
  <button type="submit">Create</button>
</form>
<!-- Sends: {"title": "New Item"} with Content-Type: application/json -->
```

### response-targets

Swap into different targets based on HTTP status code.

```html
<script src="https://unpkg.com/htmx-ext-response-targets@2.0.3/response-targets.js"></script>

<div hx-ext="response-targets">
  <form hx-post="/submit"
        hx-target="#success-div"
        hx-target-422="#form-errors"
        hx-target-5*="#server-error">
    ...
  </form>
</div>
```

### head-support

Merges `<head>` tag content (styles, meta, title) from htmx responses.

```html
<script src="https://unpkg.com/htmx-ext-head-support@2.0.3/head-support.js"></script>
<body hx-ext="head-support">...</body>
<!-- Response <head> elements are merged into the page <head> -->
```

### preload

Preloads linked content on mousedown/mouseover for near-instant navigation.

```html
<script src="https://unpkg.com/htmx-ext-preload@2.0.3/preload.js"></script>

<nav hx-ext="preload">
  <a href="/page1" preload="mousedown">Page 1</a>
  <a href="/page2" preload>Page 2</a>  <!-- default: mouseover -->
</nav>
```

### sse (Server-Sent Events)

```html
<script src="https://unpkg.com/htmx-ext-sse@2.2.3/sse.js"></script>

<div hx-ext="sse" sse-connect="/events" sse-swap="message">
  <!-- Content replaced on each SSE "message" event -->
</div>

<!-- Named events -->
<div hx-ext="sse" sse-connect="/events">
  <div sse-swap="notification">Waiting for notifications...</div>
  <div sse-swap="update">Waiting for updates...</div>
</div>
```

### ws (WebSockets)

```html
<script src="https://unpkg.com/htmx-ext-ws@2.0.3/ws.js"></script>

<div hx-ext="ws" ws-connect="/chat">
  <div id="messages"></div>
  <form ws-send>
    <input name="message">
    <button type="submit">Send</button>
  </form>
</div>
<!-- Server sends HTML with id="messages" to swap content -->
```

### idiomorph

DOM morphing instead of swapping (preserves state, focus, scroll):

```html
<script src="https://unpkg.com/idiomorph@0.7.0/dist/idiomorph-ext.min.js"></script>

<div hx-get="/content" hx-swap="morph" hx-ext="idiomorph">
  <!-- Content is morphed, preserving DOM state -->
</div>
```

---

## 9. Common Patterns

### Active Search

```html
<input type="search" name="q" placeholder="Search..."
       hx-post="/search"
       hx-trigger="input changed delay:500ms, keyup[key=='Enter'], load"
       hx-target="#results"
       hx-indicator="#search-spinner">

<span id="search-spinner" class="htmx-indicator">Searching...</span>
<table><tbody id="results"></tbody></table>
```

### Infinite Scroll

```html
<table>
  <tbody id="items">
    <tr>...</tr>
    <!-- Last row triggers loading next page -->
    <tr hx-get="/items?page=2"
        hx-trigger="revealed"
        hx-swap="afterend"
        hx-indicator="#load-indicator">
      <td>Last visible item</td>
    </tr>
  </tbody>
</table>
<img id="load-indicator" class="htmx-indicator" src="/spinner.svg">
```

Server returns more `<tr>` elements; the final one includes `hx-get` for the next page.

### Click-to-Edit

```html
<!-- Display mode -->
<div hx-target="this" hx-swap="outerHTML">
  <p><b>Name:</b> Joe Blow</p>
  <p><b>Email:</b> joe@blow.com</p>
  <button hx-get="/contact/1/edit" class="btn">Click To Edit</button>
</div>

<!-- Server returns edit form (on GET /contact/1/edit) -->
<form hx-put="/contact/1" hx-target="this" hx-swap="outerHTML">
  <input name="name" value="Joe Blow">
  <input name="email" value="joe@blow.com">
  <button type="submit">Save</button>
  <button hx-get="/contact/1">Cancel</button>
</form>
```

### Bulk Update

```html
<form hx-post="/users/bulk" hx-swap="innerHTML settle:3s" hx-target="#toast">
  <table>
    <tbody>
      <tr>
        <td>Joe</td>
        <td><input type="checkbox" name="active:joe@smith.org"></td>
      </tr>
      <tr>
        <td>Jane</td>
        <td><input type="checkbox" name="active:jane@doe.org" checked></td>
      </tr>
    </tbody>
  </table>
  <button type="submit">Bulk Update</button>
  <output id="toast"></output>
</form>
```

### Lazy Loading

```html
<div hx-get="/chart-data" hx-trigger="load">
  <img class="htmx-indicator" src="/spinner.svg" alt="Loading...">
</div>

<style>
  .htmx-settling img { opacity: 0; }
  img { transition: opacity 300ms ease-in; }
</style>
```

### Modal Dialog (Bootstrap)

```html
<button hx-get="/modal/confirm"
        hx-target="#modals-here"
        hx-trigger="click"
        data-bs-toggle="modal"
        data-bs-target="#modals-here">
  Open Modal
</button>

<div id="modals-here" class="modal modal-blur fade"
     style="display:none" tabindex="-1">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content"></div>
  </div>
</div>

<!-- Server response -->
<div class="modal-dialog modal-dialog-centered">
  <div class="modal-content">
    <div class="modal-header"><h5>Confirm</h5></div>
    <div class="modal-body"><p>Are you sure?</p></div>
    <div class="modal-footer">
      <button data-bs-dismiss="modal">Cancel</button>
      <button hx-delete="/item/5" hx-target="#item-5" hx-swap="outerHTML"
              data-bs-dismiss="modal">Delete</button>
    </div>
  </div>
</div>
```

### Tabs (HATEOAS)

```html
<div id="tabs" hx-get="/tab1" hx-trigger="load" hx-target="#tabs" hx-swap="innerHTML"></div>

<!-- Server returns complete tab bar + content -->
<div class="tab-list" role="tablist">
  <button hx-get="/tab1" class="selected" role="tab" aria-selected="true">Tab 1</button>
  <button hx-get="/tab2" role="tab" aria-selected="false">Tab 2</button>
  <button hx-get="/tab3" role="tab" aria-selected="false">Tab 3</button>
</div>
<div id="tab-content" role="tabpanel">
  Tab 1 content here.
</div>
```

### Progress Bar

```html
<!-- Start button -->
<div hx-target="this" hx-swap="outerHTML">
  <button hx-post="/jobs/start">Start Job</button>
</div>

<!-- Server returns polling progress bar -->
<div hx-trigger="done" hx-get="/jobs/1" hx-swap="outerHTML" hx-target="this">
  <h3>Running</h3>
  <div hx-get="/jobs/1/progress" hx-trigger="every 600ms" hx-target="this" hx-swap="innerHTML">
    <div class="progress">
      <div class="progress-bar" style="width:0%"></div>
    </div>
  </div>
</div>

<!-- When done, server triggers "done" event via HX-Trigger header -->
```

### Inline Validation

```html
<form hx-post="/register">
  <input name="email" type="email"
         hx-post="/validate/email"
         hx-trigger="change"
         hx-target="next .error">
  <span class="error"></span>

  <button type="submit">Register</button>
</form>
```

### Delete Row with Animation

```html
<tr id="row-5">
  <td>Item 5</td>
  <td>
    <button hx-delete="/items/5"
            hx-target="closest tr"
            hx-swap="outerHTML swap:500ms"
            hx-confirm="Delete this item?">
      Delete
    </button>
  </td>
</tr>

<style>
  tr.htmx-swapping { opacity: 0; transition: opacity 500ms ease-out; }
</style>
```

---

## 10. Integration with Rust

### Axum + htmx

**Cargo.toml:**
```toml
[dependencies]
axum = "0.8"
axum-htmx = { version = "0.8", features = ["auto-vary", "guards", "serde"] }
askama = "0.13"
askama_axum = "0.5"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
tower-http = { version = "0.6", features = ["fs"] }
```

**Full/Partial rendering based on HX-Request:**
```rust
use axum::{response::Html, routing::get, Router};
use axum_htmx::{HxBoosted, HxRequest};
use askama::Template;

#[derive(Template)]
#[template(path = "index.html")]
struct IndexPage { items: Vec<String> }

#[derive(Template)]
#[template(path = "items_fragment.html")]
struct ItemsFragment { items: Vec<String> }

async fn get_items(HxRequest(is_htmx): HxRequest) -> Html<String> {
    let items = vec!["Alpha".into(), "Beta".into(), "Gamma".into()];
    if is_htmx {
        let t = ItemsFragment { items };
        Html(t.render().unwrap())
    } else {
        let t = IndexPage { items };
        Html(t.render().unwrap())
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/items", get(get_items));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Askama template — `templates/index.html`:**
```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.8/dist/htmx.min.js"></script>
</head>
<body>
  <h1>Items</h1>
  <div id="item-list">
    {% include "items_fragment.html" %}
  </div>
  <button hx-get="/items" hx-target="#item-list">Refresh</button>
</body>
</html>
```

**Askama template — `templates/items_fragment.html`:**
```html
<ul>
{% for item in items %}
  <li>{{ item }}</li>
{% endfor %}
</ul>
```

**Server-triggered events:**
```rust
use axum_htmx::HxResponseTrigger;

async fn create_item() -> (HxResponseTrigger, Html<String>) {
    // ... create item in DB ...
    (
        HxResponseTrigger::normal(["itemCreated", "showToast"]),
        Html("<li>New Item</li>".into()),
    )
}
```

**Request guard (reject non-htmx requests):**
```rust
use axum_htmx::HxRequestGuardLayer;

fn htmx_only_routes() -> Router {
    Router::new()
        .route("/fragments/sidebar", get(sidebar_fragment))
        .layer(HxRequestGuardLayer::default()) // redirects non-htmx to /
}
```

**Boosted link handling:**
```rust
async fn page(HxBoosted(boosted): HxBoosted) -> Html<String> {
    if boosted {
        Html("<main>Page content only</main>".into())
    } else {
        Html("<!DOCTYPE html><html>..full page..</html>".into())
    }
}
```

**Response header overrides:**
```rust
use axum_htmx::{HxReswap, HxRetarget, SwapOption};
use axum::response::IntoResponse;

async fn smart_response() -> impl IntoResponse {
    (
        HxRetarget("#error-container".to_string()),
        HxReswap(SwapOption::InnerHtml),
        Html("<p class='error'>Validation failed</p>".into()),
    )
}
```

### Actix-web + htmx

```rust
use actix_web::{get, web, App, HttpRequest, HttpResponse, HttpServer};
use askama::Template;

#[derive(Template)]
#[template(path = "search_results.html")]
struct SearchResults { results: Vec<String> }

#[get("/search")]
async fn search(req: HttpRequest, query: web::Query<SearchQuery>) -> HttpResponse {
    let is_htmx = req.headers().get("HX-Request").is_some();
    let results = do_search(&query.q);

    if is_htmx {
        let fragment = SearchResults { results };
        HttpResponse::Ok()
            .content_type("text/html")
            .body(fragment.render().unwrap())
    } else {
        // Return full page
        HttpResponse::Ok()
            .content_type("text/html")
            .body(full_page_with_results(results))
    }
}
```

**Key principle:** Always add `Vary: HX-Request` when serving different content based on `HX-Request` header (for caching correctness). The `axum-htmx` crate handles this automatically with the `auto-vary` feature flag.

---

## Quick Reference: Configuration Options

```html
<meta name="htmx-config" content='{
  "historyEnabled": true,
  "historyCacheSize": 10,
  "defaultSwapStyle": "innerHTML",
  "defaultSettleDelay": 20,
  "defaultSwapDelay": 0,
  "includeIndicatorStyles": true,
  "timeout": 0,
  "withCredentials": false,
  "selfRequestsOnly": true,
  "allowEval": true,
  "allowScriptTags": true,
  "scrollBehavior": "instant",
  "globalViewTransitions": false,
  "methodsThatUseUrlParams": ["get","delete"],
  "triggerSpecsCache": null
}'>
```
