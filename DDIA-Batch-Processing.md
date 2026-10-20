### Chapter Introduction: The Core of Batch Processing

* **Online vs. Offline Systems**: Online systems prioritize fast response times for interactive requests/queries. Batch (offline) systems process large datasets or heavy computations (e.g., model training, data transformation) where throughput matters more than low latency.
* **Immutability and Derived Data**: Batch jobs treat input data as read-only and generate output from scratch without mutating existing state.
* **Key Benefits**:
  * **Human fault tolerance**: If buggy code produces bad output, you can roll back the code and rerun the job over the immutable inputs (enabled by table-format features like *time travel*).
  * **Minimizing irreversibility**: Safe recovery accelerates feature development.
  * **Resource efficiency**: Specialized batch engines process large volumes far more cheaply than OLTP systems.


* **Modern Evolution (Highlighted in Viewport)**:
  * MapReduce is largely obsolete; workloads now run on modern engines like **Spark**, **Flink**, or cloud data warehouse query engines using high-level interfaces (dataflow APIs, SQL, DataFrames).
  * Schedulers evolved from Hadoop-specific tools (Oozie, Azkaban) to workflow orchestrators like **Airflow, Dagster, and Prefect**.
  * Storage shifted from distributed filesystems (HDFS, CephFS) toward **cloud object storage (Amazon S3)**, while systems like BigQuery and Snowflake blur the line between data warehouses and batch processing.



---

### Batch Processing with Unix Tools

* **The Analogy**: A distributed batch framework mirrors a single operating system: it requires a **filesystem** (storage), a **scheduler** (orchestrator), and **pipelines** (programs passing data to one another).
* **The Log Analysis Example**:
A common task—finding the top 5 requested URLs in an NGINX access log—can be solved in a standard shell pipeline:
```bash
cat /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -r -n | head -n 5

```


* `awk` extracts the requested URL field.
* `sort` groups identical URLs together sequentially.
* `uniq -c` aggregates consecutive duplicates into counts.
* `sort -r -n` orders the counts in descending order, and `head -n 5` keeps the top results.



---

### Key Architectural Insight: Sorting vs. In-Memory Aggregation

A custom program (e.g., written in Python) typically uses an **in-memory hash table** (`counts[url] += 1`), whereas the Unix pipeline relies on **sorting and deduplicating**.

* **In-Memory Hash Tables**:
  * **Pros**: Very fast if the distinct set of keys (the working set) fits entirely into RAM.
  * **Cons**: Fails with out-of-memory errors once the number of distinct keys exceeds available memory.


* **Sorting-Based Processing**:
  * **Pros**: Scales gracefully to datasets far larger than available RAM. GNU `sort` spills sorted chunks to disk and merges them sequentially (external merge sort). Sequential disk I/O remains fast and predictable.
  * **Cons**: Introduces disk writes and sorting overhead when datasets could have fit comfortably in memory.

* **Bridge to Distributed Systems**: The Unix pipeline is limited to a single machine's local disk and CPU. Distributed batch systems take this exact sorting principle and scale it across hundreds or thousands of nodes using a distributed sort known as a **shuffle**.
