# What?
>**The Proxy Pattern** provides a surrogate or placeholder for another object to control access to it.

# Why?
We might want to control access to an object when the object is:
- running on a remote machine
- expensive to create
- in need of security

# How?
- make a common interface that both the object and the proxy implement
- proxy can take place of the original object anywhere