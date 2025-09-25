# What is Metaprogramming?
>Metaprogramming is writing programs that treat code itself as data — inspecting its structure or metadata in order to generate new code or change behavior dynamically.

- Metaprogramming is all about code treating other code as data
- Basically we write code that can manipulate other code

# How can it be Helpful?
- Helping in reasoning about the code (e.g. adding attributes might add clarity)
- Simplifying code (e.g. code of certain process can be simplified with a single attributed

# What are the Common Metaprogramming Practices?
## Attributes + Reflection
---
Runtime metaprogramming

>Reflection is an ability of program to examine and modify it's own structure/behavior at runtime

1. Add attributes to code like `[HttpGet]` or `[Test`
2. Compile
3. The attributes will be stored as metadata in DLL
4. Framework will inspect the attributes (reflection) at runtime and adapt behavior

## Data-driven Programming
---
Example: Jenkins

1. Use external file (e.g. YAML/JSON/XML) that acts as an instruction of how the program should work
2. Feed it to the program

## Source Code Generation
---
Compile-time metaprogramming

There are attributes (e.g. `[AutoNotify]`, `[JsonSerializable]`) that when used, the compiler will generate more code (e.g. `.g.cs`) and add it to DLL

## LINQ Expression 
---
`db.Users.Where(u => u.Age > 18); // LINQ-to-SQL`

EF Core inspects -> build SQL queries (runtime inspection)


## Dynamic Proxies
---
Generate new classes at runtime in memory

Example: Moq objects