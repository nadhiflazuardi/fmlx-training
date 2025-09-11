- Used when you don't know exactly which concrete implementation will the logic be dealing with
- Or, when you want to encapsulate and separate object creation matter

- Make an interface of the "product" that you want, and then concrete classes that implements it
- The concrete classes will later be instantiated by the factories

- Make an abstract class with some logic methods and an abstract object creation method
- Subclasses that act as "factories" will have to implement the object creation themselves. They will create concrete products according to their factory type