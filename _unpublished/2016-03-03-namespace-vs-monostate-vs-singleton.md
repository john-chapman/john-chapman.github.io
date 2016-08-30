---
layout: post
title: Namespaces vs Monostate vs Singleton
date: 2016-03-03
published: true
tags: c++, idioms, namespace, monostate, singleton
---

MONOSTATE
- Effectively a namespace
- Allows public,protected,private - although not useful for static data unless
  you're using inheritance.
- Forces you always to have only a single instance, which is why some say
  Singleton is better although it's really no different, as with Singleton you
  must always call the "GetInstance" function.