---
title: 'Java Streams'
date: 2024-07-29T16:21:42-04:00
draft: false
tags:
  - java
  - streams
  - interview
categories: notes
keywords:
  - java
  - streams
---

## JAVA Streams

### Sum of Elements

Given a list of integers, compute the sum of all numbers.

```java
// using sum
List<Integer> arr = Arrays.asList(1, 4, 5, 6, 22, 3, 90, 89, 2, 1, 3, 4, 55, 6);

int sum = arr.stream()
            .mapToInt(Integer::intValue)
            .sum();
// or
int sum3 = arr.stream()
            .reduce((a,b) -> a + b)
            .get();

// or
int sum2 = arr.stream()
        .reduce(0,
                (total, n) -> total + n,
                (total1, total2) -> total1 + total2);
```

### Average of Elements

Given a list of integers, compute the average of all numbers.

```java
List<Integer> arr = Arrays.asList(1, 4, 5, 6, 22, 3, 90, 89, 2, 1, 3, 4, 55, 6);
// average of all numbers
double avg = arr.stream()
        .mapToInt(n -> n)
        .average()
        .orElse(0);
```

### Square of All Numbers

Given a list of numbers, get the average of squares of all numbers whose square is > 100.

```java
List<Integer> arr = Arrays.asList(1, 4, 5, 6, 22, 3, 90, 89, 2, 1, 3, 4, 55, 6);
    double avg = arr.stream()
            .mapToInt(n -> n * n)
            .filter(n -> n > 100)
            .average()
            .orElseGet(() -> 0.0);
```

### Partition Numbers

Given a list of numbers, separate even and odd numbers.

```java
List<Integer> arr = Arrays.asList(11, 2, 3, 45, 67, 9, 90, 87, 8, 2);
Map<Boolean, List<Integer>> collect = arr.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));

// or mapping
Map<String, List<Integer>> listMap = arr.stream()
            .collect(groupingBy(n -> n % 2 == 0 ? "EVEN" : "ODD", toList()));
```

### Print Numbers Starts With Prefix 2

Given a list of numbers, print numbers starting with 2.

```java
List<Integer> arr = Arrays.asList(11, 2, 3, 45, 67, 9, 90, 87, 8, 2, 22, 201, 302, 1222);

List<Integer> startingWith2 = arr.stream()
        .map(String::valueOf)
        .filter(str -> str.startsWith("2"))
        .map(Integer::valueOf)
        .toList();

// or, Solution is the class with startWith2 function
List<Integer> startingWith2 = arr.stream()
            .filter(Solution::startWith2)
            .toList();

private static boolean startWith2(int n) {
    int temp = n;
    while (temp > 0 && temp != 2) {
        temp /= 10;
    }
    return temp == 2;
}
```

### Print Duplicate Numbers using Streams

Given a list of numbers, find all the duplicate numbers.

```java
List<Integer> arr = Arrays.asList(11, 2, 3, 45, 67, 9, 90, 87, 8, 2, 22, 201, 302, 1222);
Set<Integer> ele = arr.stream()
        .filter(n -> Collections.frequency(arr, n) > 1)
        .collect(Collectors.toSet());

// or good solution would be
List<Integer> duplicates = arr.stream()
            .collect(groupingBy(n -> n, counting()))
            .entrySet()
            .stream()
            .filter(entry -> entry.getValue() > 1)
            .map(Entry::getKey)
            .toList();

// or
List<Integer> duplicates2 = arr.stream()
            .collect(Collectors.collectingAndThen(
                    Collectors.groupingBy(e -> e, Collectors.counting()),
                    map -> map.entrySet().stream()
                            .filter(entry -> entry.getValue() > 1)
                            .map(Map.Entry::getKey)
                            .collect(Collectors.toList())
            ));
```

### Find Max and Min Numbers using Streams

Get the minimum value of the list.

```java
List<Integer> arr = Arrays.asList(11, -2, 3, 45, 67, 9, 90, 90, 87, 8, 2, 22, 201, 302, 1222);

int min = arr.stream()
            .min(Comparator.comparing(Integer::intValue))
            .get();
```

### Get the sum of first 5 numbers

Given the list of numbers, find the sum of the first 5 numbers.

```java
List<Integer> arr = Arrays.asList(11, -2, 3, 45, 67, 9, 90, 90, 87, 8, 2, 22, 201, 302, 1222);

int sum = arr.stream()
        .limit(5)
        .mapToInt(n -> n)
        .sum();

// or

int sum2 = arr.stream()
        .limit(5)
        .reduce(0, Integer::sum);
```

### Second Lowest Smallest Number

Find the second smallest number in the stream.

```java
List<Integer> arr = Arrays.asList(11, -2, 3, 45, 67, 9, 90, 90, 87, 8, 1, 0, 2, 22, 201, 302, 1222);

Integer secondSmallest = arr.stream()
        .sorted()
        .skip(1)
        .findFirst()
        .orElseGet(() -> Integer.MAX_VALUE);
```
