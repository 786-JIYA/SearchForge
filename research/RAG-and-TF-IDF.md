# SearchForge Study Step 2: RAG Explained Simply

*RAG = Retrieval-Augmented Generation. Connects to Section 3 of the SearchForge LLM and Agentic AI Guide.*

---

## What is RAG?

RAG means: **look up the evidence first, then let the LLM write the answer using that evidence.**

## The open-book exam analogy

- **Plain LLM** is a closed-book exam. The student answers from memory, and if they don't remember, they guess confidently.
- **RAG** is an open-book exam. The student first finds the right page in the book, then writes the answer from that page.

For SearchForge, the "book" is your company's documents, and "finding the right page" is your inverted index + TF-IDF.

## The 3 steps

1. **Retrieve:** the user asks "Why are PostgreSQL connections timing out?" Your search engine finds the best matching passages, for example the Root Cause paragraph from Incident-104.
2. **Augment:** you paste those passages into the prompt, along with the question.
3. **Generate:** the LLM writes a clear answer based only on those passages and tells the user which sources it used.

## Tiny example

**Without RAG**, the LLM might say: *"Increase max_connections to 500."* That is a guess, and it might be wrong for your company.

**With RAG**, the prompt contains the real passage, so the answer becomes: *"The pool reached its maximum capacity (Incident-104). The guide recommends checking long-running queries first (Database-Troubleshooting.pdf)."* This is grounded in your documents, and the user can check the sources.

## Why RAG fits SearchForge

| Benefit | Why |
|---|---|
| **Fresh knowledge** | When a document is updated, you just re-index it. No retraining. |
| **Citations** | You know exactly which passage was used. |
| **Permissions** | You remove passages the user cannot see *before* building the prompt, so the LLM never sees them. |

## One thing to remember

**Better retrieval means better answers.** If your search finds the wrong passages, the LLM will write a fluent answer from the wrong evidence. That is why your TF-IDF ranking matters so much.

## Quick self-check

**Question:** If a user cannot access a private document, at which step must it be removed: retrieve, augment, or generate?

**Answer:** Before augment. Remove it before it ever goes into the prompt.

## Key terms from this step

| Term | Meaning |
|---|---|
| **RAG** | Retrieve evidence first, then generate using it |
| **Retrieval** | Finding relevant passages (your inverted index + TF-IDF) |
| **Augment** | Adding the retrieved passages to the prompt |
| **Grounding** | Tying the answer to the supplied sources |

## Next step

When exams are done: **tokens and the context window** (how much text the model can see at once, and why it affects cost and answer quality).

---

## Added: TF-IDF Explained Simply

**TF-IDF** is a way to score how important a word is to a passage. SearchForge uses it to **rank** the passages your inverted index finds.

### The two parts

**TF (Term Frequency): how often does the word appear in this passage?**

```
TF = (times the word appears in the passage) / (total words in the passage)
```

More appearances usually means the passage is more about that word.

**IDF (Inverse Document Frequency): how rare is the word across all passages?**

```
IDF = log(N / DF)
```

- **N** = total number of passages
- **DF** = number of passages that contain the word

Common words like "the" get a score near 0. Rare, specific words like "postgresql" get a high score.

**TF-IDF = TF × IDF**

### Tiny example

You have **N = 4** passages. One passage has 10 words, and "pool" appears 2 times in it. "Pool" appears in 2 of the 4 passages.

- TF = 2 / 10 = **0.2**
- IDF = log10(4 / 2) = **0.30**
- TF-IDF = 0.2 × 0.30 = **0.06**

Now take "the". It appears in all 4 passages, so IDF = log10(4 / 4) = **0**. Its TF-IDF is 0 no matter how often it appears. That is the point: common words don't help ranking.

### Why it matters for SearchForge

1. The inverted index quickly finds **candidate** passages.
2. TF-IDF **orders** them, so the most relevant passage comes first.
3. The top passages go into the RAG prompt. Better ranking means better answers.

### Remember this

TF-IDF matches **exact words only**. "Pool exhausted" will not match "ran out of connections". That is why your product definition calls it a *baseline*, not semantic understanding.

### Quick self-check

**Question:** A word appears in every single passage. What is its IDF, and what does that tell you?

**Answer:** log(N/N) = 0. The word doesn't help tell passages apart.

### Key terms

| Term | Meaning |
|---|---|
| **TF** | How often a word appears in one passage |
| **IDF** | How rare a word is across all passages |
| **TF-IDF** | TF × IDF, the relevance score for ranking |
| **Baseline** | A simple starting method you improve on later |