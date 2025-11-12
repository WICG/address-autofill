# Autofill Improvements

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

## 1. Javascript handler for autofill
In certain situations, the form triggering autofill needs to be adapted based on the autofilled values, or the website needs to perform other custom logic before values can be autofilled. Today, since autofill does not trigger any JS events, websites cannot react to such situations.


We are proposing to introduce an event handler on the form element for autofill events to allow receiving autofilled values:
```javascript
<script>
const autofillhandler = event => {
  event.refill(async () => {
    await updateFormAsync(event.autofill_values());
  }
};
document.addEventListener("autofill", autofillhandler);
</script>

```


The event object passed to the handler has two unique properties:
* An `autofill_values()` method that returns an object with key/value pairs that represent the autofilled names and values. Developers can use that to update their forms to be able to accept all the relevant values (e.g. based on the autofilled country).
* A `refill()` method where the developer can pass in a Promise. When that method is called, the current autofill pass gets delayed and will be retriggered when the promise is resolved.

After the user agent autofills, the form’s onautofill handler is called. The developer can use the `autofill_values()` call to know what data is about to be filled.
At this point they can perform their own validations and and modify the form to accomodate the data, an operation that may be async. (e.g. require a fetch, or done through virtual DOM changes)

The form modification function is passed to the `refill()` method. When the promise the form modification function returns is resolved, that tells the browser the modifications are done and that it should retrigger autofill on the form.

An important aspect to note is that the JS handler is not supposed to actually perform the autofill operation. This is left to the browser after the event handler has completed its work and the relevant Promise is resolved, to ensure the browser is able to indicate what values have actually been autofilled (e.g. providing a different background for autofilled values after the fill operation).

Exposing a Javascript handler also opens opportunities for alternative autofill providers to provide data to the site. For example, a password manager that contains one or several addresses can invoke the handler with the same structured data, ensuring that site-specific code and behavior is executed and applied.

##  2. "full-address"

We are also proposing a "full-address" `autocomplete` attribute value on the form, that would enable the browser to know to ask permissions for the user's full address, beyond the fields that are present in the current form.

##  3. Deprecate auto-filling of hidden fields
With the above steps in place we pave the way to change the default to stop autofill of hidden fields. This behavior was suggested in [W3C fork of the spec](https://github.com/w3c/html/blob/master/sections/semantics-forms.include#L10764-L10779) but has never made it into the official specification. Practically, such a process would likely require.. 

1. User preference: allow users to toggle behavior on/off, as a privacy enhancement.
2. Site Opt-in period: allow sites to signal and opt-out from hidden fields autofill 
3. Deprecation period: gradually change behavior to default-off.

# Open questions
1. Do we need `autofill_values()`? Maybe we can pass the relevant form(s) on the event object instead?
2. Browser extensions like password managers also offer autofill and for that they alter the DOM. How would they deal with this model?
