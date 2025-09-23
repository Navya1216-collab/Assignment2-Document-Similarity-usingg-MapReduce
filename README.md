Assignment 2: Document Similarity using MapReduce
Name: NAVYA REDDY THADISANA
Student ID: 801425759
------------------------------------------------------------
Approach and Implementation
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
Overall Data Flow
1. Input dataset (documents) is uploaded into HDFS.
2. Mapper emits (word, docID) pairs.
3. Shuffle/Sort groups by word.
4. Reducer processes grouped values to calculate document pair similarities.
5. Output file contains similarity scores between all document pairs.
------------------------------------------------------------
Step-by-Step Execution Guide
1. Start Hadoop Cluster
```
docker compose up -d
```
2. Build the Code
```
mvn clean package
```
3. Copy JAR to Namenode
```
NN=$(docker compose ps -q namenode)
docker cp target/DocumentSimilarity-0.0.1-SNAPSHOT.jar "$NN":/tmp/docsim.jar
```
4. Upload Input Dataset to HDFS
```
docker compose exec namenode hdfs dfs -mkdir -p /input
docker cp dataset/three_docs_input.txt "$NN":/tmp/three_docs_input.txt
docker compose exec namenode hdfs dfs -put -f /tmp/three_docs_input.txt /input/
```
5. Run the MapReduce Job
```
docker compose exec namenode hdfs dfs -rm -r -f /out/three_docs_run
docker compose exec namenode hadoop jar /tmp/docsim.jar
com.example.controller.DocumentSimilarityDriver /input/three_docs_input.txt /out/three_docs_run
```
6. View the Output
```
docker compose exec namenode hdfs dfs -cat /out/three_docs_run/final/part-*
```
7. Save Output to Repository
```
docker compose exec namenode hdfs dfs -get -f /out/three_docs_run/final/part-*
/tmp/three_docs_output.txt
docker cp "$NN":/tmp/three_docs_output.txt results/three_docs_output.txt
```
8. Run on Single Datanode (1-node test)
```
docker compose up -d --scale datanode=1
```
9.Run again, saving results as
```
results/three_docs_output_1node.txt.
```
------------------------------------------------------------
Challenges and Solutions
- Path Issues (Maven build): Initially, Java files were under src/main/com/...
Maven only compiles src/main/java/...
→ Moved files under src/main/java.
- ClassNotFoundException: Fixed by verifying classes were packaged using jar tf target/*.jar.
- HDFS Output Conflicts: Hadoop jobs fail if the output directory exists.
→ Added hdfs dfs -rm -r -f before every run.
- Multi-node vs Single-node: Successfully executed with both 3 datanodes and 1 datanode to compare
results.
------------------------------------------------------------
Results
Sample Input (three_docs_input.txt)
document1 <1000 words>
document2 <3000 words>
document3 <5000 words>
Sample Output (Expected)
document1, document2 Similarity: 0.56
document1, document3 Similarity: 0.42
document2, document3 Similarity: 0.50
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
Repository Structure
Assignment2-Document-Similarity-usingg-MapReduce/
 dataset/
■  three_docs_input.txt
■ results/
■ three_docs_output.txt # 3-node run
■  three_docs_output_1node.txt # 1-node run
■ src/main/java/com/example/
■ DocumentSimilarityMapper.java
■ DocumentSimilarityReducer.java
■ controller/DocumentSimilarityDriver.java
■ README.md
■ pom.xml

-----------------------------------------------------------
## Observations

- **Execution Time**
  - 3-Node Run: ~8–10 seconds
  - 1-Node Run: ~20–25 seconds
  - Multi-node was ~2–3× faster.

- **Similarity Results**
  - Both runs produced identical similarity scores → consistency confirmed.

- **Cluster Utilization**
  - 3-Node Run: Distributed workload across nodes.
  - 1-Node Run: Single node handled all tasks → slower.

- **Scalability**
  - Same output in both runs shows correctness.
  - Larger clusters mainly improve speed, not results.

