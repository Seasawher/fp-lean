# Programming, Proving, and Performance

This chapter is about programming.
Programs need to compute the correct result, but they also need to do so efficiently.
To write efficient functional programs, it's important to know both how to use data structures appropriately and how to think about the time and space needed to run a program.

This chapter is also about proofs.
One of the most important data structures for efficient programming in Lean is the array, but safe use of arrays requires proving that array indices are in bounds.
Furthermore, most interesting algorithms on arrays do not follow the pattern of structural recursion—instead, they iterate over the array.
While these algorithms terminate, Lean will not necessarily be able to automatically check this.
Proofs can be used to demonstrate why a program terminates.

Rewriting programs to make them faster often results in code that is more difficult to understand.
Proofs can also show that two programs always compute the same answers, even if they do so with different algorithms or implementation techniques.
In this way, the slow, straightforward program can serve as a specification for the fast, complicated version.

Combining proofs and programming allows programs to be both safe and efficient.
Proofs allow elision of run-time bounds checks, they render many tests unnecessary, and they provide an extremely high level of confidence in a program without introducing any runtime performance overhead.
However, proving theorems about programs can be time consuming and expensive, so other tools are often more economical.

Interactive theorem proving is a deep topic.
This chapter provides only a taste, oriented towards the proofs that come up in practice while programming in Lean.
Most interesting theorems are not closely related to programming.
Please refer to [Next Steps](next-steps.md) for a list of resources for learning more.
Just as when learning programming, however, there's no substitute for hands-on experience when learning to write proofs—it's time to get started!



