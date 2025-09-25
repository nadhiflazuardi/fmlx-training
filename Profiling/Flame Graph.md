## Note:
---
- Flame graph is NOT in chronological order
- The boxes are grouped by stack trace path (related calls are stacked)
- The width of each box is proportional to how often it appears on the samples

# What is a Flame Graph?
Visualization of **stack samples** taken from a running process

# How does it work?
A profiler "takes a screenshot" of your program many times per second and records all functions active at the moment.

Example:
main → a → b → c
main → a → b → c
main → a → d → e
main → a → b → c
>If c() appears in 90% of the samples, then your program was spending 90% of its time inside c()

After collecting samples, the profiler will then:
- aggregates the samples
- group identical call paths together
- stack them vertically

# Why use a Flame Graph?
To see:
- Overall structure of the runtime
- Which methods are using the most CPU time
- Redundant calls
