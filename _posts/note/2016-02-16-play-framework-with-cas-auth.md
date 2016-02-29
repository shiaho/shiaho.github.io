---
layout: post
title: "Play2.4.x with CAS"
author: ksh
description: ""
category: note
tags: [play,scala]
---
{% include JB/setup %}

## 依赖

在 `build.sbt` 中添加:

```scala
libraryDependencies ++= Seq(
  cache,
  "org.pac4j" % "play-pac4j-scala_2.11" % "2.0.1",
  "org.pac4j" % "pac4j-cas" % "1.8.5"
)
```

## 具体实施

这边使用 `kafka-magager` 这是一个play2.4.X 的项目，用它作为例子，展示Play2.4.X 加装 CAS