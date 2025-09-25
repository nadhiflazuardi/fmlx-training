>**The Template Method Pattern** defines the skeleton of an algorithm in a method, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm’s structure.

There's an abstract class with a "template method" that holds the main algorithm. 
It holds algorithm by calling some other methods.
From those "other methods", there are some concrete methods, and some abstract methods.

In order for the algorithm to work, there need to be concrete classes that inherit the abstract class and complete the algorithm by implementing the abstract methods themselves (therefore "template method").

BTW, we can also have some "hook" methods that could optionally steer the flow of the algorithm
