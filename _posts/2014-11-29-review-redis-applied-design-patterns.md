---
layout: post
title: "Book Review: Redis Applied Design Patterns"
date: 2014-11-29 10:00:00
disqus: true
disqus_id: review-redis-applied-design-patterns
---

I’m a [Redis](http://redis.io) fan. It is a nice piece of software with a million use cases. It is written in a modern and worth studying C language, supported by a vibrant community and you can find tons of documentation and examples about it. And it’s developed by a cool [italian guy](https://twitter.com/antirez).

I’ve been reading this new book **Redis Applied Design Patterns** by [Arun Chinnachamy](http://bit.ly/1yPlraY). It is meant to be a very practical and small (100 pages) guide for developers to using Redis effectively by taking advantage of the uniqueness of its features.

<div style="float: left; margin: 15px" markdown="1">
![the book](/images/6713OS_Redis_Applied_Design_Patterns_Cover.jpg)
</div>

The first pages of the book briefly introduce the reader to the NoSql way of thinking, with some examples showing data denormalization in a key-value storage and explaining the need to carefully design your data structures based on how you want to access your data.
Then the author provides a good collection of real-life use cases for Redis: cache management is a must for a book on this topic, but the real focus is on the kind of things that are too slow or impossible to achieve with a relational database (features like autosuggest, leaderboards, notifications, real time analysis) and also more complex examples about using Redis as primary in-memory data store (for a commenting systems, an advertising network or your new shiny social network).

Each chapter, as the book itself, keeps short and focused on the discussed use case. It provides a brief context to the reader and describes a step-by-step solution. After relevant explanation and code examples, the author usually comments on the flexibility of a Redis-based solution and/or suggests how the system could be further improved to handle more complex requirements.

The book is short enough that could be read in a day, but such a collection of documented design pattern is good to keep at hand everyday for quick review. Each use case is self-contained, showing commands and explaining data structures with descriptions and links to documentation. The code examples are straightforward to read and understand.

What I didn’t like is that while the book seems to be written for an intermediate user (one who is already familiar with the basics of Redis and is able to use a client library), some of the patterns described here are probably [obvious](http://oldblog.antirez.com/post/take-advantage-of-redis-adding-it-to-your-stack.html) to this kind of reader. The author just provides very basic examples to start with, covering only the surface of the problems without in-depth discussion: when things start to get more interesting and not so obvious, you’re at the end of chapter and it feels like you didn’t have the opportunity to understand how more complex scenarios can be solved in real production applications.

This (first?) edition of the book suffers from several problems. For example, in some cases there are inconsistencies between the examples and the text, and the code uses different variable naming conventions. Clearly nothing that harms the understanding of the concepts but, in general, seems like the book has been written in a hurry and/or at different times: different chapters try to explain the same thing without self-reference (e.g. the ZREVRANGEBYSCORE command is explained twice in ch. 8 and in ch. 10). Some examples are outdated: the chapter on autocomplete could at least mention the [ZRANGEBYLEX](http://redis.io/commands/zrangebylex) command, while [Redis Sentinel](http://redis.io/topics/sentinel) is nowadays a stable solution for high-availability (no more “under development”). A book about Redis use cases is not complete without discussing the [Lua Scripting](http://redis.io/commands/eval) feature and how many problems it solves.

[Redis Applied Design Patterns](http://bit.ly/1yPlraY) is a short and maybe worth reading book for any non-advanced Redis user. It is a collection of the established, well-known practices: you may find it useful for a quick overview of the kind of problems Redis helps to solve, but this book doesn’t really add anything new to the amount of documentation and resources already available by the community.
