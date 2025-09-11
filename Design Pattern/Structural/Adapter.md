>**The Adapter Pattern** converts the interface of a class into another interface the clients expect. Adapter lets classes work together that couldn’t otherwise because of incompatible interfaces.

- Adapters allow clients to use new libraries without changing any code

- Take a solid look at the expected interface and adaptee's interface, see how the latter can be mapped into the former

- There are 2 ways to implement this pattern: object adapter and class adapter
	- Object adapter is done through wrapping the adaptee with the adapter
	- Class adapter is done through inheriting both interfaces

- Client doesn't even know that it's talking to a wrapper