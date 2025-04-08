---
title: "Parsing JSON data from PICK Basic"
categories:
  - PICK Basic
tags:
  - json
  - integration
  - tutorial
---

# Parsing JSON Data in PICK Basic: An Effective Approach

![JSON Hero](/assets/images/JSON_Hero.png)
<a href="https://github.com/ianmcgowan/SIMBIAN.BP/tree/main">Link to the code on GitHub</a>

## Introduction

JSON (JavaScript Object Notation) has become standard for exchanging data between web services, APIs, and configuration files. It
replaced XML in most modern applications because it's simpler, more readable, and easier for developers to use. This shift
highlights a key lesson: developer productivity often beats "better" technology.

A quick example makes the difference clear. Here's data represented in JSON:

```json
{
  "Countries": [
    {
      "Name": "USA",
      "Capital": "Washington, D.C.",
      "Population": 331000000,
      "Languages": ["English", "Spanish"]
    },
    {
      "Name": "Canada",
      "Capital": "Ottawa",
      "Population": 38000000,
      "Languages": ["English", "French"]
    }
  ]
}
```

and the equivalent XML:

```XML
<Countries>
  <Country>
    <Name>USA</Name>
    <Capital>Washington, D.C.</Capital>
    <Population>331000000</Population>
    <Languages>
      <Language>English</Language>
      <Language>Spanish</Language>
    </Languages>
  </Country>
  <Country>
    <Name>Canada</Name>
    <Capital>Ottawa</Capital>
    <Population>38000000</Population>
    <Languages>
      <Language>English</Language>
      <Language>French</Language>
    </Languages>
  </Country>
</Countries>
```

JSON's structure — objects with key-value pairs and arrays — makes nested data straightforward. It lacks XML's richness and
extensibility, but the trade-off is worth it for simplicity.

Multivalued database systems like PICK store data as flat, delimited records — not hierarchies. Without native pointers or objects,
parsing JSON directly can be tricky. Some modern MV systems like jBASE and OpenQM offer built-in support for objects, but most
traditional PICK environments don't.

So how can PICK developers handle JSON effectively? In this post, I'll review some existing approaches, discuss the challenges, and
share a practical method that works efficiently in PICK Basic.

## Common JSON Parsing Techniques

In mainstream languages like JavaScript, Python, and Java, parsing JSON is straightforward. JavaScript handles JSON natively—just
use JSON.parse() to quickly convert text into an object. Python's json module does the heavy lifting, turning JSON strings into
dictionaries and lists with minimal effort. Java developers typically use libraries such as Jackson or Gson, which map JSON directly
into objects or classes. These languages offer convenient ways to easily navigate and manipulate JSON data.

When you're scripting or working from the command line, tools like [jq](https://jqlang.org/) make JSON parsing quick and painless.
`jq` lets you filter, slice, and transform JSON directly in shell scripts or command prompts, without needing extra libraries or
complicated parsing logic. It’s lightweight, powerful, and perfect when you just want to grab specific data or reshape JSON
structures on-the-fly.

## Challenges in Parsing JSON with PICK Basic

Without recursive data objects (e.g. dynamic arrays that contain other dynamic arrays), parsing deeply nested JSON in PICK Basic can
be challenging - there's nowhere to "put" the data.

Some existing PICK Basic solutions for parsing JSON show creative approaches to the problem:
 
* [International Spectrum Article](https://www.intl-spectrum.com/Article/r1031/Parsing_JSON_Data_with_jBASE_QM_and_U2)

* [Krowemoh's JSON.PARSE.RECURSE](https://github.com/Krowemoh/TCL-Utilities/blob/main/JSON.PARSE.RECURSE)

## A Different Approach to JSON Parsing in PICK Basic

Since the problem is really about how to store the data, we take a different approach. As a developer consuming the data, we will
typically be drilling down one layer at a time and doing some processing of each layer. So it's not strictly necessary to have the
entire parsed JSON structure in memory at once - we can return the first level and any values that are "complex" (objects or
arrays) we will defer processing those until they are needed.  For example, using the json in the introduction, if we need the names
of the countries and their capitals, we can do that with the following:

```basic
CALL JSON.GET.OBJECT(JSON.STRING, COUNTRY.LIST, ERR)
```

Which will return COUNTRY.LIST as a dynamic array with three attributes:
1. "Countries"
2. "ARRAY"
3. The JSON array of the list of countries "[ {Name: USA....}, {Name: Canada...} ]"

Then we can use the JSON.GET.ARRAY to get the array of countries:

```basic
CALL JSON.GET.ARRAY(COUNTRY.LIST<3,1>, COUNTRY.ARRAY, ERR)
```

Which will be a dynamic array (somewhat similar to the [JSON Lines](https://jsonlines.org/) format) with two "rows":

1. { "Name": "USA", "Capital": "Washington, D.C.", "Population": 331000000, "Languages": ["English", "Spanish"] },
2. { "Name": "Canada", "Capital": "Ottawa", "Population": 38000000, "Languages": ["English", "French"] }

Which can be processed in a loop:

```basic
FOR I = 1 TO DCOUNT(COUNTRY.ARRAY, @AM)
  CALL JSON.GET.OBJECT(COUNTRY.ARRAY<I>, COUNTRY.OBJECT, ERR)
  PRINT "Country: " : COUNTRY.OBJECT<3,1> ;* JSON objects are inserted in the same order as they are in the JSON, we know the index
  PRINT "Capital: " : COUNTRY.OBJECT<3,2> ;* Can also do a LOCATE "Capital" IN COUNTRY.OBJECT<1> SETTING IDX to get the index
NEXT I
```

This is not as efficient as using a native JSON parser, but it is simple and effective for most use cases.

## Applying Test-Driven Development (TDD)

One nice aspect of this approach is that it is easy to implement TDD or [Test Driven
Development](https://en.wikipedia.org/wiki/Test-driven_development) for the JSON parsing. TDD is something I've been wanting to try
in my development process for a while, but never found a good way to start - most of my development is data load-and-extracts or
CRUD, where it's hard to know ahead of time what the expected outcome is. But with JSON parsing, we can easily define a set of test
cases that have very clear and repeatable outcomes.

The `JSON.GET.TEST` script is a simple test harness that can be used to validate the JSON parsing. It uses the same JSON.GET
functions as the main JSON parsing code, but instead of returning the parsed data, it runs through a set of test cases and checks
the response against the expected outcome.  As bugs or edge cases are discovered, they can be added to the test cases first, then
run to show that the test is correctly identifying an issue.  Then the code can be modified to fix the issue, and the test can be
re-run to show that the issue is fixed.  This is a great way to ensure that the code is working as expected.

![Test run](/assets/images/JSON.GET.TEST.jpg)

![Test run](/assets/images/JSON.GET.TEST2.jpg)

## Conclusion

In this post, we explored a practical approach to parsing JSON data in PICK Basic. By deferring parsing of complex nested structures
until they are needed, we can simplify the parsing process and make it more efficient. We also discussed how to apply test-driven
development (TDD) to JSON parsing, which can help ensure that the code is working as expected and make it easier to identify and fix
bugs.
