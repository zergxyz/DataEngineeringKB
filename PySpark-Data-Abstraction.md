# Spark Data Abstraction
This document summarize the 3 types of data abstraction in Apache Spark and focus main on the pySpark perspective. 

### Three Types of Abstraction
- RDD (resilient distributed dataset):
  - Low-level API
  - Denoted by RDD[T] (each element has type T)
- DataFrame (similar to relational tables):
  - High-level API
  - Denoted by Table(column_name_1, column_name_2, ...)
- Dataset (similar to relational tables):
  - High-level API (not available in PySpark)
----
### RDD Operations
Spark RDDs are read-only, immutable, and distributed. RDDs support two types of operations: transformations, which transform the source RDD(s) into one or more new RDDs, and actions, which transform the source RDD(s) into a non-RDD object such as a dictionary or array. 
<img width="1238" height="266" alt="rdd-transform-action" src="https://github.com/user-attachments/assets/1456d68b-457e-4551-ba34-6cff9aed0bf0" />
