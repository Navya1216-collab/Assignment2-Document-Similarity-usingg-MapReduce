# Document Similarity using MapReduce
Assignment 2: Document Similarity using MapReduce
Name: NAVYA REDDY THADISANA
Student ID: 801425759
------------------------------------------------------------
# Approach and Implementation
Mapper Design
The Mapper takes each input line in the format (documentID, documentText).
- It tokenizes the text into words, normalizes them (lowercasing, removing punctuation).
- For each word, it emits a key-value pair:
(word → documentID)
- This builds an inverted index mapping words to the documents they appear in.
Reducer Design
The Reducer receives (word → [list of documentIDs]).
- For each word, it generates all unique document pairs containing that word.
- It counts the number of shared words (intersection size).
- Using precomputed document sizes, it calculates Jaccard Similarity:
Jaccard(A,B) = |Intersection(A,B)| / |Union(A,B)|
- Final output is:
(documentA, documentB → similarityScore)

----------------------------------------------------------------
# Overall Data Flow
1. Input dataset (documents) is uploaded into HDFS.
2. Mapper emits (word, docID) pairs.
3. Shuffle/Sort groups by word.
4. Reducer processes grouped values to calculate document pair similarities.
5. Output file contains similarity scores between all document pairs.

## Prerequisites

Before starting, ensure the following are installed and configured:

1. **Java 8+**
   - Verify installation:
     ```bash
     java -version
     ```

2. **Maven**
   - Verify installation:
     ```bash
     mvn -version
     ```

3. **Docker & Docker Compose**
   - [Docker Installation Guide](https://docs.docker.com/get-docker/)
   - [Docker Compose Installation Guide](https://docs.docker.com/compose/install/)

4. **Hadoop in Docker**
   - This project uses the Hadoop Docker setup provided in class (with Namenode + Datanodes).

---

## Project Structure

```
Assignment2-Document-Similarity-usingg-MapReduce/
├── dataset/
│   └── three_docs_input.txt
├── results/
│   ├── three_docs_output.txt
│   └── three_docs_output_1node.txt
├── src/
│   └── main/java/com/example/
│       ├── DocumentSimilarityMapper.java
│       ├── DocumentSimilarityReducer.java
│       └── controller/DocumentSimilarityDriver.java
├── docker-compose.yml
├── pom.xml
└── README.md
```

- **dataset/**: Contains the input documents.  
- **results/**: Contains outputs from both 3-node and 1-node runs.  
- **src/**: Java source files for Mapper, Reducer, and Driver.  
- **docker-compose.yml**: Cluster setup.  
- **pom.xml**: Maven build configuration.  
- **README.md**: Assignment report.  

---

## Setup Instructions

### 1. Start Hadoop Cluster
```bash
docker compose up -d
```

### 2. Build the Project
```bash
mvn clean package
```

### 3. Copy JAR to Namenode
```bash
NN=$(docker compose ps -q namenode)
docker cp target/DocumentSimilarity-0.0.1-SNAPSHOT.jar "$NN":/tmp/docsim.jar
```

### 4. Upload Dataset to HDFS
```bash
docker compose exec namenode hdfs dfs -mkdir -p /input
docker cp dataset/three_docs_input.txt "$NN":/tmp/three_docs_input.txt
docker compose exec namenode hdfs dfs -put -f /tmp/three_docs_input.txt /input/
```

### 5. Run the MapReduce Job
```bash
docker compose exec namenode hdfs dfs -rm -r -f /out/three_docs_run
docker compose exec namenode hadoop jar /tmp/docsim.jar   com.example.controller.DocumentSimilarityDriver   /input/three_docs_input.txt   /out/three_docs_run
```

### 6. View the Output
```bash
docker compose exec namenode hdfs dfs -cat /out/three_docs_run/final/part-*
```

### 7. Save Output to Local Results Folder
```bash
docker compose exec namenode hdfs dfs -get -f /out/three_docs_run/final/part-* /tmp/three_docs_output.txt
docker cp "$NN":/tmp/three_docs_output.txt results/three_docs_output.txt
```

---

## Overview

This project implements a MapReduce job to calculate **document similarity** using the **Jaccard Similarity metric**.  
- The **Mapper** builds an inverted index of words to document IDs.  
- The **Reducer** generates document pairs from shared words and computes similarity.  
- Results show how similar pairs of documents are, based on their shared tokens.  

---

## Objectives

1. Implement a Hadoop MapReduce program in Java.  
2. Understand the data flow of Mapper → Shuffle/Sort → Reducer.  
3. Deploy and run jobs in a distributed environment (multi-node vs single-node).  
4. Compare performance and correctness across setups.  

---

## Dataset

### Input (`three_docs_input.txt`)
- Document1: ~1000 words  
- Document2: ~3000 words  
- Document3: ~5000 words  

Each line starts with `documentID` followed by the content.  

---

## Assignment Tasks

### 1. Mapper Design
- Input: `(offset, line)` where line = `docID text`.  
- Output: `(word, docID)` pairs.  
- Purpose: Build inverted index for words.  

### 2. Reducer Design
- Input: `(word, [docID list])`.  
- Process: Generate document pairs and intersection counts.  
- Output: `(docA, docB, similarityScore)` using Jaccard formula.  

### 3. Execution on Cluster
- Run once on a **3-node cluster**.  
- Run again on a **1-node cluster** (scale datanode=1).  
- Save both results for comparison.  

---

## Results

### Sample Expected Output
```
document1, document2 Similarity: 0.56
document1, document3 Similarity: 0.42
document2, document3 Similarity: 0.50
```

### Obtained Output

```
---------------------------------------------------------------------------

Obtained Output
3-Node Run
document1, document2 Similarity: 0.21
document1, document3 Similarity: 0.19
document2, document3 Similarity: 0.20
1-Node Run
document1, document2 Similarity: 0.21
document1, document3 Similarity: 0.19
document2, document3 Similarity: 0.20
------------------------------------------------------------

---

## Observations

Execution Time

3-Node Run: ~8–10 seconds
1-Node Run: ~20–25 seconds
Multi-node was ~2–3× faster.
Similarity Results

Both runs produced identical similarity scores → consistency confirmed.
Cluster Utilization

3-Node Run: Distributed workload across nodes.
1-Node Run: Single node handled all tasks → slower.
Scalability

Same output in both runs shows correctness.
Larger clusters mainly improve speed, not results. 

---

## Challenges and Solutions

- **Path issues**: Fixed by moving source files under `src/main/java`.  
- **ClassNotFoundException**: Verified classpath with `jar tf` and corrected package names.  
- **HDFS overwrite errors**: Resolved using `hdfs dfs -rm -r -f` before re-running jobs.  
- **Cluster scaling**: Compared results between 3 nodes and 1 node to observe performance differences.  
