# Knowledge Base Setup: ADL for Grantmaking

The **Answer Program Questions** action (Action 4) is powered by knowledge grounding, not by a Flow. This guide covers when to use ADL, which ingestion method to choose, and how to set it up.

---

## When to Recommend ADL

Recommend ADL knowledge grounding when the agent needs to answer questions from **documents or knowledge articles** — not from structured records.

| Signal from user | ADL approach |
|---|---|
| "Answer questions from our program guide PDFs" | ADL with PDF retriever indexing |
| "Agents should know our eligibility policies from the policy manual" | ADL with PDF retriever indexing |
| "We have Knowledge Articles in Salesforce already" | Knowledge Article data source |
| "Constituents ask about program requirements, deadlines, FAQs" | Either, based on where the content lives |
| "Answer questions about FundingRequest status" | NOT ADL — use TrackApplicationStatus Flow instead |

**Key rule:** ADL answers from documents. Flows answer from records. Don't mix them up.

---

## Option A: PDF / Document Retriever Indexing

Best when: content is in PDFs, Word docs, or unstructured documents (program guides, policy manuals, FAQ sheets).

### Setup (delegate to `developing-agentforce` skill for full ADL provisioning)

1. **Prepare your documents**
   - Collect all grant program guides, eligibility policy documents, FAQ sheets
   - Clean up formatting: remove headers/footers that don't add meaning, ensure text is selectable (not scanned images)
   - Recommended format: PDF (text-based, not scanned)

2. **Create an Agentforce Data Library (ADL)**
   - Setup → Einstein → Agentforce Data Libraries → New
   - Name: "Grant Programs Knowledge Base"
   - Description: "Policy documents, program guides, and FAQs for grant programs"

3. **Upload documents**
   - ADL → Documents → Upload
   - Upload each program guide, policy document, FAQ
   - If using Experience Cloud, include publicly accessible documents only

4. **Configure the retriever**
   - Chunk size: 512 tokens (recommended starting point)
   - Overlap: 50 tokens (helps avoid context being split at chunk boundaries)
   - Embedding model: `text-embedding-ada-002` (standard)
   - Trigger indexing: ADL → Refresh (takes 5-30 minutes depending on document volume)

5. **Wire to the agent**
   - NGA: add `knowledge: "Grant Programs Knowledge Base"` block to the FAQ subagent
   - Legacy: add Knowledge topic in Agent Builder, reference the ADL

6. **Timing critical:** Start ADL indexing early — it runs in the background and takes time. Kick it off before writing `.agent` code so it's ready when you need to test.

---

## Option B: Salesforce Knowledge Articles

Best when: the content already lives in Salesforce Knowledge (articles have been authored and published).

### Setup

1. Confirm Knowledge is enabled: Setup → Knowledge → toggle ON
2. Ensure grant-related Knowledge Articles are published and categorized
3. In ADL → Data Sources → Add Knowledge Articles
4. Configure: select article types to index (e.g., "Grant Programs", "Eligibility FAQs")
5. Trigger indexing

**Difference from PDF retriever:** Knowledge Articles are structured (title, body, article type) so chunking and retrieval behave differently. No need to tune chunk size — articles are already organized.

---

## Option C: Web Search (for external / federal program information)

If constituents ask about federal grant programs or external funding sources, web search is the right tool — not ADL.

See [web-search-setup.md](web-search-setup.md) for the standard General Web Search Action setup.

**Rule:** ADL for your org's own documents. Web search for external/federal information.

---

## Guiding the ADL Retriever Configuration

When a partner mentions PDFs, guide them specifically:

1. **Chunking:** 512 tokens is a starting point. If your documents have long, detailed sections, increase to 1024. If questions are very specific (looking for a single fact), decrease to 256.
2. **Overlap:** 50 tokens prevents important sentences from being cut across chunks. Increase to 100 for documents with complex multi-sentence rules.
3. **Top-K retrieval:** How many chunks the agent retrieves per query. Start at 3-5. Increase if answers need more context.
4. **Test retrieval quality:** After indexing, ask the agent the kinds of questions constituents would ask. If it misses answers that are clearly in the documents, try larger chunk size or higher top-K.

---

## Knowledge Article vs PDF Retriever: When to Use Each

| Factor | Knowledge Articles | PDF Retriever |
|---|---|---|
| Content already in Salesforce | Preferred | Extra work |
| Content is in PDFs | Not applicable | Preferred |
| Content updates frequently | Articles easier to update | Re-upload PDF each time |
| Content needs versioning / approval | Articles have workflow | PDFs don't |
| Metadata filtering (by article type, category) | Yes | Limited |
| Initial setup complexity | Low (already in SF) | Medium (upload + configure) |
