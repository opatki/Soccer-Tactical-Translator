# The Tactical Translator — Project Spec

A RAG assistant that explains elite soccer tactics (Gegenpressing, False 9, Inverted Fullbacks, etc.) to casual fans by translating them into the mechanics of sports/domains they already know (basketball, video games, American football).

**Stack:** Docling (ingestion) → Vector DB (retrieval) → Langflow (RAG pipeline orchestration) → LLM (generation) → Next.js/React frontend (scaffolded with IBM Bob).

---

## 1. Goals & Success Criteria

- [ ] User can ask a soccer-tactics question and choose (or the system infers) an analogy domain.
- [ ] Answers are grounded in real tactical source material, not hallucinated.
- [ ] Answers follow a consistent "Concept → Analogy → Why it matters in-game" structure.
- [ ] End-to-end demo: ingest sources → query → grounded, analogy-based answer in chat UI.
- [ ] (Stretch) Multi-turn follow-ups keep context ("what about when they use it defensively?").

---

## 2. Architecture Overview

```
[Soccer PDFs/Glossaries]
        │  (Docling)
        ▼
[Structured Markdown/JSON chunks]
        │  (embedding model)
        ▼
[Vector Store]  ←───────┐
        │                │  retrieved context
        ▼                │
   ┌─────────────────────┴─────┐
   │   Langflow RAG Pipeline   │
   │  Retriever → Prompt       │
   │  Template → LLM →         │
   │  Analogy-Formatter        │
   └─────────────┬─────────────┘
                 │ REST/API call
                 ▼
   [Next.js/React Chat UI]  (scaffolded via IBM Bob)
                 │
              End user
```

---

## 3. Phase 1 — Data & Ingestion (Docling)

**Goal:** Turn tactical soccer knowledge into clean, chunkable text.

1. **Source collection** (aim for 4–8 sources to start):
   - Tactical glossaries (e.g., generic "soccer tactics glossary" PDFs)
   - Coaching theory PDFs (formations, pressing systems, roles)
   - Wikipedia-style explainers exported to PDF, if licensing allows
   - Your own written glossary doc for terms not well covered elsewhere (fallback ground truth)
2. **Ingest with Docling**:
   ```python
   from docling.document_converter import DocumentConverter
   converter = DocumentConverter()
   result = converter.convert("sources/tactics_glossary.pdf")
   markdown = result.document.export_to_markdown()
   ```
3. **Chunk** the markdown output — target ~300–500 token chunks, split on headings/terms so each chunk is ideally "one tactical concept."
4. **Tag each chunk** with metadata: `term`, `source_title`, `source_url_or_file`. This metadata will let the LLM cite/ground its answer and lets you debug retrieval quality.
5. **Output**: a folder of `.md`/`.json` chunk files ready for embedding, e.g. `data/chunks/false_nine.json`.

**Deliverable:** `ingest.py` script + `data/chunks/*.json`, runnable end-to-end from raw PDFs to chunked, tagged output.

---

## 4. Phase 2 — Vector Store & Retrieval

1. **Choose a vector DB** — for a hackathon/prototype scope, pick one:
   - Chroma (simplest, local, no infra)
   - FAISS (local, fast, no server)
   - Milvus/Elasticsearch (if you want something Langflow has first-class connectors for)
2. **Embed chunks** using an embedding model available in your stack (watsonx embeddings, or any Langflow-supported embedding component).
3. **Build the index**: one collection, e.g. `tactics_kb`, with metadata fields (`term`, `source`) stored alongside vectors.
4. **Sanity-test retrieval** before wiring the full pipeline: query "False 9" manually and confirm the top-k chunks are actually about False 9s, not noise.

**Deliverable:** populated vector index + a short retrieval-quality checklist (top-3 relevant for 10 sample tactical terms).

---

## 5. Phase 3 — RAG Pipeline in Langflow

Build this visually in Langflow as a flow with these components:

1. **Input** — user's natural language question, e.g. *"What's a False 9? Explain it like I'm a basketball fan."*
2. **Domain extractor** (prompt-based or simple regex/keyword step) — parses out:
   - The tactical term(s) being asked about
   - The target analogy domain (basketball, video games, American football, or "default/generic" if unspecified)
3. **Retriever** — vector search against `tactics_kb` using the extracted tactical term; return top-k chunks + metadata.
4. **Prompt template** — this is the core of the product. Suggested system prompt:

   ```
   You are a soccer tactics translator. You explain tactical concepts ONLY using
   the grounding context provided below — do not invent tactical claims not
   supported by the context.

   Grounding context:
   {retrieved_chunks}

   Task: Explain the concept "{term}" to someone who understands {target_domain}
   but not soccer. Structure your answer as:
   1. One-sentence definition of the soccer concept (grounded in context).
   2. The {target_domain} analogy (a specific player, position, or play type).
   3. Why this matters for how the match flows / what to watch for next.

   Keep it under 120 words. If the context doesn't cover this term, say so
   honestly rather than guessing.
   ```
5. **LLM node** — generation step (watsonx.ai Granite model, or whichever LLM your Langflow instance is configured with).
6. **Output formatter** — enforce the 3-part structure (definition/analogy/why-it-matters) before returning to the frontend; can be a light post-processing step or just strong prompting.
7. **API export** — Langflow flow exposed as a REST endpoint (Langflow supports this natively) that the frontend calls.

**Deliverable:** exported/shareable Langflow flow (`.json`) + a documented API endpoint (`POST /run` with `{question, domain?}` → `{answer, sources}`).

**Key design decision to lock in early:** should domain selection be (a) auto-detected from phrasing like "...like I'm a basketball fan," (b) a UI dropdown, or (c) both, with dropdown as fallback? Recommend **(c)** — infer domain from text with dropdown override, so the demo isn't fragile if the LLM misparses domain.

---

## 6. Phase 4 — Frontend (Next.js/React via IBM Bob)

1. **Scaffold** — use IBM Bob to generate boilerplate for a Next.js chat app: `create-next-app`, base layout, chat message list, input box, domain-select dropdown.
2. **Core UI components**:
   - `ChatWindow` — message history (user + assistant bubbles)
   - `MessageInput` — text box + send button
   - `DomainSelector` — optional dropdown (Basketball / Video Games / American Football / Auto)
   - `SourceFootnote` — small expandable "grounded in: [term glossary source]" under each answer, using the metadata returned from Langflow
3. **API integration** — a single client function that POSTs to the Langflow REST endpoint and renders the structured response.
4. **Nice-to-have polish**:
   - Loading state ("Reading the tactics book...")
   - Example prompt chips ("What's Gegenpressing?", "Explain inverted fullbacks like I play FIFA/EA FC")
   - Light soccer-themed styling (pitch-green accent, jersey-style chat bubbles)

**Deliverable:** running Next.js app, `npm run dev`, chat UI hitting the live Langflow endpoint.

---

## 7. Phase 5 — Testing & Grounding QA

Before calling it done, validate against a fixed test set of ~10–15 terms:

| Term | Domain | Grounded? (Y/N) | Notes |
|---|---|---|---|
| False 9 | Basketball | | |
| Gegenpressing | Video games | | |
| Inverted fullback | American football | | |
| Low block | Basketball | | |
| Overlap/underlap | Video games | | |
| ... | | | |

For each: confirm (a) the analogy is actually apt, (b) the soccer claim is traceable to a retrieved chunk, (c) no hallucinated stats/names.

---

## 8. Suggested Build Order / Timeline

1. **Day 1:** Collect sources, build Docling ingestion script, produce chunked KB.
2. **Day 2:** Stand up vector store, validate retrieval quality, sketch the Langflow flow.
3. **Day 3:** Wire full Langflow RAG pipeline, lock prompt template, expose REST endpoint.
4. **Day 4:** Scaffold frontend with IBM Bob, wire to API, style chat UI.
5. **Day 5:** QA pass against test-term table, polish demo script/examples, record demo.

---

## 9. Open Decisions (resolve before or during build)

- Which LLM/embedding backend is actually available to you (watsonx.ai? OpenAI? local)?
- Vector store choice (Chroma vs FAISS vs Milvus) — mostly a "what does Langflow make easiest" call.
- Auto domain-detection vs dropdown-only (recommend hybrid, see Phase 3).
- How much source material licensing matters for your submission (prefer public glossaries/your own written glossary over scraped copyrighted coaching content).

---

## 10. Stretch Goals (post-MVP)

- Multi-turn conversation memory (follow-up questions reference prior term).
- Let users pick their *own* comparison domain freely ("explain it like I play chess").
- Visual diagram overlay (simple formation graphic) alongside the text answer.
- "Quiz me" mode: bot asks the user to explain a tactic back using their chosen analogy.
