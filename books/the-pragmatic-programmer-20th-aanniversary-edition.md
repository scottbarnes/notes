The Pragmatic Programmer: 20th Anniversary Edition
==================================================

## Chapter 4: Pragmatic Paranoia
- How can I try implementing more design by contract (DBC) into Python and Go? Asserts and maybe decorators in Python. Or the library [DEAL](https://github.com/life4/deal). Go has a pair of similar libraries. Something about the original Go authors disliking `assert`? Why?
- Finish what you start: if a function starts something (e.g. opens a file or otherwise allocates a resource), it should finish it too. Context managers in Python and `defer` in Go. This way the two tasks aren't separated and are less likely to become broken down the line as people change code and increase the likelihood that a function I think will be called to finish the task won't be called (e.g. because of some new conditional check).

## Chapter 5: Bend, or Break
### Tip 45: Tell, don't ask
- Don't make decisions based on the internal state of an object and then update that object.
### Tip 46: Don't chain method calls

## Exercises
### Exercise 14
#### `int getSpeed()`
**pre-conditions**
- None? It's has no inputs.
**post-conditions**
- speed is 0 to 9?
- final speed == requested speed.

##### `void setSpeed(int x)`
**pre-conditions**
- Speed-to-be is 0 to 9
- Speed change is one unit.
- !empty if speed !0
**post-conditions**
- Speed is zero to 9
- Speed change is one unit.
- !empty if speed !0

##### `boolean isFull()` 
**pre-conditions**
- It's either full or !full
**post-conditions**
- It's either full or !full

##### `void fill()`
**pre-conditions**
- !full
**post-conditions**
- full

##### `void empty()`
**pre-conditions**
- full
**post-conditions**
- !full
