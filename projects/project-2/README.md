# Project 2: Verification with Permissions

Throughout this course, we focused on program correctness, that is, how to specify what a program should do and how to verify that it behaves as specified. Apart from writing correct programs, we are, of course, also interested in writing efficient programs. In this project, we will develop a verification methodology for proving upper bounds on the runtime of heap-manipulating programs on top of proving them functionally correct.

We first explain the project guidelines, followed by an introduction to our runtime model. Finally, we discuss the individual tasks.

## Guidelines

All project tasks must be solved using Viper. In addition to the course material, you may find the Viper tutorial useful. Your solutions must verify with at least one Viper backend (Silicon or Carbon) and should verify with both backends.

The project consists of four challenges, which will be explained in detail further below. For each challenge, we provide a Viper file with a code skeleton. You must put your solutions into those files. All challenges are independent. You can work on them in any particular order.

Submit your final solution via the corresponding assignment on DTU learn.

Unless explicitly specified otherwise (e.g. if the task says "implement"), you can add ghost code and specifications, but you are not allowed to change the production code.

As in the first project, we provide a number of stars for each task to indicate a rough estimate of the required effort and difficulty.

- In total, there are 30 stars.
- To obtain a grade of:
    - **12**, you should aim for at least 27 stars;
    - **10**, you should aim for at least 22 stars;
    - **7**, you should aim for at least 18 stars;
    - **4-2**, you should aim for at least 15 stars.

Those numbers assume a group of three students. You can subtract 1 (resp. 2) stars if you work in a group of two (resp. one) students.

Please document your code carefully such that we can award partial stars for solutions that do not work perfectly.

Please note that everything you manually add to the trusted code base must be justified, e.g. by adding suitable comments to the affected code. In particular, it should not be possible to successfully assert false anywhere in your solution.

## Runtime Model

To abstract from the execution times on a specific machine, we will assume an abstract time unit and use a runtime model in which every method call and every loop iteration consumes one abstract time unit; all other statements are considered to consume no time at all. Certain basic statements, such as allocating a static array, are considered as statements instead of method calls and thus, for simplicity, consume no time.

We will use an abstract Viper predicate to represent an abstract, non-duplicable resource, called a time credit, that to represent one abstract unit of time:

```viper
predicate time_credit()
```

Every predicate instance of `time_credit` that we hold thus allows us to perform an operation that requires one abstract unit of time. We say that the time credit is consumed by that operation, i.e. we cannot use the same time credit to perform two operations that require time credits.

More precisely, we use the following abstract method to model consuming a time credit:

```viper
method consume_time_credit()
    requires time_credit()
```

Whenever a statement consumes time, we invoke the above method. Intuitively, the number of time credits available to our program thus represents an upper bound on the number of time units we can use during execution. Verification will fail if we do not have enough time credits left. In other words, a time credit can be thought of as a resource that gives us permission to use execution time, i.e. to perform a loop iteration or call a method.

For example, the method below does not verify. In the precondition, we claim that we can execute it with just two time credits, but we attempt to consume three time credits in its body:

```viper
method example()
    requires time_credit() && time_credit()
{
    consume_time_credit()
    consume_time_credit()
    consume_time_credit() // error: not enough time credits
}
```

Alternatively, we can represent the above precondition using fractional permissions to predicate instances, that is, using the following precondition:

```viper
requires acc(time_credit(), 2/1)
```

In contrast to heap locations, recall that we can hold more than one full permission to predicate instances. This notation also allows the number of time credits to depend on program variables. For example:

```viper
method example2(n: Int)
  requires acc(time_credit(), n/1)
```

In the following tasks, we consider a runtime model that adheres to the following two rules:

1. Every (non-ghost) method implementation must start with a call to `consume_time_credit()`. Hence, every (executable) method consumes at least one time credit.

2. Apart from ghost code, every loop body must start with a call to `consume_time_credit()`. Hence, every loop iteration consumes at least one time credit.

We do not allow creating time credits out of thin air. Instead, the amount of time credits available is determined by a method's precondition. If we manage to verify a program with these time credits, then the initial amount of time credits is an upper bound on the execution time of a program in our runtime model.

In other words, if time credits are correctly consumed by any operation we use, we show that the amount of time credits in the precondition is sufficient to execute a given method. We thus prove an upper bound on the program's execution time with respect to our runtime model. We only prove an upper bound, because we could have allocated more time credits in the precondition than required.

## Challenge 1: Recursive Fibonacci Implementation (★)

The Viper code in the file `fibonacci.vpr` contains a method that recursively computes the n-th Fibonacci number. First, prove functional correctness, i.e. that the method indeed returns the n-th Fibonacci number. After that, find the smallest upper bound on the method's runtime in our runtime model. Finally, prove that your bound is indeed the smallest upper bound on the method's runtime.

## Challenge 2: Iterative Fast exponentiation (★★)

The Viper code in the file `fastexp.vpr` shows a method that iteratively computes n^e.

Prove that the provided contract is correct. Furthermore, find an upper bound on the method's runtime (according to our runtime model) and use time credits to prove that bound. Try to make your upper bound as small as possible.

## Challenge 3: Amortized Analysis of Dynamic Arrays

In this challenge, we consider dynamic arrays, i.e. arrays that are stored on the heap and whose capacity can be resized if necessary. Similar data structures are found in many mainstream programming languages, for example ArrayList in Java or std::vector in C++.

Intuitively, a dynamic array stores a static array of some fixed capacity on the heap. As long as this static array is not full, inserting an element into a dynamic array simply amounts to inserting that element into the underlying static array.

However, if the underlying static array is full, we first have to resize our dynamic array. That is, we create a new static array that is twice as large and copy over all elements from the old static array. After that, we insert the given element into our new static array as usual.

The Viper file `dyn_array.vpr` contains an implementation of such a dynamic array in Viper. In particular, the method

```viper
method append(arr: Ref, val: Int) returns (new_arr: Ref)
```

takes a reference to a dynamic array `arr` and a value `val` as an input then then inserts `val` into `arr`. The returned reference `new_arr` returns object `arr`, but potentially grows the underlying static array if necessary.

Throughout the tasks described further below, you should prove the following for the given implementation in `dyn_array.vpr`:

1. The implementation of `append` is memory safe.
2. The implementation of `append` is functionally correct.
3. The worst-case execution time of `append` is linear in the given dynamic array's capacity.
4. The amortized execution time of `append` is constant.

### Worst-Case Runtime vs. Amortized Runtime

Regarding items (3) and (4), the worst-case execution time of `append` is linear in the size of the array, because the underlying static array might be full. In this case, we first have to allocate a new array and copy over all previous elements before inserting a new one. The execution time for copying is linear in the size of the original static array.

Notice that the worst-case execution time of `append` is constant if we know that the given array is not full yet.

The idea of amortized analysis is that such a resizing of the underlying static array rarely happens, because we first have to fill an array in order to encounter this behavior. Instead of considering the worst-case execution time of a single execution of `append`, amortized analysis thus proposes to consider the average number of time credits required over a long sequence of append operations.

For example, if we have an empty dynamic array of capacity n and insert n+1 elements, then the underlying static array is enlarged only once. We thus consider 1 operation that has a runtime of O(n), say n, and n operations that have a runtime of O(1), say 1. Since we consider a sequence of n+1 operations, the average runtime of each operation is (n + n * 1) / (n+1), i.e. constant. We thus say that the amortized execution time of `append` is constant.

A common approach for analyzing the amortized runtimes is the so-called banker's method. In terms of our runtime model, the main idea is to save additional time credits whenever we perform a cheap operation and then use those savings to pay for expensive operations.

For example, whenever we attempt to insert an element into an array that is not full yet, we may require an additional time credit, which is attached to the inserted array element. If we later have to copy the full array, we may consume those saved time credits to copy all array elements. The `append` method thus only needs to supply a constant number of time credits in its precondition as long as we have saved a time credit for every element whenever the array is full.

### Task 3.1 (★)

Define a predicate modelling the representation invariants and permissions of dynamic arrays.

You may also store other (ghost) information in your predicate. Feel free to add accessor functions to simplify fold-unfold reasoning.

### Task 3.2 (★)

Implement a proven-correct method `cons(_capacity: Int) returns (arr: Ref)` that creates a new dynamic array of length 0 with the given capacity.

To be considered "correct", you should at least prove that `cons` is memory safe, creates a dynamic array of the given capacity, and can be executed with a constant number of time credits.

### Task 3.3 (★★)

Define an abstraction function `arr_contents` that maps a dynamic array to the mathematical sequence of values stored in its elements.

Hint: For other tasks, you might have to prove additional properties about the abstraction.

### Task 3.4 (★★★)

Prove that the method `append_nogrow(arr: Ref, val: Int)` appends the value `val` to the dynamic array `arr` without increasing the capacity first.

You must prove memory safety, preservation of the dynamic array's representation invariants, and that the method can be executed in constant time. Furthermore, use abstraction to prove that `val` has been correctly appended.

Hints:

- Notice that `append_nogrow` can only be called if there is enough space left in the dynamic array.
- For amortized analysis, we also want to save a time credit such that we can grow the array later if necessary.
- You can add specifications and ghost code, but do not modify the production code

### Task 3.5 (★★★★)

Prove that the method `grow` correctly grows the dynamic array by creating a copy of the underlying static array with twice the capacity.

To this end, show memory safety, preservation of the dynamic array property, that the returned array has the right capacity, and that it indeed contains a copy of the original array.

Hints:

- Notice that `append_nogrow` can only be called if there is enough space left in the dynamic array.
- For worst a worst-case execution time analysis, you can require a number of time credits that depends on the array's capacity.
- However, for amortized analysis, your method may require only a constant number of time credits, i.e. the number of time credits cannot depend on any variable. You may, of course, use additional time credits that have been stored in your dynamic array for later use.

### Task 3.6 (★★★★)

Prove that the `append` method is memory safe, functionally correct, and can be executed in amortized constant time.

Hints:

- You may require that a certain number of time credits have been saved up in the data structure that you can use for growing the data structure.

## Challenge 4: Binary Search Trees

A binary tree is a structure consisting of nodes containing an integer value, an optional left subtree, and an optional right subtree. A binary search tree is a binary tree where all node values in the left subtree are strictly smaller than the current node's value, and all node values in the right subtree are strictly greater than the current node's value. Note that this definition also implies that there are no duplicates in the tree.

The goal of this challenge is to develop a proven-correct implementation of an insertion method for heap-allocated binary search trees (BSTs) and to analyze its runtime using time credits. More precisely, inserting a new node into a binary tree should preserve the BST property from above and require at most h+1 time credits, where h denotes the height of the original BST.

The Viper file `bst.vpr` already provides a code skeleton with the necessary fields and some utility functions. Modify this code skeleton when solving the following tasks.

### Task 4.1 (★★★)

Define a BST predicate `bst(self: Ref)` specifying that `self` points to the root of a binary search tree.

Hint: You may want to add ghost arguments or heap-dependent functions for abstraction.

### Task 4.2 (★★★★)

The method

```viper
method bst_insert(tree: Ref, val: Int)
```

takes a reference to the root of a BST `tree` and a value `val` as an input. It then inserts `val` into the BST.

Implement `bst_insert` and verify the following properties:

1. Your implementation is memory safe.
2. Your implementation indeed preserves the BST property, i.e. if `tree` is the root of a BST before the insertion, it is also a BST after the insertion.

Make sure to implement `bst_insert` faithfully, i.e. your implementation should indeed insert the given value (if it does not already exist) into the given BST, even if you do not fully verify this.

### Task 4.3 (★★)

Use the time credit model to verify that executing `bst_insert(tree: Ref, val: Int)` requires at most h+c abstract time units, where h is the height of the given input tree and c is some constant (typically 1 or 2).

If the input tree is balanced, this essentially shows that inserting a node can be performed in time logarithmic in the total number of nodes.

Hint: You might have to re-visit your implementation from Task 4.2.

### Task 4.4 (★★★)

Verify that the set of values stored in the initial tree extended by `val` is the same as the set of values stored in the tree after execution of `bst_insert(tree, val)` has terminated.

Hint: You might have to re-visit your implementation from Task 4.2.

## Summary of Tasks

| Challenge | Task | Stars | Description |
|-----------|------|-------|-------------|
| 1 | Fibonacci | ★ | Recursive Fibonacci with runtime analysis |
| 2 | Fast Exponentiation | ★★ | Iterative exponentiation with runtime bound |
| 3.1 | Dynamic Arrays | ★ | Define dynamic array predicate |
| 3.2 | Dynamic Arrays | ★ | Implement constructor |
| 3.3 | Dynamic Arrays | ★★ | Define abstraction function |
| 3.4 | Dynamic Arrays | ★★★ | Verify append_nogrow |
| 3.5 | Dynamic Arrays | ★★★★ | Verify grow method |
| 3.6 | Dynamic Arrays | ★★★★ | Verify append with amortized analysis |
| 4.1 | Binary Search Trees | ★★★ | Define BST predicate |
| 4.2 | Binary Search Trees | ★★★★ | Implement and verify bst_insert |
| 4.3 | Binary Search Trees | ★★ | Verify runtime bound |
| 4.4 | Binary Search Trees | ★★★ | Verify functional correctness |

**Total: 30 stars**
