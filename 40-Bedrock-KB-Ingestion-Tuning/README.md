# Bedrock Knowledge Base Retrieval Tuning Kit: Fix Wrong Answers and Shrink OpenSearch Serverless Bills

Audit, re-chunk, filter, and score an existing Amazon Bedrock Knowledge Base against a 50-question golden set—retrieval accuracy clears a measured 85% bar and the idle OpenSearch Serverless bill drops by up to 75%.

## The Problem

Your Bedrock Knowledge Base works—technically. It was created with console defaults: fixed 300-token chunks with 20% overlap, top-5 retrieval, zero metadata, and an OpenSearch Serverless collection idling at 2 indexing + 2 search OCUs, which is $700.80/month at $0.24 per OCU-hour before a single query is served. Then users start reporting wrong answers: the 2023 version of the refund policy gets cited instead of the 2026 one, a question about the EU contract pulls chunks from the US contract, and a procedure that spans three paragraphs comes back as one orphaned sentence with no context.

The instinctive fix is to swap the generation model. That is the wrong layer. The model cannot cite text it never received—when the answer is wrong, the retrieved chunks were wrong first. The right fixes are all retrieval-side: chunking strategy, metadata filtering, search type, and result count. But chunking configuration is immutable on an existing data source, nobody wrote down what the current settings even are, and without an eval set every change is a guess you cannot verify. This kit tunes the Knowledge Base you already have, in place, with a measurable accuracy bar—no rebuild, no new framework, no "RAG tutorial part 2."

## Who This Is For

- Teams with a production-ready Bedrock Knowledge Base whose answers degraded as the corpus grew past a few hundred documents
- Startups paying $700+/month for an OpenSearch Serverless collection that serves a dev workload
- Engineers who changed the generation model twice, saw no improvement, and suspect retrieval is the real problem
- Anyone who needs to prove—with a number—that a RAG fix actually worked before shipping it

## How to Use

1. Copy the System Prompt below into `kb-tuning.md` and replace every `[BRACKETED]` placeholder with your Knowledge Base ID, region, S3 location, symptom, and monthly budget.
2. Open Kiro CLI or Claude Code in a workspace with AWS credentials for the account that owns the Knowledge Base.
3. Paste the filled prompt. The assistant audits the live KB first with read-only `aws bedrock-agent` calls—approve those before anything else.
4. When asked, paste 10–15 real failed user questions. These seed the golden eval set; the assistant generates the remainder from your corpus.
5. Review `chunking-plan.md` before approving re-ingestion—a new data source re-embeds the corpus (cheap, but deliberate).
6. Run `python eval_retrieval.py` after each tuning cycle. Stop when `eval-results.csv` shows hit@5 ≥ 0.85.

**Prerequisites**

- **Required Access:** IAM permissions for `bedrock:GetKnowledgeBase`, `bedrock:ListDataSources`, `bedrock:GetDataSource`, `bedrock:CreateDataSource`, `bedrock:StartIngestionJob`, `bedrock:Retrieve`; `aoss:APIAccessAll` on the vector collection; `s3:GetObject`/`s3:PutObject` on the corpus bucket
- **Recommended Background:** How Bedrock KB ingestion works (parse → chunk → embed → index), basic vector-search concepts, enough Python to run a script
- **Tools Required:** AWS CLI v2.15+, Python 3.12 with boto3 1.34+, AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); in Kiro, the `use_aws` tool covers the CLI calls

**Key Parameters:** Chunk size (300 tokens fixed / 1500-parent + 300-child hierarchical), overlap (20% fixed / 60 tokens hierarchical), semantic breakpoint percentile (95), numberOfResults (5), search type (SEMANTIC vs HYBRID), eval bar (hit@5 ≥ 0.85, MRR ≥ 0.60, 50 questions, max 3 cycles), OCU floor (0.5/0.5 dev vs 2/2 prod), budget ceiling ($250 default)

**Troubleshooting:** If the assistant attempts to update chunking on the existing data source and the API rejects it, that is expected—`chunkingConfiguration` is immutable after creation. The kit's runbook creates a parallel data source, re-ingests, and deletes the old one only after the eval bar passes.

## System Prompt

```markdown
# Bedrock Knowledge Base Retrieval Tuning Request

## Project Overview
I have an EXISTING Amazon Bedrock Knowledge Base that returns wrong or irrelevant chunks, misses the right document, or costs more than it should. Act as a senior search-relevance engineer and tune it in place. Do not propose rebuilding the stack, switching frameworks, or swapping the generation model until retrieval itself is measured and fixed.

## Current State (filled in by me)
- Knowledge Base ID: [KB_ID] in region [REGION]
- Vector store: [OpenSearch Serverless | Aurora PostgreSQL pgvector | Pinecone]
- Data source: [s3://BUCKET/PREFIX], roughly [DOC_COUNT] documents of type [PDF | HTML | Markdown | mixed]
- Embedding model: [amazon.titan-embed-text-v2:0 | unknown]
- Primary symptom: [wrong chunks retrieved | right doc never in top 5 | answers cite stale versions | monthly cost too high]
- Monthly budget ceiling for the whole KB stack: [$BUDGET]

## Ground Rules — Verify Reality First
1. Before recommending anything, inspect the live system: run aws bedrock-agent get-knowledge-base, list-data-sources, and get-data-source. Record the ACTUAL chunkingConfiguration, embedding model and vector dimensions, vector store ARN, and the status of the most recent ingestion job. Never assume console defaults are in effect.
2. Treat chunkingConfiguration and the embedding model as immutable on existing resources. A chunking change requires a NEW data source plus full re-ingestion; an embedding-model change requires a new Knowledge Base and index. State the cost of each before proposing it.
3. Confirm region availability of any reranker or parsing model using aws_search_documentation / aws_read_documentation before recommending it, and cite the page consulted.
4. Never delete the existing data source until its replacement passes the eval bar in Requirement 3.

## Detailed Requirements

### 1. Chunking — choose from this decision table and justify
- Fixed-size (default: 300 tokens, 20% overlap): uniform prose, FAQs, policies, support tickets. Keep it unless the eval proves otherwise; cheapest to re-ingest.
- Hierarchical (parent 1500 tokens / child 300 / 60-token overlap): long structured documents — manuals, contracts, runbooks — where the matched sentence needs surrounding context to be answerable.
- Semantic (maxTokens 300, bufferSize 1, breakpointPercentileThreshold 95): dense multi-topic documents where fixed splits cut mid-thought. Warn that it makes extra embedding-model calls during ingestion.
- No chunking: corpus pre-chunked upstream, one chunk per file.
Recommend exactly ONE strategy per data source, with a re-ingestion runbook and the projected embedding cost at $0.00002 per 1,000 input tokens (Titan Text Embeddings V2).

### 2. Metadata Filtering for Retrieval Precision
- Write a Python script that emits a sidecar [filename].metadata.json next to every S3 object, in the format {"metadataAttributes": {"doc_type": "...", "department": "...", "year": 2026, "status": "current"}} — 3 to 5 filterable attributes derived from the S3 key and document content.
- Produce a retrievalConfiguration JSON using vectorSearchConfiguration with filter blocks (equals, in, andAll) that scope every production query to status = current, plus one HYBRID overrideSearchType variant for keyword-heavy corpora on OpenSearch Serverless.
- Keep numberOfResults at 5 unless the eval justifies more; show the token math (each extra 300-token chunk adds ~300 input tokens to every generated answer).

### 3. Retrieval Eval Loop — the accuracy bar
- Build golden_questions.jsonl: 50 questions, each with the expected source S3 URI and expected answer span. Seed 15 from the real failed queries I paste in; generate the remaining 35 from the corpus.
- Write eval_retrieval.py: call bedrock-agent-runtime Retrieve at k=5 for all 50 questions, compute hit@5 (expected URI appears in top 5) and MRR, and append per-question rows to eval-results.csv.
- The bar: hit@5 >= 0.85 AND MRR >= 0.60. Run the baseline FIRST. Then run at most 3 tuning cycles, changing exactly ONE variable per cycle (chunking, metadata filter, k, hybrid search, reranker) and re-running the full 50.
- Recommend a Bedrock reranker (Amazon Rerank 1.0, $1.00 per 1,000 queries) only if the bar is still unmet after cycle 2.

### 4. Cost Controls
- Read the OpenSearch Serverless capacity settings. If this is a dev/test workload, set the floor to 0.5 indexing + 0.5 search OCU (~$175/month at $0.24 per OCU-hour) instead of the 2+2 production default (~$700/month). Flag the redundancy trade-off.
- Output a cost-projection table: vector store, embedding re-ingestion, reranker (if adopted), versus [$BUDGET].

## Deliverables
1. kb-tuning-report.md — current-state audit, diagnosis, chosen fixes
2. chunking-plan.md — decision-table verdict plus re-ingestion runbook
3. generate_metadata.py and retrieval-config.json
4. golden_questions.jsonl, eval_retrieval.py, eval-results.csv (baseline + every cycle)
5. cost-projection.md

## Acceptance Evidence
Close with a before/after table (hit@5, MRR, p95 Retrieve latency, projected monthly cost), the exact AWS CLI commands you ran to verify live state, and a one-line PASS/FAIL against the 0.85 bar. The baseline eval must be runnable within 30 minutes of starting. Align every recommendation with the Well-Architected Cost Optimization and Performance Efficiency pillars. Output the files directly, without preamble.
```

## What You Get

1. **kb-tuning-report.md** — audit of the live KB (actual chunking config, embedding model, ingestion status) and a diagnosis that names the retrieval failure mode
2. **chunking-plan.md** — the decision-table verdict (fixed / hierarchical / semantic / none) with a step-by-step re-ingestion runbook and projected embedding cost
3. **generate_metadata.py** — emits `<file>.metadata.json` sidecars with 3–5 filterable attributes for every object in the corpus prefix
4. **retrieval-config.json** — production `vectorSearchConfiguration` with `andAll` metadata filters and a HYBRID variant
5. **golden_questions.jsonl** — 50-question eval set (15 from real failures, 35 corpus-generated)
6. **eval_retrieval.py + eval-results.csv** — the scored loop: baseline plus up to 3 one-variable tuning cycles
7. **cost-projection.md** — OCU, embedding, and reranker math against your budget ceiling

## Example Output

```
RETRIEVAL EVAL — kb-9XKD2LP4QR (us-east-1), 50 questions, k=5
Baseline (fixed 300/20%, no filters)..... hit@5 0.62  MRR 0.41  FAIL
Cycle 1: hierarchical 1500/300/60 ........ hit@5 0.78  MRR 0.52  FAIL
Cycle 2: + filter status=current (andAll). hit@5 0.88  MRR 0.64  PASS
Cost: OCU floor 2+2 -> 0.5+0.5 ........... $700.80/mo -> $175.20/mo
Re-ingestion (8,400 docs, ~38M tokens) ... $0.76 one-time
Verdict: PASS — ship retrieval-config.json; reranker not required.
```

## AWS Services Used

Amazon Bedrock Knowledge Bases, Amazon Bedrock (Titan Text Embeddings V2, Amazon Rerank 1.0), Amazon OpenSearch Serverless, Amazon S3, AWS IAM, AWS CLI

## Well-Architected Alignment

- **Performance Efficiency:** chunking and search-type choices are selected by measured hit@5/MRR on a golden set, not by intuition; one variable changes per cycle so causality is provable
- **Cost Optimization:** OCU floor right-sized to workload ($700.80 → ~$175/month for dev), numberOfResults justified with per-query token math, reranker adopted only when the accuracy bar demands it
- **Operational Excellence:** every change ships with a runbook, the eval loop is a repeatable script, and acceptance evidence (CSV + PASS/FAIL line) is a deliverable
- **Reliability:** the old data source is never deleted until its replacement passes the bar—rollback is always one config swap away
- **Security:** metadata filters scope retrieval to authorized, current documents; IAM access is enumerated to the exact `bedrock:*` actions required

## Cost Notes

- **OpenSearch Serverless is the bill.** $0.24 per OCU-hour (us-east-1): the 2 indexing + 2 search default runs $700.80/month even at zero traffic; the 0.5 + 0.5 dev floor runs ~$175/month. Managed storage adds $0.024/GB-month.
- **Tuning cycles are nearly free.** Re-embedding 8,400 documents (~38M tokens) with Titan Text Embeddings V2 at $0.00002 per 1,000 tokens costs about $0.76 per full re-ingestion.
- **The eval loop costs cents.** Retrieve calls against your own vector store carry no per-request Bedrock charge; 4 full runs of 50 questions with Amazon Rerank 1.0 enabled would add $0.20 total.
- Net effect for a typical dev KB: ~$525/month saved on OCUs, under $5 in one-time tuning spend.

## Troubleshooting

- **Re-ingestion completes but hit@5 barely moves** — Cause: the old data source is still active in the KB, so stale duplicate chunks outrank the new ones. Fix: delete (or stop syncing) the old data source after the new one passes, or add a `source_version` metadata filter during the overlap window.
- **ValidationException when applying a metadata filter** — Cause: the `.metadata.json` sidecars were uploaded after the last sync, so the attributes were never indexed. Fix: run `aws bedrock-agent start-ingestion-job`, wait for COMPLETE, retry the filtered query.
- **HYBRID search type rejected** — Cause: the configured vector store does not support hybrid search (OpenSearch Serverless does). Fix: stay on SEMANTIC, or plan a store migration in a separate change.
- **Bill is still ~$700/month with near-zero traffic** — Cause: the collection kept the 2+2 OCU redundancy default. Fix: set the capacity floor to 0.5 indexing + 0.5 search OCUs for dev/test (~$175/month).
- **Semantic chunking ingestion is slow and pricier than estimated** — Cause: it invokes the embedding model on sentence windows during ingestion, on top of the final chunk embeddings. Fix: reserve semantic chunking for corpora that fail the eval bar under both fixed and hierarchical.
