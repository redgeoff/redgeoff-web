+++
title = "How to autogenerate forms in React and Material-UI with MSON"
description = "Implementing great forms can be a real time-waster. With just a few lines of JSON, you can use MSON to generate forms that perform real-time validation and have a consistent layout. And, MSON comes with a bunch of cool stuff like date pickers, masked fields and field collections."
images = ["/posts/mson-react-material-ui-form/demo.png"]
date = 2018-10-22
draft = false
categories = [
  "Programming",
]
series = ["mson"]
tags = [
  "react",
  "forms",
  "programming",
  "tech",
  "design",
]
+++

{{< img src="/posts/mson-react-material-ui-form/demo.png" alt="MSON Demo" link="https://redgeoff.github.io/mson-react" >}}

Implementing great forms can be a real time-waster. With just a few lines of JSON, you can use [MSON](https://github.com/redgeoff/mson) to generate forms that perform real-time validation and have a consistent layout. And, [MSON](https://github.com/redgeoff/mson) comes with a bunch of cool stuff like date pickers, masked fields and field collections.

__Disclaimer__: this post is geared towards those wishing to use Material-UI with React. Future versions of MSON will support other rendering layers.

### What the heck is MSON?

[MSON](https://github.com/redgeoff/mson) is a declarative language that has the capabilities of an object-oriented language. The MSON compiler allows you to generate apps from JSON. The ultimate goal of MSON is to allow anyone to develop software visually. You can also use pieces of MSON to turbo charge your form development.

### A Basic Form Generated with MSON

Simply declare your form in JSON. Then let the MSON compiler and UI-rendering layer autogenerate your UI:

{{< codesandbox id="00ql22mx8w" >}}

Did you try submitting the form without filling in any values? Did you notice how real-time validation is automatically baked in?

Now, let’s take a closer look at what is happening. The first block of code contains a JSON definition that describes the fields in the form:

```js
const definition = {
  component: "Form",
  fields: [
    {
      name: "heading",
      component: "Text",
      text: "# Form using [MSON](https://github.com/redgeoff/mson)"
    },
    {
      name: "fullName",
      component: "PersonFullNameField",
      required: true
    },
    {
      name: "birthday",
      component: "DateField",
      label: "Birthday",
      required: true
    },
    {
      name: "phone",
      component: "PhoneField",
      label: "Phone"
    },
    {
      name: "submit",
      component: "ButtonField",
      label: "Submit",
      type: "submit",
      icon: "Send"
    }
  ]
};
```

This code adds the following fields to your form:

1. The _Text_ component displays some [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)
1. The _PersonNameField_ is used to capture a person’s first and last names
1. The _DateField_ allows a user to choose a date using a slick date picker
1. The _PhoneField_ uses an input mask and country codes to guide the user when entering a phone number
1. The _SubmitField_ contains a _Send_ icon and allows the user to submit the form via a click or by pressing enter

Now, let’s take a look at the code that renders the component and handles the submit event:

```js
ReactDOM.render(
  <Component
    definition={definition}
    onSubmit={({ component }) => {
      alert(JSON.stringify(component.getValues()));
    }}
  />,
  document.getElementById("root")
);
```

That’s it!? Yup! The [mson-react](https://github.com/redgeoff/mson-react) layer __automatically__ knows how to render the form component. It uses [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) and [Pure Components](https://reactjs.org/docs/react-api.html#reactpurecomponent) to keep the rendering up-to-date.

When there is no validation error and the user clicks the submit button, an event with the name equal to the button’s name is emitted. In our case, this event is called _submit_. Therefore, we can define a handler using the _onSubmit_ property. To keep things simple, we just alert the user of the entered values. Typically you want to do something like contact an API, redirect, etc…

### Basic Form 2

Now, let’s go a bit deeper into the CRUD with a different example:

{{< codesandbox id="n10q2y7ojj" >}}

The first thing you may notice are the [validators](https://github.com/redgeoff/mson#validators) in the definition:

```js
validators: [
  {
    where: {
      "fields.email.value": "nope@example.com"
    },
    error: {
      field: "email",
      error: "must not be {{fields.email.value}}"
    }
  }
]
```

Each field has a default set of [validators](https://github.com/redgeoff/mson#validators), e.g. the _EmailField_ ensures that email addresses are in a valid format. You can also extend these validators for a particular field or even for an entire form. For example, you can prevent the user from entering _nope@example.com_.

Next, let’s take a look at the code that loads the initial values when the component is mounted:

```js
onMount={({ component }) => {
  // Load any initial data, e.g. from an API
  component.setValues({
    id: "abc123",
    firstName: "Bob",
    lastName: "Marley",
    email: "bob@example.com"
  });
}}
```

This code could just as easily be replaced by code that retrieves the values from some API asynchronously.

Finally, we use a more sophisticated event listener to handle the submit action. In a real-world app, you’d probably want to send a request to an API to save the data. You would receive a response from this API. If you receive an error, e.g. the email address is already in use, you can present this error to the user:

```js
onSubmit={({ component }) => {
  // TODO: Contact some API with the data
  console.log("submitting", component.getValues());

  // Simulate response from API saying that email address is already in use and report this
  // error to the user
  if (component.get("fields.email.value") === "taken@example.com") {
    component.set({ "fields.email.err": "already in use" });
  } else {
    // Everything was successful so redirect, show confirmation, etc...
  }
}}
```

### [Live Demo](https://redgeoff.com/mson-react/)

This post only touches on a small piece of what you can do using [MSON](https://github.com/redgeoff/mson). Because [MSON](https://github.com/redgeoff/mson) is a full-featured language, you can declare all types of cool components. If you are interested in seeing more live examples, check out the [live demo](https://redgeoff.com/mson-react/).

{{< img src="/posts/mson-react-material-ui-form/demo.png" alt="MSON Demo" link="https://redgeoff.github.io/mson-react" >}}

### Wrap It Up!

If you are using React and Material-UI, you can speed up your app development by autogenerating your forms from JSON. This can be particularly useful if you need to bootstrap an app quickly and don’t want to have to worry about building a UI from scratch.
If you liked this post, please give it a clap or two. Happy [autogenerating!](https://github.com/redgeoff/mson)

{{< disqus >}}