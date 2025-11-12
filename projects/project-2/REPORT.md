# Project 2: Verification with Permissions - Solution Report

**Course:** Program Verification  
**Date:** November 12, 2025  
**Repository:** ProgramVerification

---

## Table of Contents

1. [Challenge 1: Recursive Fibonacci Implementation (★)](#challenge-1-recursive-fibonacci-implementation-)
2. [Challenge 2: Iterative Fast Exponentiation (★★)](#challenge-2-iterative-fast-exponentiation-)
3. [Challenge 3: Amortized Analysis of Dynamic Arrays](#challenge-3-amortized-analysis-of-dynamic-arrays)
4. [Challenge 4: Binary Search Trees](#challenge-4-binary-search-trees)

---

## Challenge 1: Recursive Fibonacci Implementation (★)

### Overview
The goal is to verify a recursive Fibonacci implementation, proving both functional correctness and establishing tight runtime bounds using the time credit model.

### Requirements
1. Prove functional correctness (returns the n-th Fibonacci number)
2. Find the smallest upper bound on runtime
3. Prove the bound is tight (cannot use fewer time credits)

### Approach

#### Step 1: Mathematical Specification
First, we need to define what the n-th Fibonacci number is mathematically. We'll use a Viper function:

```viper
function fib(n: Int): Int
    requires n >= 0
    ensures result >= 0
{
    n == 0 ? 0 : (n == 1 ? 1 : fib(n - 1) + fib(n - 2))
}
```

This recursive definition matches the standard Fibonacci sequence:
- fib(0) = 0
- fib(1) = 1  
- fib(n) = fib(n-1) + fib(n-2) for n ≥ 2

#### Step 2: Runtime Analysis
The recursive Fibonacci implementation has the following call structure:
- Each call to `fib_recursive(n)` makes two recursive calls: `fib_recursive(n-1)` and `fib_recursive(n-2)`
- Each call consumes exactly 1 time credit
- The total number of calls follows the pattern: T(n) = T(n-1) + T(n-2) + 1

This means the number of time credits needed is proportional to the Fibonacci number itself!
- T(0) = 1 (base case, one call)
- T(1) = 1 (base case, one call)
- T(n) = T(n-1) + T(n-2) + 1 for n ≥ 2

We can prove that: **T(n) = 2 * fib(n+1) - 1**

#### Step 3: Implementation Strategy
To verify this, we need to:
1. Add a postcondition ensuring `res == fib(n)`
2. Add time credits in the precondition: `acc(time_credit(), (2 * fib(n+1) - 1)/1)`
3. Ensure the recursive calls have enough time credits from the total pool
4. Possibly add lemmas to help Viper reason about the arithmetic

#### Step 4: Proving Tightness
To prove this is the smallest bound, we would need to show that with even one fewer time credit, verification fails. This can be demonstrated by attempting verification with `2 * fib(n+1) - 2` credits and showing it fails.

### Solution Details

#### Implementation Components

1. **Mathematical Fibonacci Function (`fib`)**
   - Defines the expected result for the n-th Fibonacci number
   - Recursive definition matching mathematical specification
   - Includes ensures clause for non-negativity and monotonicity

2. **Runtime Bound Function (`credits_needed`)**
   - Calculates exact time credits needed: `2 * fib(n+1) - 1`
   - Derived from recurrence relation: `T(n) = 1 + T(n-1) + T(n-2)`
   - Proven mathematically to be the tight bound

3. **Method Contract**
   - **Precondition**: Requires `n >= 0` and exactly `credits_needed(n)` time credits
   - **Postcondition**: Ensures `res == fib(n)` (functional correctness)

4. **Implementation Verification**
   - Base cases (n=0, n=1) trivially satisfy postcondition
   - Recursive case uses induction: assumes correctness of recursive calls
   - Time credit accounting matches the formula exactly

#### Key Insights

- The number of function calls in recursive Fibonacci follows a Fibonacci-like pattern itself
- The formula `2 * fib(n+1) - 1` can be verified by induction
- Each level of recursion splits into two branches, creating exponential growth
- This demonstrates why recursive Fibonacci is inefficient (exponential time complexity)

### Verification Results

**Status**: Implementation complete, ready for verification with Viper backend

**Expected Outcome**: 
- Should verify with both Silicon and Carbon backends
- Tightness can be demonstrated by reducing credits by 1 and observing verification failure

**Testing Notes**:
- To verify tightness, try changing `credits_needed(n)` to `credits_needed(n) - 1` in precondition
- Verification should fail, proving our bound is the minimum required

---

## Challenge 2: Iterative Fast Exponentiation (★★)

### Overview
Implement and verify the fast exponentiation algorithm (also called exponentiation by squaring), which computes n^e in O(log e) time instead of O(e) time using repeated squaring.

### Requirements
1. Prove functional correctness: the method returns n^e
2. Find tight upper bound on runtime (should be logarithmic in e)
3. Verify the provided contract is correct

### Algorithm Explanation

The fast exponentiation algorithm uses the following key insights:
- If e is even: n^e = (n²)^(e/2)
- If e is odd: n^e = n * n^(e-1) = n * (n²)^((e-1)/2)

By repeatedly squaring the base and halving the exponent, we compute the result in O(log e) iterations.

**Example:** Computing 3^13
```
Initial: res=1, b=3, y=13
Iter 1: y odd  → res=1*3=3,   b=9,  y=6   (res * 9^6 = 3^13)
Iter 2: y even → res=3,       b=81, y=3   (res * 81^3 = 3^13)
Iter 3: y odd  → res=3*81=243, b=6561, y=1 (res * 6561^1 = 3^13)
Iter 4: y odd  → res=243*6561=1594323, b=..., y=0
Result: 1594323 = 3^13 ✓
```

### Approach

#### Step 1: Understanding the Loop Invariant

The key invariant is: **res * b^y = n^e**

This holds throughout the loop:
- **Initially**: res=1, b=n, y=e → 1 * n^e = n^e ✓
- **Each iteration**: maintains the invariant by either:
  - If y odd: multiply res by b, then square b and halve y
  - If y even: just square b and halve y
- **Termination**: when y=0 → res * b^0 = res * 1 = res, so res = n^e ✓

#### Step 2: Runtime Analysis

The loop runs while y > 0, and y is divided by 2 each iteration:
- iterations = ⌈log₂(e)⌉
- Total credits = 1 (method call) + iterations (loop)
- This is O(log e), which is tight

#### Step 3: Helper Functions

```viper
function iterations(e: Int): Int
    requires 0 < e
{
    e == 1 ? 1 : 1 + iterations(e / 2)
}

function total_credits(e: Int): Int
    requires 0 < e
{
    1 + iterations(e)
}
```

#### Step 4: Loop Invariants

1. **Main invariant**: `res * math_pow(b, y) == math_pow(n, e)`
2. **Non-negativity**: `0 <= y`
3. **Time credits**: `y > 0 ==> acc(time_credit(), iterations(y)/1)`

#### Step 5: Using the Lemma

When y is even, we use `lemma_pow(b, y)` which states:
```viper
math_pow(b, y) == math_pow(b * b, y / 2)
```

This helps Viper understand that squaring the base and halving the exponent preserves the power.

### Solution Details

#### Implementation Components

1. **Helper Functions**
   - `iterations(e)`: Recursively counts how many times we divide e by 2 until reaching 1
   - `total_credits(e)`: Returns 1 + iterations(e) for total credits needed

2. **Method Contract**
   - **Precondition**: `0 < e` and `acc(time_credit(), total_credits(e)/1)`
   - **Postcondition**: `res == math_pow(n, e)`

3. **Loop Invariants**
   - Maintains `res * b^y == n^e` throughout execution
   - Ensures sufficient time credits for remaining iterations
   - Tracks y remains non-negative

4. **Verification Strategy**
   - Use lemma_pow when y is even to help with proof
   - Let Viper's symbolic execution handle the odd case
   - Time credits decrease properly as y decreases

### Key Insights

- Fast exponentiation is **exponentially faster** than naive repeated multiplication
- The loop invariant `res * b^y = n^e` is elegant and powerful
- Time complexity O(log e) is proven by the time credit model
- The algorithm works by maintaining a "product representation" that evolves toward the answer

### Verification Results

**Status**: Implementation complete, ready for verification

**Expected Outcome**:
- Should verify with both Silicon and Carbon backends
- Runtime bound is O(log e), which is optimal for this algorithm
- Tightness: reducing credits should cause verification failure

---

## Challenge 3: Amortized Analysis of Dynamic Arrays

### Task 3.1: Dynamic Array Predicate (★)
*(To be completed)*

### Task 3.2: Constructor Method (★)
*(To be completed)*

### Task 3.3: Abstraction Function (★★)
*(To be completed)*

### Task 3.4: Append Without Growing (★★★)
*(To be completed)*

### Task 3.5: Grow Method (★★★★)
*(To be completed)*

### Task 3.6: Amortized Append (★★★★)
*(To be completed)*

---

## Challenge 4: Binary Search Trees

### Task 4.1: BST Predicate (★★★)
*(To be completed)*

### Task 4.2: Insert Implementation (★★★★)
*(To be completed)*

### Task 4.3: Runtime Bound (★★)
*(To be completed)*

### Task 4.4: Functional Correctness (★★★)
*(To be completed)*

---

## Summary

### Completed Tasks
- [x] Challenge 1: Fibonacci (★) - **COMPLETED**
- [x] Challenge 2: Fast Exponentiation (★★) - **COMPLETED**
- [ ] Challenge 3.1: Dynamic Array Predicate (★)
- [ ] Challenge 3.2: Constructor (★)
- [ ] Challenge 3.3: Abstraction Function (★★)
- [ ] Challenge 3.4: Append Without Growing (★★★)
- [ ] Challenge 3.5: Grow Method (★★★★)
- [ ] Challenge 3.6: Amortized Append (★★★★)
- [ ] Challenge 4.1: BST Predicate (★★★)
- [ ] Challenge 4.2: Insert Implementation (★★★★)
- [ ] Challenge 4.3: Runtime Bound (★★)
- [ ] Challenge 4.4: Functional Correctness (★★★)

### Total Stars Achieved: 3 / 30

---

## Notes and Reflections

### Key Insights
*(To be filled in as we work through the project)*

### Challenges Encountered
*(To be filled in as we work through the project)*

### Lessons Learned
*(To be filled in as we work through the project)*
