# Testing Pattern Chains and Structured Detection Tasks with PrismML's 1-bit Bonsai 8B

I've been testing PrismML's Bonsai 8B (1.15 GB, true 1-bit weights) to see what you can actually do with pattern chaining on a model this small. The goal was to figure out where the capability boundaries are and whether multi-step chains produce measurably better results than single-pass prompting. I ran everything on a free Colab T4.

## Setup

I created an API specification for a fictional document management service, along with a permissions model (7 rules covering auth, access control, scoping, rate limits, secrets, bulk ops, and soft-delete). Then seeded 5 specific violations into the spec:

| # | Endpoint | What's wrong | Rule violated |
|---|----------|--------------|---------------|
| 1 | `GET /users` | Returns all users with no admin check | Users can only access their own resources |
| 2 | `POST /documents` | No write scope required | Write ops need explicit scope |
| 3 | `GET /auth/debug` | Returns the server signing key | No secrets in response bodies |
| 4 | `DELETE /documents/{id}` | Permanently removes data | Deletion must be soft-delete only |
| 5 | `POST /users/bulk-import` | Any authenticated user can run it | Bulk ops require admin role |

Can Bonsai find all 5 violations? Does a multi-step chain find more than a single pass?

## The chain: extract, verify, report

Three API calls give each step a focused job.

**Step 1 (Extractor):** "Pull every security-relevant claim from this API spec." The model reads the spec and outputs a JSON list of claims like "GET /users returns a list of all registered users" and "DELETE /documents/{id} permanently removes the document." It is not trying to judge anything yet, just extracting what the spec says.

**Step 2 (Verifier):** "Here are the claims. Here are the rules. For each claim, is it compliant, a violation, or incomplete?" The model receives the claims from step 1 and the permissions model. It checks each claim against each rule and classifies it.

**Step 3 (Reporter):** "Here are the verification results. Produce a severity-ranked audit report." The model receives the classified claims and groups them into critical, high, medium, and low.

## Results

| | Chain (3 calls) | Single pass (1 call) |
|---|---|---|
| Violations found | **5/5** | 4/5 |
| Wall time | **44s** | 50s |
| Tokens generated | **1,548** | 2,048 |

The chain found all 5 violations. The single pass missed V1 (GET /users returning all users without an admin check). The chain is also faster and generates fewer tokens total, because each step produces a short, focused output instead of one long response trying to do everything at once.

## What Bonsai 8B is good at

**Structured extraction.** Give it a document and a JSON schema, and it will pull out the right information. The extractor step correctly identifies every security-relevant claim in the mock API spec, including the subtle ones like "Users can view any profile" on GET /users/{id}. This worked consistently across multiple runs.

**Rule-based classification.** Give it a claim and a set of rules, and it will tell you which rules apply and whether the claim violates them. The verifier step correctly mapped violations to specific rules (RULE 2, RULE 3, RULE 5, RULE 6, RULE 7). It also correctly marked compliant endpoints as compliant.

**Structured critique.** In earlier tests, I gave Bonsai a paragraph that violated two stated architectural constraints and asked for a JSON critique. It correctly identified both violations, correctly passed the valid claims, assigned appropriate severity ratings, and produced a clean rewrite suggestion. This was the single strongest result across all testing.

**Reranking and relevance scoring.** I tested the 1.7B model (240 MB) as a reranker. Given a query and four candidate text chunks, it correctly ranked them by relevance. The ordering was right, the scores were reasonable, and it was fast (104 tok/s on Colab). This is a viable service role for retrieval pipelines.

**Change narration.** Given old and new versions of a paragraph, the 1.7B correctly described what was added, removed, and reversed with 54 tokens in 0.5 seconds.

## What Bonsai 8B is bad at

**Editing text.** This was the biggest finding. Bonsai can identify exactly what is wrong with a document, but it cannot fix it. I tested two approaches:

*Full rewrite:* Ask the model to produce a corrected version of the entire document. It reproduced the original document verbatim, violations included. Every single violation survived in the "revised" output. Both the chain and the baseline did this.

*Diff-based editing:* Ask the model to output specific find/replace pairs (exact text to remove, exact text to replace it with). Three failure modes: (1) the model targeted the wrong sections of the document, (2) the "find" strings did not match the actual document text because the model changed capitalization, dropped markdown formatting, or collapsed lines, and (3) many edits had identical find and replace strings, meaning the model described the intent to fix something without actually changing anything.

This is not a prompting issue. I tried multiple prompt formats across runs. The model can detect the problem and articulate what needs to change, but it can't produce the corrected text. At this model size, detection works, editing does not.

**Self-correction.** I asked Bonsai to review its own JSON output and fix any errors. It claimed to have made corrections, produced a table of "changes," but the before and after outputs were identical. It narrated fixes it did not make. This is the most dangerous failure mode: false confidence. A reflexion loop with Bonsai 8B wouldn't converge on better output, with the model instead saying "looks good" about unchanged text.

**Code generation.** I asked Bonsai to write a Python function for recursive document decomposition (chunking a corpus, processing each chunk, aggregating results). The output was an ugly wall of repeated regex patterns with no actual logic. It didn't implement chunking, didn't produce structured intermediates, and didn't aggregate anything. This was the single worst result: Bonsai 8B isn't suited to be the code-writing component of an agentic system.

**Cross-reference reasoning.** When given two separate JSON outputs and asked to find contradictions between them, both the 8B and 1.7B models failed. The 1.7B paired wrong claims together and missed the actual contradictions entirely. The 8B was better but still lost information during merge operations. Any task that requires comparing two contexts and reasoning about their relationship proved unreliable.

**Planning with constraints.** I asked Bonsai to plan a multi-phase research session with specific context window limits. It produced a reasonable phase sequence (correct pattern types in a logical order) but completely ignored the token budget constraints. It assigned 15,000 tokens of documents to a 4,096 token context window without comment. The structure was there, but constraint reasoning was absent.

## What the 1.7B can and can't do

I also tested Bonsai 1.7B (240 MB) for service roles:

**Works:** Extracting structured information from a single chunk of text. Scoring items against a query (reranking). Summarizing diffs between document versions. Producing metadata records from documents. These are all single-context-in, structured-output tasks.

**Doesn't work:** Comparing two chunks and finding contradictions. Any task requiring cross-reference reasoning between two inputs. The 1.7B can process one thing at a time, it can't hold two things in mind and reason about their relationship.

This maps to a clean architectural split: the 1.7B extracts at the leaves, the 8B classifies and critiques in the middle, and a larger model plans and self-corrects at the top.

## Why the chain is faster

Three API calls taking 44 seconds total versus one call taking 50 surprised me, but the math makes sense:

The single pass receives a long prompt (the full spec plus the full permissions model, about 728 tokens of input) and generates a long response (2,048 tokens) because it has to do extraction, verification, and reporting all in one shot. It tends to be verbose because it is juggling multiple objectives.

The chain splits this into three short calls. Each call has a focused prompt and generates a focused response (about 500 tokens each). The total generation is 1,548 tokens versus 2,048. Less generation means less time, even with the overhead of three separate HTTP requests.

The decomposition also means each step gets a simpler job. The extractor does not need to know the rules. The verifier doesn't need to produce a formatted report and the reporter does not need to re-read the original spec, so each model call operates on a smaller, cleaner context.

## What this means for local inference

Bonsai 8B in 1.15 GB is genuinely useful for structured detection tasks. A 3-step chain running on a free Colab GPU found the violations that a single pass missed, faster and with fewer tokens. Every intermediate step produced machine-readable JSON that you could pipe into other tools or review manually.

But the model has hard limits: it can't edit documents, it can't write code, it can't self-correct, it can't reason about relationships between separate contexts. These are capability boundaries at this parameter count and precision level that the right prompting can't sidestep.

The practical architecture that falls out of this testing uses Bonsai in a detection and classification layers pipeline with a larger model for planning, editing, and to take care of anything Bonsai flags but can't fix; the 1-bit model is an auditor, not an editor.

## Notebooks

**3-step chain:**
[Open in Colab](https://colab.research.google.com/drive/1pJWQGJgrReacsdrx7u32fSBj9dFuPP9v?usp=sharing)


---
