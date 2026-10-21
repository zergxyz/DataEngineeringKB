# PySpark Architecture
PySpark is built on top of Spark’s Java API. Data is processed in Python and cached/shuffled in the Java Virtual Machine, or JVM
A high-level view of PySpark’s architecture is presented as followed: <img width="1184" height="754" alt="daws_0107" src="https://github.com/user-attachments/assets/a15aca22-7af2-40f3-aa69-01260b3d7e1b" />

And PySpark’s data flow is illustrated as followed: 
<img width="1440" height="793" alt="daws_0108" src="https://github.com/user-attachments/assets/ee99bca7-1de5-4ec1-8ed6-4b89f76e13c8" />
In the Python driver program (your Spark application in Python), the SparkContext uses Py4J to launch a JVM, creating a JavaSparkContext. Py4J is only used in the driver for local communication between the Python and Java SparkContext objects; large data transfers are performed through a different mechanism. 
