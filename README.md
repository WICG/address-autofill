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

## 1. Provide `full-address` as a new atomic address unit
When a user selects an address to autofill, their goal and expectation is that it is filled fully and accurately. If the form needs to be updated to match requirements of their geography, that should happen and the resulting form should contain accurate information. This behavior is excessively hard and brittle to implement correctly today.

To solve the problem of the dynamic nature of address formats depending on the country and region and the resulting dynamic forms (e.g. forms hide/show input fields depending on country or region selection), we propose a new `full-address` attribute that provides address as atomic unit:

```javascript
<form autocomplete="full-address" onautofill="autofillhandler">
...
</form>
```

`full-address` acts as an enhancement to existing behavior. Autofill can continue to use form input fields to preview and fill the fields when the user confirms their selection. Presence of full-address indicates that the browser should also provide a structured address object that the developer can process and use to implement own validation and fill behavior.

![](https://screenshot.click/26-11-29h70-lrrrj.png)
*Some browsers provide a preview state where selected address information is shown in the form but not yet visible to the page and script. This behavior is great and can remain as is.*

With `full-address` site owners can eliminate the need for and use of hidden input fields for address. 

## 2. Javascript handler for autofill
In the HTML spec, the [autocomplete attribute fulfills two separate roles](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#autofilling-form-controls%3A-the-autocomplete-attribute): the spec talks about the autofill expectation mantle vs. the anchor mantle. In short, we rely on input fields to represent both the information that users would expect the browser to fill, as well as the place in which the browser would fill it (and to an extent, the UI in which users interact with the autofill browser feature). We believe it should be also possible to split these apart, to allow site owners with custom logic to bring their own validation and fill behaviors.

We are proposing to introduce an event handler on the form element for autofill events to allow receiving autofilled values. This event would be the primary consumer of `full-address` object:
```javascript
<script>
const autofillhandler = event => {
  const validated_values = validate(event.autofill_values());
  updateAddress(validated_values);
  event.preventDefault(); // stop browser autofill in favor of custom validation+fill logic
};
</script>


<form onautofill="autofillhandler" autocomplete="full-address">...</form>
```

When the user commits to autofill, the form’s onautofill delegate is called with requested autocomplete data in structured format. At this point the developer can perform own validations and transforms on provided data, update the form, and fill it.

Exposing Javascript handler also opens opportunities for alternative autofill providers to provide data to the site. For example, a password manager that contains one or several addresses can invoke the handler with the same structured data, ensuring that site-specific code and behavior is executed and applied.

The Contact Picker API allows you to prompt the user to select a contact from their phone and share the information with the requesting website. 

##  3. Deprecate auto-filling of hidden fields
With the above steps in place we pave the way to change the default to stop autofill of hidden fields. This behavior was suggested in [W3C fork of the spec](https://github.com/w3c/html/blob/master/sections/semantics-forms.include#L10764-L10779) but has never made it into the official specification. Practically, such a process would likely require.. 

1. User preference: allow users to toggle behavior on/off, as a privacy enhancement.
2. Site Opt-in period: allow sites to signal and opt-out from hidden fields autofill 
3. Deprecation period: gradually change behavior to default-off.


# Alternatives Solutions
As discussed during TPAC 2024, an alternative to the above described flow could be to make Autofill a two-stage process. The JS handler is called when the country has been filled, giving the website a chance to update it's UX. In the JS handler the website indicates when the website has been updated which gives the browser a signal when to continue autofilling.

This has the advantage of not needing to decide on an address format for the JS handler because the address is still shared through the form fields and the browser knows which fields are autofilled.

Disadvantages of this approach are:
* We could end up in a multi-stage process where the address format depends on more than 1 field. E.g. country, region, rest
* The JS handler to receive the autofill values feels like a more modern API

# Open questions
1. Format of the provided data. In breakout a new ISO standard was mentioned that we need to investigate.
2. Browser extensions like password managers also offer autofill and for that they alter the DOM. 
    1. These browser extensions should provide the same behaviour like the browsers autofill. Do we need a browser API that let’s browser extensions trigger autofill with its own values?
    2. TODO: Nicholas Steele from 1password expressed interest in this
3. ...
