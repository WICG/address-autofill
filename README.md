# Address Autofill API

Autofill is a key feature of the web that reduces friction for millions of users everyday. It is widely used on login screens, e-commerce, and contact forms, to name a few examples.  \
At the same time, autofill on the web can be improved.

The API’s reliance on form fields and the `autocomplete` attribute effectively ties it to the site’s markup, making it hard for developers to build dynamic websites with flawless autofill experiences.

 

Autofilling addresses can be tricky when the form itself needs to change based on the autofilled values. The fields can change their order, new fields can be added or fields removed. 

As an example, think of an address form that may or may not include a “state” field, based on the selected country. The site may not want to burden the user with this form field that they may not need, but would want to add it if and when it is necessary, and would want the browser to autofill that field if that’s the case.

To get around these limitations of HTML-based autofill, developers often rely on hidden form fields that enable them to collect such optional information and then surface it to the user if needed.

While that kinda works, it is a source of interoperability issues (as different browsers tend to do different things when it comes to such hidden fields), and it often results in a lot of developer pain.

This document explores a proposal to improve the status quo.


# Decoupling autofill from HTML and the DOM

In the HTML spec, the [autocomplete attribute fulfills two separate roles](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#autofilling-form-controls%3A-the-autocomplete-attribute): the spec talks about the autofill expectation mantle vs. the anchor mantle. In short, we rely on input fields to represent both the information that users would expect the browser to fill, as well as the place in which the browser would fill it (and to an extent, the UI in which users interact with the autofill browser feature).

We think that it may make sense to create a separate path in which we can split apart those different roles.

As discussed above, in some cases, the information that we want from the browser’s autofill store can’t necessarily be mapped to a static set of input fields with autocomplete attributes.

Similarly, the information that developers want to get from the browser’s autofill may not be immediately mapped to the available input fields available in the DOM when autofill happens.

Wouldn’t it be neat if we decoupled the autofill functionality from HTML and the DOM, and enabled developers to perform both the “expectation” and the “anchor” roles directly?


# Straw proposals


## Form-free autofill

What if we had an explicit API that enabled developers to request exactly the type information they want autofilled, and then provided it to them as a JS object?

Such an API could take in a form field and an array of [autofill field names](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#autofill-field) and asynchronously return the information the developer wants.

That could look something like the following

```javascript
// API sketch: 
// An API that takes in an array of form elements and an array autofill field
// names and returns a Promise that resolves to an object with the field name as keys and 
// the user's autofill data as values.
// While the explicit call enables the developer to request autofill data, that
// data could be held up until the user interact with the page, or even
// interacts with the specific form the API is requesting data for.

async function userDetails() {
  let values;
  try {
    values = await navigator.autofill.trigger({
      fields:["name", "email", "full-address"]});
    // "full-address" is a new alias for all fields related to the address, 
    // to simplify differences in addresses between countries and regions.
  } catch {
    // The user declined to provide the data. Create a form for them to fill.
    createForm();
    return;
  }
  const validated_values = validate(values);
  updateAddress(validated_values);
};
```


### UI

In terms of UI, this API shape decouples autofill from a specific form, and hence won’t be amenable to the current UI used for autofill. 

It could have UI similar to the following:

![UI mock](https://github.com/user-attachments/assets/80aaa8ec-59e1-4cfa-84a6-9daec5dbb7a2)



That would enable further decoupling of autofill from forms, and enable totally new UX patterns to be used in combination with Autofill.

 


### New UX patterns

Requesting and handling of autofill in JS instead of the DOM would allow completely new user experiences. Instead of showing lengthy forms the app can prompt the user to share their data after an appropriate user interaction with the application and show the shared data in an optimized view. This would allow the website to build a much better experience.

For example, many online shop systems have a more compact checkout design for logged in users which can be used to show the autofilled data for any guest checkouts. 



### Autofill in iframes

Websites are often composed of isolated components that run in separate iframes. For those cases apps should be able to centrally handle autofill requests without handling the data. This is naturally more amenable in a Promise based API shape.

The top frame can request autofill and can specify which origin or which specific child frame can access the data. 

The user will only get asked once to share data with the app. Code running in an iframe will get access to data that’s allowed for its origin or for its iframe, even without the parent page handling the data. This will ensure that the top frame does not get into privacy or PCI compliance scope for the requested data.


### Alternatives considered

Instead of specifying the autofill requests for all iframes in one single autofill.trigger call we could also allow iframes to trigger it by introducing a new permission for autofill.trigger.

This would look like follows:

This would allow the JS code executing in the iframe to trigger a user prompt to autofill its data.

There are a couple of issues with that approach:



* As we’d like to notify users of all the information that would be filled by autofill, multiple calls for autofill information by the different frames would either compromise keeping users in the loop, or result in them seeing multiple prompts, reducing their ability for informed consent. 
* Different iframes are only allowed to get certain bits of information from autofill, but not all of it. The current “allow” attribute may not provide enough flexibility for that configuration. 

The downside of this approach is, that the autofill.trigger calls of different frames/origins are not coordinated which could lead to the user seeing multiple requests for autofilling data. For example the main frame asks about the name and address and the iframed credit card fields ask for credit card details.


## Form-bound autofill

While the above is an approach that evolves autofill in a totally new direction there are also ways in which we can improve the current autofill implementation.

The API shape for that could take that of an event listener, where the options are provided as a separate attribute on &lt;form>. E.g. 

This option would keep the current UI, and would still give developers significantly higher control of the values filled, enabling them to dynamically modify the form to match the data, or even animate the form away once the data is filled.


### Iframes

In this API shape, we’d need some [control mechanism](https://github.com/w3ctag/design-reviews/issues/831) to declare the eligibility of forms in different iframes to be autofilled. We’d also need the handlers in each one of these frames to handle the specific bits of data it should handle, not unlike the situation today. 


# Isn’t this proposal the same as..


## `[requestAutocomplete](https://web.dev/articles/requestautocomplete)`



Not really.

There are a few fundamental differences between the above proposals and requestAutocomplete.

`requestAutocomplete` solved a fundamentally different problem - Filling in forms when the user interacted with other parts of the page, rather than on the form itself.

The problem we’re trying to solve here is the dynamic nature of the data filled in, and the fact that it can’t be reflected in static DOM elements before we know what the data is.

Because requestAutocomplete solved a different problem, it never decoupled autofill from a specific form, and therefore didn’t enable gathering information beyond what’s already represented in the DOM (which is a significant difference when it comes to addresses).

Because of the reliance on forms, it implicitly relies on hidden forms, and never included the user in the loop of what data is being filled in.

Finally, it seems like the main reason for its [removal](https://groups.google.com/a/chromium.org/g/blink-dev/c/O9_XnDQh3Yk?pli=1) was lack of adoption. While I don’t have context on the details there, that could be related to the fact it solved a problem that merchants didn’t necessarily struggle with, and one where the regular browser autofill (when users focus on a form) was a good enough solution for. 


## [Payment Request API](https://web.dev/articles/how-payment-request-api-works) / [Payment Handler API](https://web.dev/articles/web-based-payment-apps-overview)?

Nope.

The Payment Handler API allows you to integrate your own payment app within a browser-driven checkout flow.

While Web Payments, including the Payment Handler API and Payment Request API, streamline the creation of straightforward commerce flows using standardized tools, they come with a significant drawback: they strip merchants of control over their checkout processes. For many merchants, the checkout page is prime real estate, and they are often very particular about maintaining full control to craft a custom flow that leverages the full capabilities of the web. Complex use-cases such as split shipping (multiple packages), split delivery (shipping and local pickup), and custom tax engines are often beyond the scope of what the Payment Request API can handle.

This proposal aims to provide the necessary flexibility without imposing such constraints, thereby allowing merchants to retain the control they need to address their unique requirements.


## [ContactPicker API](https://www.google.com/url?q=https://developer.mozilla.org/en-US/docs/Web/API/Contact_Picker_API&sa=D&source=docs&ust=1724777682236540&usg=AOvVaw27YnejfCwPWROVgnMGnHBs)?

The Contact Picker API allows you to prompt the user to select a contact from their phone and share the information with the requesting website. 

Key differences between Autofill JS API and ContactPicker API are



* They use different storage of data. When you autofill a form you usually refer to your own data. In your phone contacts you have 100s of contacts that are usually never relevant in the context of autofilling.
* Contact Picker API does not support Payment details (and it doesn't make sense for contacts)
* Contact picker [seems to only be available on mobile](https://developer.mozilla.org/en-US/docs/Web/API/Contact_Picker_API#browser_compatibility). Autofill data would be available on Desktop and Mobile contexts.
