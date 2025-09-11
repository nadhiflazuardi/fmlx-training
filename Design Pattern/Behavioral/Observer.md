A subject notifies one or more observers when something happens

- Observers need to implement a common interface so that subject can store observers and call their update methods without having to care about their concrete implementation.
- The subject is recommended to implement an interface in case observer wants to swap subject.
- Data can be pushed by subject to observers, or pulled by each of observer from the subject. The latter is considered more "correct" because each observer can pull just the data that they need. Otherwise, every observer will have to deal with all data sent by the subject, even if they don't need it.