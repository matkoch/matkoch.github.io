---
title: YAML Hell in CI
tags:
- .NET
- Azure
- DevOps
- Kotlin
- NUKE
- TeamCity
hidden: true
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: assets/images/2019-12-12-replace-package-with-project-references.jpg
---

If you dive into the DevOps world, chances are high you meet YAML around the next corner. 

- Azure Pipelines
- Bitrise
- Travis
- AppVeyor
- CircleCI


https://twitter.com/onovotny/status/1207433967097520129
https://twitter.com/csharpfritz/status/1207434317179305986

What's wrong with YAML?
- error-prone
- not refactoring safe
- language quirks
- no symbols, no compile errors
- not imperative, cumbersome conditionals, no loops
- https://noyaml.com/

https://twitter.com/vizaster/status/1179692600883924992

While I see how YAML configuration can be attractive, I truly believe that this is only because the example pipelines are more or less sample code.

https://twitter.com/onovotny/status/1207435056316375042

https://twitter.com/icaromedeiros/status/1207602746041536513

Okay, you can use build tools like CAKE or FAKE. Then our YAML would become some kind of one-liner:

CODE

But at the same time, you'll immediately lose _actual_ CI features, most importantly the ability to run tasks in parallel on different agents, like tests.

https://twitter.com/matkoch87/status/1189614613106692096

which I only managed to find myself in some [legacy test code](https://github.com/microsoft/azure-pipelines-tasks/blob/746443be5cee275e65f33d5a9fdeefc183632d65/Tests-Legacy/L0/PublishCodeCoverageResults/_suite.ts#L193)

# Alternatives



## Kotlin DSL in TeamCity

## NUKE

TeamCity and Kotlin DSL are great. Yet, I'm a fan of being able to _have_ the freedom to change. which is why I started with NUKE as an open-source build system two years ago.

# Conclusion

Hopefully, I made some reasonable arguments. YAML is really not a great choice these days. Even if you spice it up with some funky UI, this is only a preliminary solution. Files will get bigger, and eventually unmaintainable. TeamCity with its Kotlin DSL is a huge step in the right direction. Configuration as _real_ code. You have code completion, refactorings, navigation, and extension points. While I love TeamCity as a CI system, I'm still not favoring it for build configuration. It has other strengths. However, NUKE is the perfect choice for implementing a build chain. You can easily switch to another CI system, and NUKE will happily generate the configurations files – in whatever format – for you. Using all features out-of-the-box!