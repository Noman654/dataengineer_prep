# Jio (Reliance) — Data Engineer Interview

> **Sources:** GeeksForGeeks, Naukri, LeetCode Discuss, InterviewBit, community experiences
> **Last updated:** 2026-04-22
> **Roles covered:** Data Engineer (L2/L3), SDE-2

---

## Round Breakdown

| Round | Format | Duration | Focus |
|---|---|---|---|
| Round 1 | Online Coding Assessment (InterviewBit / similar) | ~60–90 min | DSA |
| Round 2 | Technical Interview I | ~60 min | Big Data, Spark, Hadoop, SQL |
| Round 3 | Technical Interview II | ~60 min | Projects, System Design |
| Round 4 | HR / Behavioural | ~30–45 min | Culture fit, STAR questions |

---

## Round 1 — Online Coding Assessment (DSA)

Questions are typically **Easy–Medium** LeetCode level.

### Questions asked

1. Find all distinct triplets in an array that sum to K.
2. Check if two strings are anagrams of each other.
3. Find the nth node from the end of a linked list.
4. Implement a stack using two queues.
5. Minimize the number of swaps required to sort an array.
6. Find the maximum water that can be trapped between heights (trapping rain water variation).
7. Find the maximum area rectangle in a histogram.

> **Tip:** Always state Time and Space complexity after every solution. Practice on LeetCode Easy/Medium.

---

## Round 2 — Technical Interview I (Big Data + SQL)

### Spark

1. Explain the Spark architecture — Driver, Executors, Cluster Manager.
2. What is the difference between `map()` and `flatMap()`?
3. What are transformations vs. actions? Give examples.
4. What is lazy evaluation in Spark and why does it matter?
5. Explain Spark Structured Streaming. How does it differ from DStream?
6. How does Spark handle data skew? What are your strategies?
7. What deploy modes does Spark support — client vs. cluster? When to use each?
8. What is the Catalyst Optimizer and how does it optimize a query plan?
9. What is the difference between `cache()` and `persist()`?
10. How does `broadcast join` work? When would you use it?

### Hadoop / HDFS

1. What is the role of the NameNode in HDFS?
2. How does HDFS handle fault tolerance (replication)?
3. What is MapReduce? How does it differ from Spark?
4. What are the components of the Hadoop ecosystem (HDFS, YARN, Hive, etc.)?

### SQL

1. Write a query to find the second highest salary without using `LIMIT` or `TOP`.
2. What is the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`?
3. Explain `LEFT JOIN` vs `INNER JOIN` with an example.
4. How do indexes work and when should you not use them?
5. What is the difference between SQL and NoSQL? When would you choose NoSQL?
6. Write a query to find duplicate records in a table.

### Data Modeling

1. What is the difference between a star schema and a snowflake schema?
2. Explain ETL vs ELT — when would you prefer each?
3. What is a fact table vs a dimension table?

---

## Round 3 — Technical Interview II (Projects + System Design)

### Project deep-dive (they will pick a project from your resume)

- "Walk me through the architecture of your data pipeline."
- "What challenges did you face — how did you debug and resolve them?"
- "Why did you choose [specific tool] over alternatives?"
- "What was the data volume? How did you scale it?"
- "What would you do differently if you redesigned it?"

### System Design (for experienced roles)

1. Design a real-time data pipeline for tracking 100M+ Jio users' app events.
2. How would you design a data warehouse for Jio's telecom billing system?
3. How do you ensure data quality in a production ETL pipeline?

---

## Round 4 — HR / Behavioural

Use the **STAR method** (Situation → Task → Action → Result):

- "Tell me about a time you handled a production incident."
- "How do you prioritize when you have multiple urgent tasks?"
- "Describe a situation where you disagreed with your manager — what did you do?"
- "Where do you see yourself in 3 years?"

> **Note:** Jio interviewers at junior/intern level sometimes ask about academic background — 10th/12th scores or JEE rank. Be prepared.

---

## Tips

- Jio operates at **massive scale** (400M+ users) — always frame answers around scalability.
- They weight DSA heavily even for Data Engineer roles — don't skip it.
- Spark Structured Streaming is a recurring topic — know it well.
- Be ready to go deep on **any** project on your resume; they probe for real understanding.
- Practice coding on a shared screen — interviewers note how you think aloud.

---

## Related

- [Spark concepts → pyspark/theory/](../pyspark/theory/)
- [Quiz practice → pyspark/quiz/](../pyspark/quiz/)
