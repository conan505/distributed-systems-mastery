# Google Interview Experience

**Leetcode Practice List:** https://leetcode.com/problem-list/2mxn884m/

## Salary Snapshot
- **Base:** ₹45–55 LPA
- **Stocks (RSUs):** ₹30–40L+ over 4 years
- **Performance Bonus:** ~15–20%
- **Total CTC:** ₹85–110 LPA (based on team & location)

---

## Interview Process
1. Online Coding Round – 2 DSA problems (LeetCode Medium-Hard level)
2. 2 Rounds of DSA + Problem Solving
3. Low-Level/System Design Round
4. Googleyness + Behavioral
5. Hiring Committee Review

---

## Favored DSA Problems
- Median in Data Stream
- Serialize and Deserialize a Binary Tree
- Hard Graph Problems (e.g., Word Ladder II, Topo Sort, Bridges)
- Range Minimum Query (Segment Tree)
- Regular Expression Matching
- Merge Intervals / Skyline Problem
- LFU Cache / Least Recently Used (LRU)
- Trie Problems (with Wildcards)

## Low Level Design (LLD)
- Design Google Docs (collaboration)
- Thread Pool
- File System
- Rate Limiter
- Elevator System / Splitwise

## High Level System Design
- Google Search Autocomplete
- YouTube-like Video Streaming
- Google Calendar Backend
- Google Maps Navigation / Live Traffic
- Distributed File Storage System (GFS-like)

---

## Interview Experience #2

### Round 1: Screening Round – DSA + Optimisation
**Problem:** Given an array of integers, return the length of the longest subarray where the sum is divisible by K.

**Edge Conditions:**
- Negative numbers may exist
- Large array size

### Round 2: Screening Round – DSA
**Problem:** Given an array of integers, return the minimum number of swaps required to sort the array in non-decreasing order.

**Bonus:**
- Optimise for time and space
- Write unit tests for already sorted, reverse sorted, and duplicate entries

### Round 3: Onsite – Concurrency + Scheduling
**Problem Statement:** You're given access to a system of worker threads. Each worker can perform a task, but may fail randomly. Implement a robust scheduler that:
- Retries failed tasks
- Ensures a task is only retried up to 3 times
- Distributes tasks evenly across all threads

**Tech Challenge:**
- Use non-blocking queues
- Retry logic
- Thread-safety considerations

### Round 4: Onsite – System Design + API Thinking
**Task:** Design a calculator library that supports basic arithmetic operations, nested expressions, and variables. It should also:
- Support setting variables (e.g., let x = 2)
- Evaluate expressions using those variables (e.g., x + 3)

**Expectations:**
- User-facing API
- Expression parsing with operator precedence
- Error handling and variable scoping

### Round 5: Googliness – Behavioral
- Tell me about a time when you made a technical decision that went against your team's opinion. How did you handle it?
- Describe a time you received critical feedback. What did you do next?
- How do you ensure junior engineers are growing while balancing project deadlines?

