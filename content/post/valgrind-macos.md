---
title: "Valgrind and macOS Sierra"
date: 2016-11-06
keywords:
- C
- Valgrind
- Docker
- Dockerfile
- macOS
- Sierra
- macOS Sierra
---

Valgrind is a code profiling tool for Linux, it is wildly known for it's memory leak detection and debugging capabilities.

At the time of writing, Valgrind is still not compatible with macOS Sierra (10.12), you can read more about it 
[here](https://bugs.kde.org/show_bug.cgi?id=365327).

### Docker to the rescue

To get around this, I've built a small Docker Image based on phusion/baseimage, build essentials and Valgrind. 
A volume is mounted under **/root/build** for ease of use.

The Dockerfile and instructions are available through [GitHub](https://github.com/joaodlf/docker-valgrind).