# Node.js Vector Query Debug: Model Mismatches Return Irrelevant Results

TL;DR: When vector search returns plausible but irrelevant help-center listings, stop tuning top-k first. Verify that the query and every stored vector came from the same embedding model, model revision, preprocessing path, and output dimension. A dimension check catches the loud failure. A fingerprint stored beside each vector catches the quieter one: two models can emit arrays of equal length that occupy incompatible vector spaces. Re-embed a tiny, labeled slice, compare exact expected hits, then rebuild the collection only after that test passes.

That order protects the real constraint in customer support: retrieval quality versus latency. Raising candidate counts can hide weak ranking while making every request slower. For a one-person SaaS shipping weekly, the useful fix is a small invariant at the ingestion boundary, not another dashboard or a longer prompt. Outsource undifferentiated storage if that buys time, but keep this invariant in application code.

## Why does a vector query return irrelevant results?

A vector dimension is only an array length. It does not identify the coordinate system that gave the numbers meaning. If an old document index and a new query encoder both produce 768 values, a database can accept the query even though similarity scores are meaningless across those two spaces. The request succeeds. The ranking fails quietly.

Shape is not meaning.

The reverse case is easier: a 1,536-value query sent to a collection configured for 768 values should be rejected at a boundary. Treat that rejection as useful evidence. Do not pad, truncate, or reshape the query to make it fit; those operations satisfy the storage shape without restoring semantic compatibility.

Preprocessing belongs in the same identity. A support article embedded with its title, body, and normalized whitespace is not equivalent to a query encoded through a different task prefix or a different text-normalization rule. Chunking changes what each document vector represents, too. None of these problems can be repaired by increasing top-k.

This matters for listings aggregated from several help-center sources. One connector may still be writing vectors under the previous configuration while another uses the current path. Fresh articles then appear relevant, old articles drift, and the mixed collection looks like an intermittent ranking problem. It is actually a provenance problem.

## The constraint that changes the build

The tempting diagnostic is to inspect a few cosine scores and adjust a threshold. That is backwards. Similarity scores are meaningful only after vector provenance is consistent, and score distributions are not portable assumptions across embedding configurations.

Use a retrieval contract with four fields: model identifier, model revision, output dimension, and preprocessing version. The exact strings are application-owned identifiers. Their value is equality, not branding. Store that contract with the collection metadata and copy its fingerprint onto each indexed record. Then reject ingestion and queries that disagree.

**A matching dimension is necessary, but it is not proof of a matching embedding space.**

The revenue-per-hour lens makes the priority clear. A full relevance evaluation suite can wait for another weekly ship. A deterministic contract check cannot. It removes an entire class of silent failures before requests reach the similarity index, while adding only a metadata lookup and string comparison to the query path. Cache immutable collection metadata if that lookup is material to latency.

## Smallest working Node.js guard

The following TypeScript keeps the storage adapter generic. It calculates a stable fingerprint from an explicit contract, validates the returned array, and refuses to query a collection created under another contract. The example dimension is local configuration, not a claim about a particular service.

```ts
import { createHash } from "node:crypto";

type EmbeddingContract = {
  model: string;
  revision: string;
  dimensions: number;
  preprocessing: string;
};

type CollectionMetadata = {
  embeddingFingerprint: string;
  dimensions: number;
};

type VectorIndex = {
  metadata(): Promise<CollectionMetadata>;
  search(vector: number[], limit: number): Promise<Array<{ id: string; score: number }>>;
};

type Embedder = {
  embed(text: string): Promise<number[]>;
};

function fingerprint(contract: EmbeddingContract): string {
  const canonical = JSON.stringify({
    dimensions: contract.dimensions,
    model: contract.model,
    preprocessing: contract.preprocessing,
    revision: contract.revision,
  });
  return createHash("sha256").update(canonical).digest("hex");
}

function normalizeSupportQuery(input: string): string {
  return input.normalize("NFKC").replace(/\s+/g, " ").trim();
}

export async function searchSupportListings(
  query: string,
  contract: EmbeddingContract,
  embedder: Embedder,
  index: VectorIndex,
  limit = 8,
): Promise<Array<{ id: string; score: number }>> {
  const metadata = await index.metadata();
  const expectedFingerprint = fingerprint(contract);

  if (metadata.embeddingFingerprint !== expectedFingerprint) {
    throw new Error("Query embedder does not match the indexed collection");
  }

  const vector = await embedder.embed(normalizeSupportQuery(query));
  if (vector.length !== contract.dimensions || vector.length !== metadata.dimensions) {
    throw new Error(
      `Embedding dimension mismatch: received ${vector.length}, expected ${contract.dimensions}`,
    );
  }
  if (vector.some((value) => !Number.isFinite(value))) {
    throw new Error("Embedding contains a non-finite value");
  }
  return index.search(vector, limit);
}
```

Apply the same fingerprint check during ingestion. Store the source identifier and content revision beside each record so a failed result can be traced back to the connector and source text. Do not log vector contents by default; the useful diagnostic fields are fingerprint, dimension, source, document revision, and index generation.

The guard is intentionally boring. Good. It makes a misconfiguration fail before customers see unrelated password-reset articles for a billing question.

Stop there.

## Prove the fix with a retrieval canary

After the contract passes, test retrieval with a small labeled fixture drawn from the actual support taxonomy. Each query needs an expected listing ID. Include close negatives: two articles that share vocabulary but resolve different intents are more revealing than an obviously unrelated pair.

Run the fixture against a new index generation before directing live queries to it. Record whether the expected ID appears in the first 1, first 3, and first 8 results, plus request latency at the application boundary. Those cutoffs are test settings, not universal quality targets. Pick the smallest candidate count that meets the product's labeled cases within its latency budget.

A clean isolation sequence is short:

1. Fetch one stored record and confirm its fingerprint and dimension.
2. Re-embed that record's exact indexed text with the declared contract.
3. Search the new vector and confirm that the same record is the nearest expected hit.
4. Run the labeled query fixture on a separate index generation.
5. Switch traffic only when both provenance and retrieval assertions pass.

If step 3 fails, freeze ranking changes. Check that the stored text is the text you think it is, that normalization is identical, and that no connector bypassed the shared ingestion function. If step 3 passes but the fixture fails, the pipeline is coherent; now chunk boundaries, source duplication, sparse lexical signals, and reranking become legitimate suspects. This distinction prevents hours of random tuning.

Do not tune blind.

Latency needs the same discipline. Measure embedding time separately from index search and any reranking stage. One total-duration metric cannot tell whether a larger candidate set, a network hop, or query encoding caused the regression.

## What changes when the corpus grows?

Do not rewrite a mixed collection in place. Build a new generation with the new contract, verify counts and labeled retrieval, then move reads to it. Keep the previous generation long enough to make rollback a metadata change rather than another bulk embedding job. Records arriving during the rebuild need a defined rule: dual-write them to both compatible generations, or replay them from an ordered change log before the switch.

At larger scale, add per-source and per-generation retrieval metrics. Aggregate accuracy can conceal one stale connector because healthy sources dominate the total. Sample failed queries with their expected and returned IDs, but establish retention and access controls because customer-support queries can contain sensitive text.

Hybrid retrieval may help exact identifiers, error codes, and product names that dense retrieval misses. It also introduces another score and another tuning surface. Add it only after the embedding contract is clean and labeled cases show a lexical gap. The same rule applies to reranking: it can improve ordering among retrieved candidates, but it cannot recover a relevant article that the first stage never returned.

The final trade-off is operational. More index generations consume storage and make deployment slightly more involved. In return, releases become testable and reversible. For a small team, that is usually the better use of engineering time than debugging a live, partially rewritten collection. Ship the invariant first.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
