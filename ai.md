# AI

## Agentic AI Patterns

Short, plain-language summaries of 17 agentic patterns used in modern LLM-based systems. Source: [Building 17 Agentic AI Patterns and Their Role In Large-Scale AI Systems — Fareed Khan](https://levelup.gitconnected.com/building-17-agentic-ai-patterns-and-their-role-in-large-scale-ai-systems-f4915b5615ce) (reference implementations: [GitHub repo](https://github.com/FareedKhan-dev/all-agentic-architectures)).

1. **Reflection** — the agent writes an answer, then re-reads it as a critic and rewrites it. Cheap way to raise quality on tasks where the first draft is usually almost-but-not-quite right (writing, code review, reasoning).

2. **Tool Use** — instead of guessing, the agent calls real tools (APIs, search, calculators, DB queries) and uses the result. This is what turns a chatbot into something that can actually *do* things in the outside world.

3. **ReAct (Reason + Act)** — the agent loops "think → call a tool → look at result → think again" until done. Default pattern when the next step depends on what the previous tool call returned.

4. **Planning** — before doing anything, the agent decomposes the task into an explicit step-by-step plan, then executes it. Good when the task is long-horizon and you don't want the model to wander.

5. **Multi-Agent Systems** — several specialized agents (e.g. researcher, coder, reviewer) collaborate, each owning one role. Helps when one prompt can't hold all the responsibilities cleanly.

6. **PEV (Plan, Execute, Verify)** — a planner produces steps, an executor runs them, a verifier checks the result of every step and triggers replanning on failure. Adds a self-correction loop on top of plain Planning.

7. **Blackboard Systems** — agents don't talk directly; they read and write to a shared "blackboard" memory, and a controller decides who acts next. Useful when many independent skills need to converge on one solution.

8. **Episodic + Semantic Memory** — combines a vector store of past interactions (episodic — "what happened") with a structured knowledge store (semantic — "what is true"). Lets the agent both remember conversations and look up stable facts.

9. **Tree of Thoughts** — instead of one chain of reasoning, the agent branches into several candidate thoughts, scores them, prunes the weak ones, and expands the promising ones. Worth the extra cost on hard puzzles where one wrong early step kills the answer.

10. **Mental Loop (Simulator)** — the agent runs an action against an internal model of the world first to predict the outcome, then decides whether to actually do it. Lets it "think before acting" in environments where mistakes are expensive.

11. **Meta-Controller** — a router agent looks at the incoming task and dispatches it to the right specialist (or model). Standard pattern for production systems with a heterogeneous pool of agents/skills.

12. **Graph (World-Model Memory)** — knowledge is stored as a graph of entities and relations, so the agent can do multi-hop reasoning ("X works at Y, Y is owned by Z, therefore..."). Stronger than flat vector memory when relations matter.

13. **Ensemble** — multiple independent agents answer the same question, then an aggregator merges or votes. Reduces variance and catches single-agent blind spots, at the cost of N× compute.

14. **Dry-Run Harness** — proposed actions are simulated in a sandbox and shown for human approval before they touch production. The standard safety pattern for agents that can write to real systems (deploys, payments, emails).

15. **RLHF / Self-Improvement** — outputs are reviewed (by humans or a stronger model), and the feedback is fed back to improve future behavior, either via fine-tuning or by storing exemplars. How an agent gets better over time instead of staying static.

16. **Cellular Automata** — many small, simple agents act locally on a shared grid; useful global behavior *emerges* from their interactions, with no central planner. Niche but interesting for simulations and decentralized problems.

17. **Reflexive Metacognitive** — the agent maintains a model of *itself* — what it knows, what it's bad at, when it should give up and escalate. The basis for safe deferral ("I'm not confident, hand off to a human").

## RAG (Retrieval-Augmented Generation)

**The 30-second version.** RAG = before answering, the LLM looks things up in an external knowledge base (your docs, a wiki, a DB) and stuffs the retrieved snippets into its prompt as context. This grounds answers in real, current, domain-specific data without retraining the model — and dramatically reduces hallucinations.

**Pipeline at a glance:**

1. **Indexing (offline)** — split source documents into chunks → embed each chunk into a vector → store in a vector DB.
2. **Retrieval** — embed the user's query → find the top-K most similar chunks.
3. **Augmentation** — inject those chunks into the prompt as context.
4. **Generation** — the LLM answers using both the question and the retrieved context, ideally citing sources.

### Recommended reads (pick one or two — they overlap)

**Conceptual / "what is it":**

- [AWS — What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/) — clean, vendor-light overview. Good first read.
- [IBM — What is RAG?](https://www.ibm.com/think/topics/retrieval-augmented-generation) — slightly more depth, good diagrams.
- [NVIDIA blog — What is RAG?](https://blogs.nvidia.com/blog/what-is-retrieval-augmented-generation/) — most accessible prose; the analogy with "open-book exam" is memorable.
- [Wikipedia — Retrieval-augmented generation](https://en.wikipedia.org/wiki/Retrieval-augmented_generation) — for definitions and historical context (the original 2020 Lewis et al. paper).

**Beginner-friendly walkthrough:**

- [Michiel Horstman — RAG for Dummies (Medium)](https://michielh.medium.com/rag-for-dummies-a-beginners-guide-to-retrieval-augmented-generation-ac3348d31302) — plain-language, step-by-step.
- [SingleStore — RAG Tutorial: A Beginner's Guide](https://www.singlestore.com/blog/a-guide-to-retrieval-augmented-generation-rag/) — beginner guide with a worked example.

**Technical depth (when you actually want to build one):**

- [Pinecone — Retrieval-Augmented Generation](https://www.pinecone.io/learn/retrieval-augmented-generation/) — best technical explanation of the retrieval/embedding side.
- [Prompt Engineering Guide — RAG](https://www.promptingguide.ai/techniques/rag) — concise, links to key papers and variants (RAG-Sequence, RAG-Token, etc.).
- [DeepLearning.AI — RAG short course](https://learn.deeplearning.ai/courses/retrieval-augmented-generation/information) — hands-on course if you prefer code over prose.

### Improving RAG Performance

A baseline RAG (chunk → embed → top-K → stuff into prompt) is easy to build and almost always underwhelming in production. The fix is rarely "a better LLM" — it's almost always **better retrieval**. Optimizations cluster into three phases of the pipeline.

Sources: [Luv Bansal — Advanced RAG: Improve RAG Performance](https://luv-bansal.medium.com/advance-rag-improve-rag-performance-208ffad5bb6a) · [DataCamp — How to Improve RAG Performance: 5 Key Techniques](https://www.datacamp.com/tutorial/how-to-improve-rag-performance-5-key-techniques-with-examples).

#### Pre-retrieval (fix the index before you search it)

- **Data cleaning** — strip HTML, duplicates, typos, stop-words, irrelevant docs *before* indexing. Garbage in → garbage retrieved.
- **Chunk size & overlap tuning** — too small loses context, too big drags in noise and burns tokens. Tune per task: larger chunks for summarization, smaller for code/QA. A small overlap (e.g. ~10–20%) prevents losing facts that straddle chunk boundaries.
- **Metadata enrichment** — attach tags (date, section, author, doc type) to each chunk so you can filter *before* semantic search ("only docs from 2025", "only API reference"). Cheap and very effective.
- **Parent Document Retrieval (Small2Big)** — embed *small* chunks for precise matching, but feed the *larger parent document* to the LLM. Best of both worlds: precision in retrieval, context in generation.
- **Sentence-window retrieval** — index individual sentences, but at retrieval time pull a window of N surrounding sentences for context.
- **Knowledge-graph indexing** — store entities and their relationships as a graph alongside (or instead of) flat vectors. Lets you answer multi-hop questions ("which products did the team that shipped X also work on?").

#### Retrieval (improve the query → docs match itself)

- **Multi-query retrieval** — ask an LLM to rephrase the user's query in 3–5 different ways, retrieve for each, then union the results. Covers vocabulary mismatch.
- **HyDE (Hypothetical Document Embeddings)** — have the LLM hallucinate a plausible answer first, embed *that*, and use it for retrieval. Works because hypothetical answers are usually closer in vector space to the real source docs than the short, sparse user query is.
- **Query2Doc** — similar idea to HyDE: generate a synthetic document that would *contain* the answer, then retrieve against it.
- **Step-back prompting** — for hard questions, ask the LLM to first generate a more abstract/general version of the question ("What are the common rules of X?") and retrieve for both the original and the step-back query.
- **Multi-step (sub-question) decomposition** — break a complex query ("Compare A and B") into sub-questions, retrieve and answer each, then synthesize. Avoids overwhelming a single retrieval call.
- **Hybrid search** — combine dense (embedding) retrieval with sparse keyword retrieval (BM25). Dense is great at semantics; sparse nails exact terms (product names, error codes, IDs). Most production systems should do this by default.
- **Fine-tuned embeddings** — for specialized domains (medical, legal, internal jargon), generic embedding models miss nuance. Fine-tune on synthetic (query, doc) pairs from your own corpus.

#### Post-retrieval (clean up what you got before generating)

- **Re-ranking** — initial vector similarity is fast but coarse. After top-K retrieval, run a stronger ranker over those K to reorder them. Two flavors:
  - **Cross-encoder rerankers** (e.g. `bge-reranker`, Cohere Rerank) — small, fast, very effective.
  - **LLM-based rerankers** (e.g. RankGPT) — slower and pricier but better on nuanced/comparative queries.
- **Contextual compression** — after retrieval, run a filter that drops or summarizes parts of each chunk that aren't relevant to the query. Saves tokens and reduces distraction for the LLM.
- **RAG Fusion** — combine multi-query retrieval + reranking: generate query variants, retrieve for each, then rerank the merged set with reciprocal rank fusion. A solid default upgrade over vanilla RAG.

#### Rule of thumb for picking what to add

1. Start with **hybrid search + reranking** — biggest quality jump for the least complexity.
2. If queries are short or vague → add **HyDE** or **multi-query**.
3. If chunks feel too narrow or too noisy → switch to **parent-document** or **sentence-window** retrieval.
4. If the domain is specialized and recall is bad → **fine-tune embeddings**.
5. Always invest in **eval** (a small labelled set of question→expected-answer pairs) before tuning anything — otherwise you're optimizing blind.
