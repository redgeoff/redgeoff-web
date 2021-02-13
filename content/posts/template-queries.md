+++
title = "What are MSON Template Queries?"
description = "Template Queries are dynamic templates constructed with MongoDB-style operators. They allow you to customize MSON components with less code and are very extensible."
images = ["/posts/template-queries/template-queries.png"]
date = 2021-02-08
draft = false
categories = [
  "Programming",
]
series = ["mson"]
tags = [
  "javascript",
  "json",
  "programming",
  "mongodb"
]
+++

{{< img src="/posts/template-queries/template-queries.png" alt="Template Queries" >}}

Template Queries are dynamic templates constructed with MongoDB-style operators. They allow you to customize MSON components with less code and are very extensible.

### OK, but what the heck is MSON?

[MSON](https://github.com/redgeoff/mson) is a low-code way to create apps from JSON. The ultimate goal of [MSON](https://github.com/redgeoff/mson) is to allow anyone to develop software visually. You can also use pieces of [MSON](https://github.com/redgeoff/mson) to turbo charge your existing apps.

A simple form component used to collect a name and email address could look like:

```js
{
  name: 'MyForm',
  component: 'Form',
  fields: [
    { name: 'name', component: 'TextField', label: 'Name' },
    { name: 'email', component: 'EmailField', label: 'Email' }
  ]
}
```

You can read more about MSON's language principles [here](https://redgeoff.com/posts/everyone-can-make-software).

### What are MongoDB operators?

MongoDB uses an extensive query language that is written in JSON. Because this language is so powerful, a number of projects, that work independently from MongoDB, have adopted support for the query language:
1. In-memory libraries for JS, like [mingo](https://github.com/kofrasa/mingo) and [sift](https://github.com/crcn/sift.js/)
1. Operators in [sequelize ORM](https://sequelize.org/master/manual/model-querying-basics.html)
1. Other databases, like [CouchDB](https://stackoverflow.com/questions/45976416/couchdb-mango-queries-couchdb-2-0-1)

This query language also contains a construct for [pipeline aggregations](https://docs.mongodb.com/manual/reference/operator/aggregation/), which can be used to execute operations like `$add`, `$multiply`, `$filter`, `$map`, etc... For example:
```
{
  $add: [ 1, 2 ]
}
```

### A basic MSON Template Query

Now that you have that background, let's take a look at an example where we extract the year from a user-provided date. Upon selecting a date with the date picker, you'll see that `Year` field is populated:

{{< codesandbox id="mson-template-query-6zb3b" >}}

Let's step through this. First, we construct a form with `date` and `year` fields:
```js
{
  component: "Form",
  fields: [
    {
      name: "date",
      component: "DateField",
      label: "Date"
    },
    {
      name: "year",
      component: "IntegerField",
      label: "Year"
    }
  ]
}
```

Second, we listen for changes to the `date` value, extract the year and then set the value of the `year` field:

```js
{
  component: "Form",
  fields: ...,
  listeners: [
    {
      event: "fields.date.value",
      actions: [
        {
          component: "Set",
          name: "fields.year.value",
          value: {
            $year: {
              $toDate: "{{fields.date.value}}"
            }
          }
        }
      ]
    }
  ]
}
```

The `$toDate` operator is used to convert the date value, which is in milliseconds, to a date object. The `$year` operator is used to extract the year portion from the date. The `Set` action is native to MSON and is used to set the value of the `year` field.

Pretty cool? Let's go a bit deeper...

### How can we make this reusable?

You can create custom actions, which can then be reused. Let's assume that we want to create an `app.GetYear` action that retrieves the year from a DateField value:
```js
compiler.registerComponent("app.GetYear", {
  // Extend Set, a core MSON component, that sets a
  // property on another component
  component: "Set",

  // Define the inputs to the custom action
  schema: {
    component: "Form",
    fields: [
      {
        name: "date",
        component: "DateField",
        required: true
      }
    ]
  },

  // Omitting the "name" here allows the value
  // to be passed to the next action via
  // "{{arguments}}"
  //
  // name: "",

  // Use the template query to extract the year
  // from the date
  value: {
    $year: {
      $toDate: "{{date}}"
    }
  }
});
```

We can then use this custom action in a chain of actions:

```js
const form = compiler.newComponent({
  component: "Form",
  fields: ...,
  listeners: [
    {
      event: "fields.date.value",
      actions: [
        // Extract the year and pass it to the next action
        {
          component: "app.GetYear",
          date: "{{fields.date.value}}"
        },

        // Use the extracted year to set the year field
        {
          component: "Set",
          name: "fields.year.value",

          // {{arguments}} is the output of app.GetYear
          value: "{{arguments}}"
        }
      ]
    }
  ]
});
```

Here is the completed example:

{{< codesandbox id="mson-custom-action-cziwj" >}}

### Another example: a conditional survey question

Let's assume that we have a survey question and we want the user to be able to enter a free response when they select `Other`:

{{< codesandbox id="mson-dynamic-survey-question-713s8" >}}

We define the form and hide the `covidOther` text field by default:

```js
{
  component: "Form",
  fields: [
    {
      name: "covid",
      component: "SelectField",
      label: "My government's response to COVID-19 was...",
      fullWidth: true,
      options: [
        { label: "Good", value: "good" },
        { label: "Bad", value: "bad" },
        { label: "Other", value: "other" }
      ]
    },
    {
      name: "covidOther",
      component: "TextField",
      label: "Please explain",
      multiline: true,
      hidden: true
    }
  ]
}
```

If the user selects `Other`, we toggle the `hidden` boolean property of the `covidOther` field. The `$ne` operator is shorthand for _not equal_.

```js
listeners: [
  {
    event: "fields.covid.value",
    actions: [
      {
        component: "Set",
        name: "fields.covidOther.hidden",
        value: {
          $ne: ["{{fields.covid.value}}", "other"]
        }
      }
    ]
  }
]
```

### Why doesn't MSON support custom JS in templates?

By now, you can probably see the power of Template Queries, but you may be wondering why MSON doesn't support custom JavaScript. To support custom JS in template parameters, it would require breaking two of MSON's core design principles:

1. __Compilation by instantiation__ - Components are _compiled_ into JS objects by simply instantiating a JS object and setting the props dynamically. This method of _compilation_ allows us to avoid a transpilation step and makes it much easier to dynamically modify components.

2. __Serialization & deserialization without `eval()`__ - Components can be serialized using `JSON.stringfy()` and stored practically anywhere. Moreover, components can be deserialized by dynamically compiling the result of `JSON.parse()`. As a result, raw JS (in a string), including JS template literals, are not supported as deserializing such JS would require the use of `eval()` or `new Function()`, which would expose a [XSS vulnerability](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#never_use_eval!) and add significant performance issues.

You can read more about this reasoning [here](https://github.com/redgeoff/mson/blob/master/DESIGN.md#why-doesnt-mson-support-custom-js-in-template-parameters).

### Wrap It Up!

Template Queries add a ton of capability to [MSON](https://github.com/redgeoff/mson) by providing another powerful way of customizing your [MSON](https://github.com/redgeoff/mson) components. And, they can be particularly useful in minimizing the code it takes to conditionally chain a series of actions.

If you liked this post, please give it a like or two. Happy building!

{{< disqus >}}