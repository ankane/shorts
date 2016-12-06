# The Safely Pattern

The Safely Pattern is a simple one. It allows you to tag non-critical code by wrapping it in function or block. It’s built on top of exception handling and follows these rules:

1. Raise exceptions in development and test environments
2. Catch and report exceptions in other environments

Its advantages over typical exception handling are:

1. It’s easier to write code without errors being caught in development and test environments
2. It allows you to keep your reporting [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)

It’s recommended to mark exceptions when reporting so it’s clear they were handled with this pattern. One way of doing this is to prefix the exception message with `[safely]`.

There are currently implementations in [Ruby](https://github.com/ankane/safely) and [JavaScript](https://github.com/ankane/safely.js).
