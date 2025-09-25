>**The Composite Pattern** allows you to compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

Works only when core model of thee app is like a tree structure

Each leaf and branch implement the same interface

Command will be sent from the root. Leaf will do the work immediately, while branch will propagate the command to the next leaves and branches, and then "sum up" the result.