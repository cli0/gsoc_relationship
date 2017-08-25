# GSoC: Holmes Automated Malware Relationships

## Introduction

![GitHub Logo](/images/architecture.png)
*Figure 1: System Architecture*

#### Overview

The purpose of this project is to develop a system capable of automatically identifying and managing the relationships
between malware objects (IP addresses, Domains, Executables, Files etc). This system uses the analytic results as
generated and stored by Holmes-Totem and Holmes-Totem-Dynamic. The goals are:

1. Define the malware attributes necessary for relationship detection through querying.
2. Implement Machine Learning algorithms for relationship detection.
3. Implement an algorithm for aggregating and scoring the final relationships.
4. Visualize relationships for the client.

This system performs malware relationship detection and scoring by using a range of queries and ML algorithms.
We implement and optimize some existing and new ML algorithms in order to ensure accuracy and efficiency. The whole
relationship detection and rating process goes through two stages and at the end the user receives a visual
representation of the generated final relationships.

#### Technology

This project uses Apache Spark 2.0 to perform its analytics and Apache Cassandra 3.10 for its backend. The Machine Learning
components of the project use the library [TensorFlowOnSpark](https://github.com/yahoo/TensorFlowOnSpark).

#### Defining and Modeling Relationships

[This](https://github.com/HolmesProcessing/gsoc_relationship/tree/master/primary_relationships) is the Spark Application responsible for generating
the Knowledge Base and Primary Relationships for this 
project. This application performs batch analytics and storage. To run the application,
please enter your configurations directly in the ```SparkConfig.scala``` file.

***Warning***: Right now, I haven't found a way to read the typesafe config file 
during the spark routine. This problem should be fixed in the near future.

To test the application, create a fat jar by running ``` sbt assembly ```. 
Afterwards, you can run the application on your spark cluster using:

```> /path/to/spark-submit --class com.holmesprocessing.analytics.relationship.PrimaryRelationshipsApplication relationship-assembly-1.0.jar```

