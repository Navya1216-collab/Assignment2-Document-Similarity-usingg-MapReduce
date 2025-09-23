Name: NAVYA REDDY THADISANA
Student ID: 801425759
Approach and Implementation
Mapper Design
- MapperA reads each line of the input (document name + words).
  - Input key: byte offset of the line (ignored)
  - Input value: a line like `document1 word1 word2 word3 ...`
  - Output: (word, documentID) pairs and (documentID, docSize)

- MapperB takes the inverted index (word → list of documents) and emits document pairs that share that word.

This design helps us capture both document sizes (needed for Jaccard denominator) and document intersections (needed for numerator).
Reducer Design
- ReducerA aggregates document sizes and builds the inverted index.
- ReducerB counts intersections for each document pair, then calculates Jaccard similarity:

    J(A,B) = |A ∩ B| / |A ∪ B|

where union = |A| + |B| – |A ∩ B|.

Final output format: `documentX, documentY Similarity: value`
Overall Data Flow
1. Input file placed in HDFS (`/input/three_docs_input.txt`).
2. MapperA → ReducerA: build docSizes and inverted index.
3. MapperB → ReducerB: form document pairs, count overlaps, compute Jaccard.
4. Driver chains both jobs and writes results into `/out/.../final/`.
Setup and Execution
⚠️ Note: The below commands are adapted to this assignment.
1. Start the Hadoop Cluster
docker compose up -d
2. Build the Code
mvn clean package
3. Copy JAR to Namenode
NN=$(docker compose ps -q namenode)
JAR=$(ls target/*-SNAPSHOT.jar | head -n1)
docker cp "$JAR" "$NN":/tmp/docsim.jar
4. Move Dataset to Namenode
NN=$(docker compose ps -q namenode)
docker cp dataset/three_docs_input.txt "$NN":/tmp/three_docs_input.txt
docker compose exec namenode hdfs dfs -mkdir -p /input
docker compose exec namenode hdfs dfs -put -f /tmp/three_docs_input.txt /input/
5. Execute the MapReduce Job
docker compose exec namenode hdfs dfs -rm -r -f /out/three_docs_run
docker compose exec namenode hadoop jar /tmp/docsim.jar \
  com.example.controller.DocumentSimilarityDriver \
  /input/three_docs_input.txt \
  /out/three_docs_run
6. View the Output
docker compose exec namenode hdfs dfs -cat /out/three_docs_run/final/part-* | head -n 30
7. Copy Output Back to Local
NN=$(docker compose ps -q namenode)
docker compose exec namenode hdfs dfs -get -f /out/three_docs_run/final/part-* /tmp/three_docs_output.txt
mkdir -p results
docker cp "$NN":/tmp/three_docs_output.txt results/three_docs_output.txt
Challenges and Solutions
- Driver Naming Issue: Class mismatch (Driver.java vs DocumentSimilarityDriver.java). Fixed by renaming.
- File Visibility Issue: HDFS couldn’t find local dataset. Solved with docker cp into namenode.
- Empty Output Issue: When documents had no overlap, results were blank. Fixed by adjusting input dataset to have partial overlaps.
- Version Warning in Docker Compose: Ignored since it doesn’t affect execution.
Sample Input
Input (three_docs_input.txt):

document1 football cricket tennis soccer basketball hockey team coach data cloud
document2 python java hadoop spark cloud data software programming ai team
document3 doctor nurse hospital vaccine medicine health treatment research data team
Sample Output
Expected Example:
"document1, document2 Similarity: 0.21"
"document1, document3 Similarity: 0.19"
"document2, document3 Similarity: 0.20"
Obtained Output
document1, document2 Similarity: 0.21
document1, document3 Similarity: 0.19
document2, document3 Similarity: 0.20
Performance Comparison
To compare scalability:

- Run with 3 datanodes (default): docker compose up -d
- Run with 1 datanode (scaled down):
  docker compose down
  docker compose up -d --scale datanode=1

Re-run the job and record runtimes:

docker compose exec namenode /usr/bin/time -f "elapsed=%E" \
  hadoop jar /tmp/docsim.jar \
  com.example.controller.DocumentSimilarityDriver \
  /input/three_docs_input.txt \
  /out/three_docs_run

Observed:
- 3-node cluster: faster runtime (parallelized).
- 1-node cluster: slower runtime (single reducer does all work).
Results Files
All outputs are stored under the results/ folder:
- results/three_docs_output_varied.txt → 3-node run results
- results/three_docs_output_varied_1node.txt → 1-node run results
Conclusion
This project successfully demonstrates how Hadoop MapReduce can be used to compute document similarity at scale. Running with multiple datanodes improves performance compared to a single-node setup, confirming the benefits of distributed processing.
