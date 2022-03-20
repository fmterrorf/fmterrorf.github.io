---
title: "Reusable Forms With Ecto Embedded Schema"
date: 2022-03-19T19:34:09-06:00
draft: true
---

## Motivation

While working with forms in LiveView I ran into a situation where i want to re-use section of forms. Take the following forms for example:

* ![All fields](/reusable-forms/all-fields.png)

* ![Email Pref](/reusable-forms/email-pref.png)


Notice that the `Email Preferences` section is being used in two forms. 


## Enter Nested Embedded Schema

In order to create reusable forms, we will use Ecto's embedded schema. The plan is to create 2 embedded schema. 

1. Only for the Email Preferences section 
2. A schema that contains the `First Name`, `Last Name` and `Email Preferences` (we will reuse the schema we created on point 1)

