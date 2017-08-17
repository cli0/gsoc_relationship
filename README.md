# GSoC: Holmes Automated Malware Relationships (WIP)

## Introduction

![GitHub Logo](/images/architecture.png)
*Figure 1: System Architecture*

###### Overview

The purpose of this project is to develop a system capable of automatically
identifying and managing the relationships between malware objects (IP addresses,
Domains, Executables, Files etc). This system will use the analytic results as
generated and stored by Holmes-Totem and Holmes-Totem-Dynamic. The goals are:


1. Define the malware attributes necessary for relationship detection through querying.
2. Implement Machine Learning algorithms for relationship detection.
3. Implement an algorithm for aggregating and scoring the final relationships.
4. Visualize relationships for the client.

This system will perform malware relationship detection and scoring by using a range of queries and ML algorithms. We will implement and optimize some existing and new ML algorithms in order to ensure accuracy and efficiency. The whole relationship detection and rating process will go through two stages and at the end the user will receive a visual representation of the generated final relationships.

## Defining Relationships

The relationship detection process goes through two stages. The first stage is
going to extract the necessary features that define relationships and establish potential relationships (primary relationships). The second stage will utilize the data generated
by the first stage to define the final relationships and their confidence score.

From a technical standpoint, the analytics of the first stage happen independently of any user requests. All of the data generated at this stage is permanently stored in Cassandra.

###### Primary Relationships

There are 4 potential types of artefacts in our database: IP addresses, Domains, Files, and Binary Executables. All of these types can potentially have a relationship with each other.

*For example:* An executable may issue a call to a specific domain who is associated with one or more IPs, which might be in turn related to other artefacts. In this scenario we already have identified several relationships:
1. Executable <-> Domain
2. Domain <-> IP
3. (and by the transitive property of a bidirectional connection): Executable <-> IP

The whole purpose of this stage of the process is to look for meaningful primary relationships between the different artefacts based on the available analytic results, and store these relationships permanently. The following graph is a high level view of the potential relationship types between artefacts.

![GitHub Logo](/images/Relationship_Types.png)
*Figure 2: High-level view of artefact relationships*

The relationships between artefacts are defined by a (continuously-growing) predefined set of
features. The first step of relationships discovery is extracting these features of interest from the Totem results and storing connections between artefacts based on these features.

###### Final Relationships

The final relationships define how objects in the system are associated with each other. These are created by analyzing the primary relationships and determining if and how strongly objects are related. Currently, we focus on relationships among malwares, domains, and IPs.  
The final relationships consist of direct relationships and indirect relationships. The direct relationships can be retrieved directly from primary relationships, and the indirect relationships need other objects as the intermediary to transfer relationship. We seek to identify the following:  

(The column: Final relationship has same content to the column: Direct relationship, so they are merged into one column.)

 Final relationship (Direct relationship) | Indirect relationships
  ----------------------------------------- | -------------
  Malware -> Malware | \
  Malware -> Domain  | 1. Malware -> Malware -> IP </br>2. Malware -> IP -> Domain
  Malware -> IP | 1. Malware -> Malware -> IP </br> 2. Malware -> Domain -> IP  
  Domain -> Malware | 1. Domain -> Malware -> Malware </br> 2. Domain -> Domain -> Malware </br> 3. Domain -> IP -> Malware
  Domain -> Domain | 1. Domain -> Malware -> Domain </br> 2. Domain -> IP -> Domain </br> 3. Domain -> Malware -> Malware -> Domain (optional)
  Domain -> IP | 1. Domain -> Malware -> IP </br> 2. Domain -> Domain -> IP </br> 3. Domain -> IP -> IP
  IP | All IP final relationships are similar to Domain final relationships

*Table 1: Definitions for Final relationships*


## Implementation

### Offline Stage

#### Knowledge Base Generation

The data we use is provided by the Holmes-Totem and Holmes-Totem-Dynamic services.
The service results contain a variety of data, some of which is useless to the  
`Relationship` service. As a result, we first create a Knowledge Base table which
contains every feature of interest for each object we are analyzing. The creation of
this intermediary table improves query time and forgoes the need for expensive parsing
and searching for every element in the database.
The Knowledge Base entries have the following format:

```
analytics_knowledge_base
  - object_id text,
  - feature_type text,
  - feature_value blob,
  - timestamp timeuuid```

The features of interest for the analysis are predefined by the analyst and subject
to future extensions. So far, our system uses the following features as indicators
of relationships:

 feature_type  | from service
 ------------- | -------------
 imphash  | PEInfo
 pehash | PEInfo
 binary_signature | PEInfo
 domain_requests | CUCKOO
 yara_rules | Yara  

*Table 2: Definitions for feature types*

The features themselves are saved as well but before storage they are compressed
with gzip, hence the blob type. The de/compression methods we use were slightly adapted
from the following [repository](https://gist.github.com/owainlewis/1e7d1e68a6818ee4d50e).
The compression method in this repository is very fast, lightweight and incurs little
to no speed penalties based on our testing.

The routine for generating the Knowledge
Base is a batch routine that receives a list of object IDs to extract and store all
the relevant features. This routine can be initiated whenever the user wants to populate the
table with new entries. Tests on our servers have proved the routine to be fairly
efficient:

```
100 hashes -> 7s
1,000 hashes -> 7s
10,000 hashes -> 23s
```

#### Primary Relationships Generation

Primary Relationships are initial aggregations of the Knowledge Base data in order
to generate and store connections between objects based on their feature values.
More concretely, primary relationships store for each and every object in the database,
a list of identifiers for other objects with which they share a feature similarity of sorts.
The storage for Primary Relationships was inspired in part by [TitanDB]("https://www.slideshare.net/knowfrominfo/titan-big-graph-data-with-cassandra"). For each feature type, the primary relationships generation routine looks for matches, assigns
a weight to each match, and stores the result as Json array:

```
{
  {"sha256":<text>, "weight": <Double>},
  {"sha256":<text>, "weight": <Double>},
  ...
}
```
The weights for these primary relationships are defined depending on the feature type. For
 example, features like `imphash`, `pehash` and `binary_signature` are
atomic values. For an exact same match for `imphash` and `pehash`, we assign the
weight of 0.5, whereas an exact match for `binary_signature` we assign 1.0 . In a way,
these weights are arbitrary values from an analyst. After extensive statistical analysis
as well as
[literature](https://www.usenix.org/legacy/event/leet09/tech/full_papers/wicherski/wicherski_ht
ml/)
[research](https://www.fireeye.com/blog/threat-research/2014/01/tracking-malware-import-hashing
.html) we have found that objects that share these hashes have a considerable chance of being
related one way or another. We chose to only look for exact matches and not cluster
based on hashes. We find that for Big Data, clustering based on the hash can create a lot
of noise unnecessary noise for the final relationships. For features such as `domain_requests`
and `yara_rules`, we use [Jaccard Similarity](https://en.wikipedia.org/wiki/Jaccard_index) to
calculate the weight. These rules for weight definition are by no means perfect or
ideal and future work can be done for more detailed scoring methods using yara rules.

### Final Relationships Stage

#### Final Relationships Score Generator

###### Direct relationship score algorithm

The direct relationship score algorithm gives the similarity score between the object queried and the related object.
The Figure 4 below shows the design of this algorithm. The input: rel\_type scores are extracted from the primary relationship table. An algorithm tuning the weights of each score will be shown in the next paragraph. The final relationship score is the sum of (weight × rel\_type score).  
This algorithm is also used to tune the weights of each score. The loss function is the sum of these final scores when the queried objects are in the different classifcations and their final score above a threshold. Finally, minimizing the loss measure by gradient descent to get proper weights.

![GitHub Logo](/images/direct_relationship_score.png)

*Figure 4: The Schematic diagram of direct relationship score algorithm*

###### Indirect relationship score algorithm(WIP)

The indirect relationship score algorithm considers two parts and is shown in Figure 5.  
1). To a certain indirect relationship, it is consisted by direct relationships. We multiply the scores of direct relationships as indirect relationship score.  
2). To related objects, the kind of relationships (including direct relationships and indirect relationships ) is one or more. We use the sigmoid function to the sum of these relationship scores.   

![GitHub Logo](/images/indirect_relationship_score.png)

*Figure 5: The Schematic diagram of indirect relationship score algorithm*

## Storage

### Staging Phase

The following are the table schematics as generated in Cassandra.

<p align="center">
  <img src="/images/storage.png" height="330" width="400">
</p>
*Figure 3: Table View*

The table descriptions are as follows:

`analytics_knowledge_base` contains the Knowledge Base as described in the previous
section.

`analytics_mv_knowledge_base_by_feature` is a materialized view of `analytics_knowledge_base`
with feature_type as partition key. This table is used to enable efficient querying in
subroutines.

`analytics_primary_relationships` is the table where primary relationships are stored,
one entry per object.



## Visualization (WIP)

#### Web Page

###### Query Page

Query page provides the searching for hash, domain, and IP and returns relationship page.

###### Relationship Page

Relationship page is designed by D3.js and shows the relationship result.

#### Implementation
