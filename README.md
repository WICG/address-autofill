# Autofill Event

Autofill is a key feature of the web that reduces friction for millions of users everyday. It is widely used on login screens, e-commerce, and contact forms, to name a few examples. For commerce and checkout flows in particular we (Shopify) observe significant benefit from autofill to both the buyer experience and merchant outcomes:

> 1/3rd of guest checkouts (where the user sees an empty form) use autofill, and sessions with autofill see up to 41% increase in checkout completion rate. — Shopify (09/24)

At the same time, autofill on the web has many deficiencies: partial or incomplete fill, cross-browser interop issues, and high-cost to implement and maintain for developers. 

One key example is address autofill which, when implemented correctly, is a dynamic form: different geographies have different shapes and requirements for address input. Selecting a country requires changing the form (re-ordering fields, adding and removing fields) and depends on input from the user, but autocomplete mediates this interaction and may not respond correctly to the rendered form — some browsers account for this with various degrees of success, others don’t. 

The ‘industry standard’ solution for this requires use of hidden form fields that try to anticipate and capture the right information and then surface it to the user. This solution is brittle and complex. Worse, it entrenches use of hidden fields to power legitimate use cases, but the same technique can be and is also often abused by bad actors.

We think we can improve autofill and it’s value on all fronts:
Users: make it more reliable and pave the path for potential deprecation of filling of hidden fields
Site developers: standardize API and behavior to simplify implementation for site owners
Autofill providers: give 3P autofill providers hooks that work seamlessly with new API


# Proposal

## 1. Programmatic refills

At a high level, when the user autofills a set of fields in a page:

1. Before filling, Autofill will fire a new DOM event of type `autofill`, to every document involved in the fill (see below).
2. Each event's target will be the frame document that contains the fields that will be filled.
3. The event handler will be able to _request a re-fill_ within a browser-specific time window (and subject to other browser constraints).
   The browser will either:
   
   * resolve the promise after extracting and refilling the form, or
   * reject the promise.

This allows the website to dynamically change its DOM based on the original fill happening, without blocking the initial fill.
It also allows the browser to control the re-fill process to meet user expectations and security requirements, whilst also keeping the website appraised on how that turns out.

### JavaScript interface

In TypeScript, the interface might be roughly equivalent to the following:

```typescript
interface DocumentEventMap {
  'autofill': AutofillEvent;
}

interface AutofillEvent extends Event {
  readonly values:         ReadonlyArray<readonly [HTMLElement, string | boolean]>;
  readonly triggerElement: HTMLElement | null;
  readonly refill:         () => Promise<void> | null;
}
```

The event is fired before the autofill is carried out – in particular, before any `focus`, `change`, `blur`, or other events that may be fired by autofilling a field.
That is, the event signals the user agent's *intention* to autofill certain values into certain fields.

The event's target is the `document` because the Autofill implementation's notion of "form" may only loosely follow the `HTMLFormElement` association; it might for example span shadow and light DOMs or even multiple documents.
The event does not bubble and is not cancelable.

The event's `values` property maps each element to value to be autofilled.
If the element is a checkbox or radio button, the value is a `boolean`; otherwise it is a `string`.
It is not guaranteed that the element actually has that value after the autofill.
Examples where the value may differ after the autofill include:
an event handler changes or removes the element;
the element is an `<input maxlength=123>` and the value exceeds the maximum length;
the element is an `<input type=color>` and the value is not a CSS color;
the element is a `<select>` and none of its `<option>` matches the given value.

The event's `triggerElement` is usually the element from which the user triggered the Autofill event.
It is `null` if the autofill was not triggered from any `HTMLElement` in that `document`.

The element types are `HTMLElement` instead of `HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement ...` because of contenteditable and form-associated custom elements.

The event's `refill()` function initiates a refill on its first call.
The promise is resolved after the refill has happened, and rejected otherwise.
For example, an Autofill implementation may decline a refill because the form changed substantially.
Subsequent `refill()` calls return a rejected promise.

The event's `refill` is `null` if refills are not supported.
That may be the case if the event is already triggered by a refill or if the browser does not support refills for the fill data.
For example, an Autofill implementation might not support refills only for addresses but not for banking details.

### Example code

```html
<input autocomplete=street-address>
<!-- Maybe other fields. -->
<input autocomplete=country>

<script>
  document.addEventListener('autofill', async function(e) {
    // Real code would walk the values to find the right HTMLElement.
    if (e.refill !== null && e.values[2][1] === 'US') {
      const state = document.createElement('select');
      state.autocomplete = 'address-level1';
      state.innerHTML = `<option>California</option> ...`;
      document.appendChild(state);
      await e.refill();
    }
  });
</script>
```

### Open questions

* Should the browser help distinguishing `focus`, `change`, `blur`, and other events caused by Autofill from others?
* Should the effect of `AutofillEvent.refill()` be limited to the calling frame or can it also affect other frames?
* What is the effect if multiple (possibly cross-origin) frames call `AutofillEvent.refill()` 'simultaneously'? Can information be communicated this way?

##  2. "full-address"

We are also proposing a "full-address" `autocomplete` attribute value on the form, that would enable the browser to ask permissions for the user's full address, beyond the fields that are present in the current form.
