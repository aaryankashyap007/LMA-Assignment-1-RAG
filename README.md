# RAG Assignment: Baseline vs. Re-Ranking

## 1. Architecture Comparison

The fundamental difference between a Bi-Encoder and a Cross-Encoder lies in how they process and compare text:

*   **Bi-Encoder (Baseline):** A Bi-Encoder processes the query and each document chunk independently. It encodes them into separate dense vectors, and their relevance is calculated using a simple metric like cosine similarity or inner product. Because the chunk representations are independent of the query, all chunk vectors can be precomputed, stored in a vector database (like FAISS), and retrieved extremely fast.
*   **Cross-Encoder (Advanced Re-Ranking):** A Cross-Encoder feeds the query and the chunk *together* through the transformer model simultaneously. This allows for joint self-attention, meaning every token in the query can attend to every token in the chunk. This deep, token-level interaction is why a Cross-Encoder successfully captures complex semantic nuances, such as negations ("not") or constraints ("except"), which independent vectors often blur. However, this joint processing makes it computationally expensive since nothing can be precomputed.

## 2. Qualitative & Rank Shift Analysis

To evaluate the practical benefits of the Cross-Encoder, I stress-tested the pipelines using queries designed to expose retrieval weaknesses. 

**Query 1: "What is the limit on how many Nominated Members a Council can have?"** (Exceptions & Corner Cases)
*   **Rank Shift:** The vital chunk containing the exception (Chunk 16) moved from **Rank #6** (Baseline) to **Rank #2** (Re-ranked).
*   **Answer Comparison:** Because Chunk 16 sat outside the Top 3 during the baseline retrieval, Answer 1 omitted a crucial clause, stating a hard limit of "one-half of the number of elected members." Answer 2, benefiting from the re-ranked chunks, correctly identified the limit but accurately appended the "except when" condition (that the cap may be exceeded to meet Part IV composition requirements). 

**Query 2: "How long can the General Secretary speak in their own defence at a dismissal hearing?"** (Buried Chunk)
*   **Rank Shift:** The vital chunk detailing the time limit (Chunk 84) moved from **Rank #7** (Baseline) to **Rank #2** (Re-ranked).
*   **Answer Comparison:** The baseline retrieval pulled in irrelevant rule chunks, causing Answer 1 to misstate the rule by incorrectly applying a 10-minute defence time meant for a different role/context. The re-ranking pipeline successfully promoted the correct chunk (15 minutes). Interestingly, Answer 2 still stated it could not definitively answer the question. This occurred because Chunk 84 starts mid-section, and the specific heading naming the "General Secretary" resides in a neighboring chunk. 

*Note on limitations:* During testing, two candidate queries completely failed because their vital chunks never reached the Top 10 in the baseline retrieval. Re-ranking cannot fix this; a Cross-Encoder can only promote chunks that the baseline successfully retrieves in the first stage.

## 3. Latency Profiling

The following table summarizes the runtime latency of the baseline retrieval versus the re-ranking pipeline:

| Metric | Time |
| :--- | :--- |
| Baseline retrieval (avg) | 7.12 ms |
| Re-ranking 10 pairs (avg) | 26.48 ms |
| Pipeline 2 total (avg) | 33.58 ms |
| LLM call, Answer 1 (avg) | 2.46 s |
| LLM call, Answer 2 (avg) | 3.26 s |

**Analysis:** The data shows that executing a Cross-Encoder over 10 pairs (26.48 ms) costs several times more than searching the entire precomputed Bi-Encoder index (7.12 ms). Additionally, there is a performance difference depending on batching: processing pairs sequentially took roughly 2.6 ms per pair, while batching (as done in the scaling tests) reduced this to 1.66 ms per pair. Ultimately, however, both retrieval latencies are entirely dwarfed by the generation time of the LLM call itself (~2.5 to 3.2 seconds).

## 4. Scaling Question

**Why not simply run a Cross-Encoder across the entire document collection from the very start?**

Because the Cross-Encoder requires the query and the chunk to be processed together, chunk scores depend entirely on the specific query. Consequently, nothing can be precomputed or indexed offline. The computational cost and time grow linearly with every new chunk added to the collection. 

To illustrate why a two-stage design is mandatory, consider the cost of running a batched Cross-Encoder (at 1.66 ms per pair) across growing collection sizes for a single query:

| Collection Size | Cross-Encoder on ALL Chunks (per query) | Cross-Encoder on Top-10 Only |
| :--- | :--- | :--- |
| 123 chunks | 0.2 seconds | 16.6 ms |
| 10,000 chunks | 16.6 seconds | 16.6 ms |
| 100,000 chunks | 165.5 seconds | 16.6 ms |
| 1,000,000 chunks | ~28 minutes (1,655 s) | 16.6 ms |
| 10,000,000 chunks | ~4.6 hours (16,552 s) | 16.6 ms |

Running a Cross-Encoder on an entire database of 1 million chunks would take nearly half an hour for a single user question. By using a two-stage pipeline, we rely on the Bi-Encoder's speed to filter the database down to the Top 10, reducing the Cross-Encoder's execution time to a constant 16.6 ms, regardless of how large the total document collection becomes.