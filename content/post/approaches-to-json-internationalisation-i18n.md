+++
date = "23 Mar 2014"
title = "Approaches to JSON internationalisation (i18n)"
draft = false
url = "/posts/approaches-to-json-internationalisation-i18n"
tags = ["json", "api", "i18n"]
image = ""
comments = true
share = true
menu= ""
author = "Michael DeRazon"
aliases = [
    "/approaches-to-json-internationalisation-i18n"
]
+++

I needed to create an API that is capable of serving objects with strings in more than one language. Let me present the problem with an example.

Lets assume that we have a web app or a mobile client that wants to get information about coffee shops around the city. Also, lets assume also that we are going to have to support two languages, *English* and *Hebrew* (I know, right ?).

So one client will like to get the information in English:

```js
{
  "venue_id": "C001",
  "venue_location": "Baker St."
  "venue_name": "Aroma Coffee"
}
```
And another client will like to get the information in Hebrew:

```js
{
  "venue_id": "C001",
  "venue_location": "רחוב האופה"
  "venue_name": "קפה ארומה"
}
```

The two main questions i'm interested in are :

1. How do I store the data in the db ?
2. How will the API serve the data in the right language ?

Lets talk about both of them

### How to store the multilingual data

For the sake of the example, I will assume that you are using a document based database like Mongodb.

You can approach the problem in couple of ways:

#### Separate languages into different collections
Create a collection for each language and query the appropriate collection according to what the client accepts.
We will have `venues_en` collection and also a `venues_he` collection.

The obvious down side of this approach is that a lot of the data is being unnecessarily duplicated.

#### Embed the multilingual data in the same document
For each field that needs translation in a document (venue in our case), we will embed the multilingual data inside the field. For example :

```js
{
  "venue_id": "C001",
  "venue_location": {
    "en": "Baker St.",
    "he": "רחוב האופה"
  }
  "venue_name": {
    "en": "Aroma Coffee",
    "he": "קפה ארומה"
  }
}
```


This solves the duplication of fields from the previous approach but adds a little complexity to the object we're saving.

I prefer the second method because I think it's much more scalable. Imagine we need to support 25 different languages. Maintaining all of the duplicate data in the first approach becomes a nightmare. In the second approach we added a single level of hierarchy to the object, not a big deal usually.

### How will the API serve the data

Eventually, after we pull the relevant data that we want to save, we are left with a question of how to serve the data to the client. The easiest way for a bad API will be to just push the data "as is" to the client and let it worry about picking the language and parsing it to display to the user.

A better API should do the heavy lifting on the server side and will serve the client with normalised data in the language that it requested in the `Accept-Language` header.

## Easy implementation in Node.js
Here's a nice and clean way to strip the unnecessary languages from the response and serve only the relevant language to the client.

I am using a nice node module to [traverse](https://github.com/substack/js-traverse) the JSON object and strip it from other languages. Here's a simple example:

<script src="https://gist.github.com/mderazon/9729626.js"></script>
The output would be:

``` js
{
  venue_id: "C001",
  venue_location: "Baker St.",
  venue_name: "Aroma Coffee"
}
```

The `filter_language` function uses traverse module to go through the object tree and look for keys with the chosen `language`. If it finds one, it updates the parent's value to the chosen language value.

This way, the client doesn't have to worry about choosing the right language and can focus on displaying the data.

You can pass the function any json object you want as long as it's a valid json.

## Performance
Just to give a rough estimation, running 1 million entries of the above sample object through the `filter_language` function took 29 seconds on my MBA - throughput of 34.4 objects / ms.
Running on a much more complex object with around 100 keys and nested objects yielded throughput of 0.6 objects / ms. So if your data contains around 1K complex objects, the overhead of this  will be around 500 ms.
