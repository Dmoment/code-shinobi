---
layout: post
title:  "Finding Maximum Sum of Contiguous Array"
date:   2023-05-21
---

<p class="intro"><span class="dropcap">K</span>adane Algorithm</p>

# Finding Maximum Sum of Contiguous Array: A Cross-Language Solution 

In this article, we're exploring a popular algorithm problem - finding the maximum sum of a contiguous array. The problem essentially requires us to find a contiguous subarray with the largest sum. We'll be using Python, Go, Ruby and JavaScript for this task.

## The Problem

Given an array of integers (both positive and negative), we need to find a contiguous subarray that yields the maximum sum.

## Python Solution

In Python, we're implementing two solutions: a traditional double loop approach and Kadane's algorithm which optimizes the process.

```python
import math

# Double loop approach
def max_subarray_sum(input):
  N = len(input)
  ans = input[0]
  for i in range(N):
    sum = 0
    for j in range(i, N):
      sum += input[j]
      ans = max(ans, sum)
  return ans 

# Kadane's Algorithm
def kadane_approach(input):
  N = len(input)
  ans = -math.inf
  sum = 0
  for i in range(N):
    sum += input[i]
    ans = max(ans, sum)
    if sum < 0:
      sum = 0
  return ans     

print(kadane_approach(input))
```

## Go Solution

In Go, we similarly provide two approaches - the traditional method and Kadane's Algorithm.

```go
  package main

import (
"fmt"
"math"
)

// Double loop approach
func maximum_subarray_sum(input []int) int{
  N := len(input)
  ans := input[0]
  sum := 0
  for i := 0; i< N ; i++ {
    sum = 0
    for j := i; j< N; j++{
      sum += input[j]
      if sum > ans {
        ans = sum
      }
    }
  }
  return ans
}

// Kadane's Algorithm
func kadane_approach(input []int) int{
  N := len(input)
  sum := 0
  ans := int(math.Inf(-1))
  for i:=0; i < N; i++ {
    sum += input[i]
    if sum > ans {
      ans = sum
    }
    if sum < 0{
      sum = 0
    }
  }
  return ans
}

func main() {
  input := []int{1, 2, 3, 4, -10}
  fmt.Println(maximum_subarray_sum(input))
  fmt.Println(kadane_approach(input))
}
```

## Ruby solution

For Ruby, we again use the two approaches.

```ruby
  # Double loop approach
def max_subarray_sum(input)
  n = input.length
  ans = input[0]
  for i in 0...n
    sum = 0
    for j in i...n
      sum = sum + input[j]
      ans = [ans, sum].max
    end
  end
  ans
end

# Kadane's Algorithm
def kadane_approach(input)
  n = input.length
  sum = 0
  ans = -Float::INFINITY
  for i in 0...n
    sum = sum + input[i]
    ans = [ans, sum].max
    sum = 0 if sum < 0
  end
  ans
end

print kadane_approach(input)
```

## Javascript Solution

Finally, we look at JavaScript, implementing both solutions.

```javascript
  // Double loop approach
function maxSubarraySum(input){
  let N = input.length
  let ans = input[0]
  for (var i = 0; i< N; i++){
    sum = 0
    for (var j = i; j<N; j++){
      sum += input[j]
      ans = Math.max(ans, sum)
    }
  }
  return ans
}

// Kadane's Algorithm
function kadane_approach(input){
  let N = input.length
  let sum = 0
  let ans = Number.NEGATIVE_INFINITY
  for (var i = 0; i< N; i++){
    sum += input[i]
    ans = Math.max(ans, sum)
    if(sum < 0){
      sum = 0
    }
  }
  return ans
}

console.log(maxSubarraySum([1, 2, 3, 4, -10]));
console.log(kadane_approach([1, 2, 3, 4, -10])); 
```

## Conclusion

While the double loop approach works, its time complexity is O(n^2), which isn't ideal for larger arrays. Kadane's algorithm, on the other hand, optimizes this and runs with a time complexity of O(n). This comparative analysis of the solutions in four different languages offers a clear view of how optimization can drastically improve performance.