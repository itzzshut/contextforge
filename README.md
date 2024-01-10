# contextforge

A self-hostable docs answer bot that cites its sources and abstains when it isn't sure.

[![CI](https://github.com/itzzshut/contextforge/actions/workflows/ci.yml/badge.svg)](https://github.com/itzzshut/contextforge/actions/workflows/ci.yml)

## Why

Small teams field the same handful of questions their docs already answer: how to rotate an API
key, what the refund policy is, why a webhook stopped firing. Hosted answer bots are expensive,
and a generic chatbot that confidently invents an answer is worse than none, because it erodes
trust and creates tickets of its own.

contextforge is a small RAG (retrieval-augmented generation) service you point at your own docs
(Markdown, PDF or a URL). It answers only from those docs, cites them, and when retrieval
confidence is too low it says so and logs the question to an admin queue instead of guessing.

## Features

- Ingest Markdown, PDF and web pages into section-aware, overlapping chunks.
- Answers carry numbered citations with the source document and a snippet.
- Abstains below a confidence threshold, and also when an answer cites a marker that was never
  retrieved (hallucination guard).
- Unanswered questions go to an admin review queue at `/admin`.
- Embeddable widget (one `<script>` tag), chat demo page and `/ingest` page.
- Runs with no API keys, no network and no database; opt in to OpenAI embeddings, Anthropic-generated answers
  and Postgres + pgvector through environment variables.
- Offline eval harness with a 30-case golden dataset that reports accuracy and hallucination rate.

## Architecture

```mermaid
flowchart LR
    subgraph Ingestion
        A["Markdown / PDF / URL"] --> B["Normalize to plain text"]
        B --> C["Chunk by section<br/>overlap on long sections"]
        C --> D["Embed each chunk"]
        D --> E[("Vector store<br/>pgvector or in-memory")]
    end

    subgraph Ask
        F["User question"] --> G["Embed query"]
        G --> H["Top-k similarity search"]
        H --> E
        H --> I{"Top score at or above<br/>confidence threshold?"}
        I -- No --> J["Abstain and log"]
        I -- Yes --> K["Generate answer from<br/>retrieved chunks"]
        K --> L{"Every citation marker<br/>was retrieved?"}
        L -- No --> J
        L -- Yes --> M["Streamed answer<br/>with citations"]
    end

    J --> N["Admin: unanswered<br/>questions queue"]
```

Each stage has a swappable provider chosen from the environment:

| Concern | Default (offline) | Opt-in |
| --- | --- | --- |
| Embeddings | TF-IDF over feature hashing, deterministic (`lib/embeddings.ts`) | OpenAI `text-embedding-3-small` when `OPENAI_API_KEY` is set |
| Answers | Extractive: only sentences lifted verbatim from retrieved chunks (`lib/answer-providers.ts`) | An Anthropic model via the Vercel AI SDK when `ANTHROPIC_API_KEY` is set |
| Vector store | In-memory | Postgres + pgvector via Drizzle (`lib/db/pg-vector-store.ts`) when `DATABASE_URL` is set |

## Quick start

Prerequisites: Node.js 22 and npm.

```bash
npm ci
npm run dev        # http://localhost:3000
```

The dev server seeds a bundled fictional support KB ("Nimbus Notes": billing, API keys, rate
limits, 2FA, webhooks, exports) on the first query, so the chat page works immediately. Use
`/ingest` to add your own content and `/admin` to see abstained questions.

Production build and tests:

```bash
npm run build && npm start
npm run lint
npm test           # vitest, fully offline
npm run eval       # golden dataset through the real pipeline
```

Embed the widget on any page:

```html
<script src="https://your-domain/widget.js" data-endpoint="https://your-domain"></script>
```

### Real LLM and Postgres

```bash
export DATABASE_URL="postgres://user:pass@host:5432/contextforge"
export ANTHROPIC_API_KEY="sk-ant-..."   # and/or OPENAI_API_KEY for embeddings
npm run db:generate && npm run db:migrate
npm run dev
```

## Usage

`POST /api/chat` takes `{"question": "..."}` and streams the answer text, then a
`<<<CONTEXTFORGE_META>>>` delimiter and a JSON footer with status, confidence and citations.

```bash
curl -s -X POST localhost:3000/api/chat -H 'content-type: application/json' \
  -d '{"question":"How much does the Team plan cost per member per month?"}'
```

```text
The Team plan costs $12 per member per month, billed monthly or annually, and raises the
attachment limit to 250 MB. [1] ...

<<<CONTEXTFORGE_META>>>

{"status":"answered","confidence":0.52,"citations":[{"documentId":"billing-and-plans","marker":1,...}]}
```

An out-of-scope question returns `"status":"abstained"` with no citations and is added to the
admin queue.

## How it works

- **Chunking** splits by section heading and adds overlap only on long sections, so a chunk
  usually corresponds to one self-contained topic.
- **Retrieval** embeds the question, takes the top-k chunks by cosine similarity and uses the top
  score as the confidence signal. The threshold is `DEFAULT_CONFIDENCE_THRESHOLD = 0.215` in
  `lib/retrieval.ts`.
- **Grounding check** parses the citation markers in a generated answer and abstains if any marker
  points at a chunk that was not retrieved.
- **Streaming**: the answer is framed as a text stream followed by a JSON footer
  (`lib/chat-stream.ts`), since status and citations are only known once generation finishes.

Eval results on the offline pipeline (`npm run eval`, 30 cases from `eval/dataset.json`): accuracy
80.0%, hallucination rate 0.0%, 0 false answers, 3 incorrect abstains and 3 answers missing an
expected keyword. The threshold is tuned conservatively: it prefers abstaining to answering wrongly.

## Testing

`npm test` runs 99 vitest tests offline (one Postgres integration test is skipped by default).
They cover chunking, embeddings, retrieval, citation validation, abstain logic, ingestion and the
eval harness, which asserts a 0% hallucination rate and at least 80% accuracy so a retrieval or
threshold regression fails CI. To run the Postgres test: set a real `DATABASE_URL` and run
`RUN_DB_INTEGRATION=1 npm test`.

## Project layout

```
app/          Next.js App Router: chat demo, /ingest, /admin, API routes
components/   Chat UI with source-highlighted citations
lib/          RAG pipeline: chunking, embeddings, retrieval, citations, abstain logic
lib/ingest/   Markdown / PDF / URL to normalized documents
lib/db/       Drizzle schema and pgvector-backed store
widget/       Vanilla-JS embeddable widget, bundled with esbuild to public/widget.js
eval/         Golden Q&A dataset and eval harness
fixtures/     Bundled demo knowledge base
```

## Limitations

- The default embedder is lexical, not neural. It beats keyword search but misses paraphrases with
  no shared vocabulary; set `OPENAI_API_KEY` for that.
- With the extractive provider the answer is streamed after it is computed, not token by token.
- PDF and URL network fetches are thin wrappers and are not covered by the offline tests; the HTML
  extraction logic is tested against a fixture.
- The unanswered-questions queue is in-memory and resets on restart; the vector store is persistent only with Postgres.

## License

MIT, see [LICENSE](LICENSE).

## Maintainer

Maintained by **Mariam Malik**, Software Engineer, Full-Stack Developer. For project inquiries,
contact [mariammalik5100@gmail.com](mailto:mariammalik5100@gmail.com).

The project was originally authored by [itzzshut](https://github.com/itzzshut).