Works only when core model of thee app is like a tree structure

Each leaf and branch implement the same interface

Command will be sent from the root. Leaf will do the work immediately, while branch will propagate the command to the next leaves and branches, and then "sum up" the result.