# Mappers and Reducers
Data summarization design patterns are techniques used to summarize large datasets into smaller, more meaningful representations, and the latest information on this topic includes various methods and models.we will explore the most important Spark transformations (mappers and reducers) in the context of data summarization design patterns, and examine how to select specific transformations for targeted problems.

---
### Data summarization design patterns 
-  Include in-mapper combining, top-10, min-max, and binning, which are used to reduce the size of large datasets while preserving important information.
-  Hierarchical indexing is a technique used to create summaries at each hierarchical level, allowing for efficient searching and retrieval of data. This method is useful in text search and information retrieval applications.
-  Abstractive summarization generates new text based on the original text to summarize it effectively.
-  Data can be summarized using various models, including complete, summarized, minimal, and identifier models.

### Transformation Design Pattern
These patterns are fundamental for writing efficient Spark applications!

**In-Mapper Combiner**
*  Reduces network traffic: Less data shuffled across nodes
*  Faster aggregations: Local combining is much faster
*  Better scalability: Performance improves with larger datasets
*  Use for aggregations (sum, count, average, etc.)
```python
from collections import defaultdict

def count_words_in_partition(iterator):
    """Combine within each partition first"""
    counts = defaultdict(int)
    
    # Count locally within this partition
    for word in iterator:
        counts[word] += 1
    
    # Return local counts as (word, count) pairs
    return [(word, count) for word, count in counts.items()]

# Apply in-mapper combining
word_counts = data.mapPartitions(count_words_in_partition) \
                  .reduceByKey(lambda a, b: a + b)

print(word_counts.collect())
# Output: [('apple', 3), ('banana', 2), ('cherry', 1)]
```

**Mapping Partitions**
*  Resource efficiency: Reuse expensive resources (DB connections, ML models)
*  Batch processing: More efficient for operations that benefit from batching
*  Reduced overhead: Fewer function calls and context switches
*  Use when you have expensive setup costs or when batch processing is more efficient than individual element processing.
```python
# Efficient: mapPartitions() - one connection per partition
def process_partition(iterator):
    # Create connection ONCE per partition
    connection = create_database_connection()
    
    results = []
    for element in iterator:
        result = connection.query(element)
        results.append(result)
    
    connection.close()
    return results

# Do this instead:
results = data.mapPartitions(process_partition)
```

### Transformation Performance Guide
For a large set of (key, value) pairs, using reduceByKey() or combineByKey() is typically more efficient than using the combination of groupByKey() and mapValues(), because they reduce the shuffling time.
