---
title: Mortal and Immortal Symbols in Ruby
date: 2017-02-23
tags:
- Ruby
---
[Symbols](https://ruby-doc.org/core-2.2.0/Symbol.html) are one of the most powerful datatypes that Ruby has to offer. Compare
and lookup operations are faster when symbols are used rather than strings. You can check this [post](http://www.reactive.io/tips/2009/01/11/the-difference-between-ruby-symbols-and-strings) to know more about symbols and their performance when compared to strings.

<!-- more -->

One disadvantage of using symbols is that, they live in the program’s memory for the entire life cycle of it’s execution. This is one of the reasons for faster symbol lookups. This also was the reason for many denial of service vulnerabilites in Ruby, for example [this one](https://www.ruby-lang.org/en/news/2013/02/22/json-dos-cve-2013-0269/).

From Ruby 2.2, there is a new concept of Mortal and Immortal Symbols. Internally ruby has three types of symbols. Mortal dynamic symbols, which are symbols created dynamically by user, Immortal dynamic symbols are the mortal dynamic symbols that have been promoted to immortal dynamic symbols and then there are immortal static symbols, which are function names, class names etc. The GC only collects the
Mortal dynamic symbols. An example of such symbols are symbols created by `to_sym` method.
