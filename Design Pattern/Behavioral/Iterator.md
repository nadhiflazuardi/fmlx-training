>**The Iterator Pattern** provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

- Used when you want to decouple a client from multiple concrete collections with different structures. Iterator pattern uses common interface for collections. This provides uniform way of accessing elements of collections.
- Also used when you want to iterate through a collection, but want to hide the complexity (e.g. tree)

- The Iterator Pattern allows traversal of the elements of an aggregate without exposing the underlying implementation.
- It also places the task of traversal on the iterator object, not on the aggregate, which simplifies the aggregate interface and implementation, and places the responsibility where it should be.