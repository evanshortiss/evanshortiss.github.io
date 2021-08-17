---
published: true
title: Optional Environment Variables with Quarkus & MicroProfile Config
layout: post
categories: java quarkus
---

Today I was reminded that Java often makes the complex simple and the simple
complex. An alternate explanation, that's equally reasonable, is that I suck at
using search engines and RTFM'ing.

Take this `application.properties` for example:

```properties
security.header.name=${SECRET_HEADER_NAME:x-secret-header}
security.header.value=${SECRET_HEADER_VALUE}
```

I wanted my program that's using these properties to start even if the
`security.header.value` is missing from the environment. If you're a Java
developer you can probably guess where I'm going wrong here:

```java
@ConfigProperty(name = "security.header.value")
String secretValue;
```

Surely `secretValue` would be set to `null` if it's missing from the
`application.properties`, right?. In reality, this raises an error that states
`could not expand value SECRET_HEADER_VALUE in property security.header.value`.
Fair enough. Quarkus has a great [config reference](https://quarkus.io/guides/config-reference)
so I stumbled upon this solution fairly quickly, but it feels wrong:

```java
@ConfigProperty(name = "security.header.value", defaultValue = "")
String secretValue;
```

My ~~JavaScript~~ TypeScript-native brain doesn't accept a empty String as
interchangeable with `null` or `undefined`, and I was generally curious why
it was hard to find an answer. I tried using the `Optional<String>` solutions
that were posted too, but ultimately the issue seems tied to the fact that the
environment variable is undefined and any attempt to read it results in the
`could not expand value` error.

The solution was embarrassingly simple. Maybe that's why I didn't find it in my
search results. Simply add the `:` symbol after the environment variable, and
leave the default value empty:

```properties
security.header.value=${SECRET_HEADER_VALUE:}
```

Now, I can write code like so and not receive `could not expand value` errors:

```java
@ConfigProperty(name = "security.header.value")
String secretValue;

public void verifyHeader () {
  if (secretValue != null) {
    // C'mon, do something...
  } else {
    // Uh, you sure about this?
  }
}
```
