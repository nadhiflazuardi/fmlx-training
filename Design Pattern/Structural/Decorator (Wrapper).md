Wrapping object over and over again to add more functionality

- Create an abstraction of the objects to be decorated
- Create another abstraction of decorator that extends it
- Concrete decorators will extend the abstract decorator. This way, all concrete decorators will share the same supertype with the objects being decorated
- Concrete decorator keeps a reference of the decorated object 
- This way, a decorator can do what the decorated object is able to, plus more