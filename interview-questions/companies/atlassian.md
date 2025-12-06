# Atlassian Interview Experience

## Compensation Snapshot (India)
- **Base Salary:** ₹40–50 LPA (avg ~₹45 LPA)
- **Bonus + RSUs:** ~₹25 LPA
- **Total CTC:** ₹65–75 LPA

---

## Interview Process
- Karat Screening (live coding + rapid-fire)
- DSA Round – medium to hard level questions
- Low-Level Design – real-world object-oriented design
- Code Design – implementing real services
- System Design – scalable architecture discussions
- Values Round
- Managerial Round

---

## Recently Asked Problems & Topics

### DSA / Coding
- Snake Game logic with boundary conditions
- File collection: calculate total size & top-K largest
- Word search in 2D grid (only right/down moves)
- Anagram grouping and frequency mapping
- Sliding window + Trie-based problems

### Code Design (LLD)
- API Rate Limiter (Token Bucket / Leaky Bucket)
- Feature Flag Service with caching
- Logger System
- Cab Booking – Classes and Service design

### System Design
- Tagging system for large-scale platforms
- Web Crawler with nested link traversal
- Podcast Search Engine
- Ride-Sharing Database
- Dropbox-like File Storage

### Behavioral & Values
- Resolving team conflict
- Managing tight deadlines
- Ownership and initiative in tough situations

---

## Why Join Atlassian?
- Strong base salary + meaningful equity
- Remote/hybrid flexibility
- Focus on engineering excellence and practical work
- Great work-life balance with a product-first culture

---

## Interview Experience #2

**Compensation:** 54L + 9  
**Position:** Senior Software Engineer  
**Application Method:** Direct Application

### Round 1: Scenario Based Technical Questions + Coding

**A. Scenario Questions:**
- What are the potential issues with using consistent hashing for music streaming servers?
- What are the pros and cons of using pre-loaded hints versus server-loaded hints in an application?
- How would you process a file that is larger than the available RAM on a single system?
- You are tasked with building a sports news classification service that downloads articles and applies machine learning to detect bias. What information would you require to estimate the resources needed for this system?
- When expanding a production-ready application to multiple countries, what backend changes and considerations must be taken into account?

**B. Coding Round:**
- Given a list of words Words = ["baby", "cat", "dada", "dog"] and a random jumbled string like "ctay", write a function find(words, word1) that returns the word if it can be formed from the characters of the given string.
  - Example: find(words, "ctay") → returns "cat"
  - Example: find(words, "dad") → returns "-" (not found)
- Given a 2D matrix of characters, determine if a given word exists in the matrix by moving only right or down.

### Round 2: Data Structures Round
- Each file has a collectionId attached. How would you generate a report to show:
  - The total size of all files.
  - The top N collections ranked by total file size.
- How would you modify the system if multiple collections can be associated with a single file?
- How would you design and optimize this solution for a multithreaded environment to ensure correctness and efficiency?

### Round 3: Code Design Round
- How would you design a Rate Limiter?
- How would you scale this system to include a credit-based model, where unused requests are carried over as credits?
- How would you implement and manage this system in a multithreaded environment?

### Round 4: System Design Round
- Design a Web Scraper system that:
  - Starts with an initial set of URLs.
  - Scrapes all nested URLs recursively.
  - Extracts and returns all image links, mapped against their respective parent URLs.
- Follow-up questions:
  - Handling depth limits.
  - Optimizing scraping for speed and server load.
  - Ensuring fault tolerance and retries.

### Round 5: Managerial Round
- Describe a situation where you successfully handled a project with vague or unclear requirements.
- Explain how you mentored or helped a team member grow technically or professionally.

