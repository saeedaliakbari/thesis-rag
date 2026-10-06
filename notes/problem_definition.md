# Problem Definition

**Thesis:** Lightweight Post-Retrieval Refinement for Reliable Retrieval-Augmented Question Answering
**Status:** Week 1 draft (2026-10-06). Contains observations and hypotheses only, no results.

---

## 1. Problem

A standard Retrieval-Augmented Generation (RAG) system retrieves a fixed number of documents for a question and passes all of them to a language model (the generator). This fails in three common situations:

- the first-stage retriever returns documents that are semantically similar to the question but do not contain the answer;
- the retrieval scores are weak, so even the top-ranked documents are unreliable;
- irrelevant documents occupy high ranks and push relevant ones down.

In each case, weak evidence reaches the generator. This can lower answer quality and makes the context longer than necessary.

This thesis studies a **lightweight second stage placed between retrieval and generation**. The stage has two jobs:

1. **Reorder** the retrieved candidates with a small cross-encoder reranker.
2. **Select** only the candidates that are reliable enough to keep, using a simple calibration model on retrieval and reranker scores. When no candidate is reliable, the system marks the question as low confidence.

The thesis is an empirical systems study. It does not claim that reranking itself is new. The generator is kept fixed, so measured differences come from the retrieval/refinement stage and not from differences between language models. All components must run on a normal laptop or free Google Colab.

## 2. Research Questions

- **RQ1.** Does a small cross-encoder reranker rank the required WixQA knowledge-base articles higher than dense retrieval alone?
- **RQ2.** Can a lightweight score-based selection step reject weak or irrelevant evidence without throwing away too much useful evidence?
- **RQ3.** How does post-retrieval refinement change retrieval quality, context size, latency, and end-to-end answer quality?
- **RQ4.** How robust is the refinement stage when the candidate list contains extra irrelevant documents or hard negatives?

## 3. Dataset and the Meaning of `article_ids`

**Knowledge Base (KB).** The KB is the `wix_kb_corpus` configuration of WixQA. It is a snapshot of 6,221 Wix Help Center articles taken on 2024-12-02 (dataset revision `d662dc42479c14e202eccd832f8c4b66a035c4cc`). By type, it contains 4,109 `article`, 2,049 `feature_request`, and 63 `known_issue` documents. Each document has a unique `id`; all 6,221 IDs were verified to be unique.

**`article_ids`.** Every question in the three QA subsets has a field `article_ids`. It lists the IDs of the KB documents that are required to answer that question (the *gold* articles). In all three subsets, every gold ID was found in the KB (0 missing).

**Why this matters.** Because the required evidence is identified by ID, retrieval can be evaluated directly. For each question, we check whether and at which rank the gold IDs appear in the retrieved list, and compute Recall@k, Hit@k, MRR@k, and nDCG@k. This does not require judging the generated answer. It also lets us separate *retrieval failures* (the gold article was never retrieved or was filtered out) from *generation failures* (the gold article was supplied but the answer is still wrong).

**Consequence for implementation.** Every preprocessing step must keep the KB `id` attached to each article and to every chunk derived from it. Otherwise retrieved text cannot be mapped back to gold articles and evaluation is impossible.

## 4. Data Split Discipline

| Subset | Size | Role |
|---|---|---|
| WixQA-Synthetic | 6,221 | Development: internal train/validation split (seed 42); choose models, top-k, features, thresholds |
| WixQA-ExpertWritten | 200 | Primary final evaluation only |
| WixQA-Simulated | 200 | Secondary final evaluation only |

**Why ExpertWritten and Simulated stay untouched.** If any decision (model, candidate size, feature set, threshold) is made by checking performance on these sets, the reported numbers no longer measure how the method generalizes. They would measure how well the method was fitted to those specific 200 questions. With only 200 questions per set, a few rounds of trial and error are enough to inflate results. These two sets also come from real support tickets and real user–expert chats, so they are the closest available estimate of real-world performance. They will be used only after the full method and its thresholds are frozen.

Note: the `datasets` library loads every configuration under the split name `train`. This is a library default and does not mean these sets are training data.

## 5. Differences Between the Subsets (Week 1 measurements)

Lengths are whitespace word counts (from `results/dataset_summary.csv`).

| | Synthetic | ExpertWritten | Simulated |
|---|---|---|---|
| Questions | 6,221 | 200 | 200 |
| Questions with >1 gold article | 0% | 26.0% | 13.5% |
| Gold IDs total / distinct | 6,221 / 6,221 | 258 / 209 | 230 / 160 |
| Question words, mean / median | 14.1 / 14 | 19.9 / 18 | 12.3 / 12 |
| Answer words, mean / median | 104.9 / 75 | 149.5 / 121.5 | 43.7 / 44 |

Qualitative observations from inspected examples:

- **Synthetic:** clean, formal questions, generated from one article each. Exactly one question per KB article. Questions often closely match the article title.
- **ExpertWritten:** real support-ticket language. Longer questions, typos, sometimes written in third person ("User is inquiring…"). Long, step-by-step reference answers. About a quarter need more than one article.
- **Simulated:** short, informal, often lowercase questions ("how to take my site off live"). Short, concise answers.

**KB document length:** mean 376.5, median 201, 95th percentile 1,216, max 6,653 words. The distribution is strongly right-skewed. `article` documents are long (median 409 words), while `feature_request` (median 56) and `known_issue` (median 30) documents are short.

## 6. Observations and Open Questions (not conclusions)

These points are recorded for later stages and must be tested before any claim is made.

1. **Title already in contents.** In all 6,221 documents, `contents` begins with the title. *Week 2 question:* index `contents` only, or `title + contents`?
2. **Title-only documents.** 29 documents (28 `known_issue`, 1 `article`) contain only their title. They are gold for 29 Synthetic questions, whose reference answers may not be supported by KB text. They are gold for 0 ExpertWritten and 0 Simulated questions. They will **not** be removed from the KB (removal would make gold articles unreachable); they will be flagged.
3. **Text-extraction artifacts.** Missing spaces between sentences and headings (e.g., `placeYou`, `Before you begin:First`). *Week 2 question:* minimal cleaning of `contents`, or re-extraction from `html_content`?
4. **Truncation risk.** Many `article` documents are long relative to typical MiniLM input limits. *Week 2 question:* full articles or chunks? To be decided after measuring token lengths with the chosen model's tokenizer.
5. **Single-gold tuning vs. multi-gold evaluation.** Tuning data (Synthetic) always has one gold article, while up to 26% of evaluation questions need several. A selection threshold tuned on Synthetic may favor keeping too few documents. To be checked in Week 5.
6. **Hypothesis: Synthetic is easier for retrieval** because its questions overlap lexically with article titles. To be tested in Week 3 by comparing dense-retrieval metrics across subsets.
7. **Feature-request template.** `feature_request` documents (one third of the KB) appear to share a fixed template, which may make them hard negatives for each other. To be checked in Week 3 error analysis.
