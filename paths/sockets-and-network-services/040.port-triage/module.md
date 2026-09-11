# Port triage

Three questions come up every time a service misbehaves, and you can now
answer all of them:

1. Is anything listening on the port?
2. Which process is it, and is it the one you meant?
3. Which address did it bind, and can the caller actually reach it?

This module stops introducing tools and starts combining them. Each rep
is a small failure of the kind that reaches you as "the API is down" -
and each one is fixed by looking at the socket table first and guessing
second.
