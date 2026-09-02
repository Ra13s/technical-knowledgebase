# Spring AI HyDE query transformation for RAG

## What it is

Hypothetical Document Embeddings (HyDE) improve vector retrieval when users describe a problem in different vocabulary from the source documents. Instead of embedding the raw question, first ask an LLM to generate a short **hypothetical answer-like passage**, then use that passage only as the retrieval query.

The hypothetical text is not trusted as answer context. Its job is to move the embedding toward the terminology used by likely source documents; the final answer is still grounded in the real documents retrieved from the vector store.

Spring AI's `QueryTransformer` and `RetrievalAugmentationAdvisor.queryTransformers(...)` make this a small reusable pre-retrieval component.

## Use when

Try HyDE when retrieval misses relevant documents because:

- users ask conversational or symptom-oriented questions;
- documentation uses specialist/product terminology the user does not know;
- plain vector similarity has poor recall even though the correct document is indexed.

Do not enable it by default merely because it sounds smarter. It adds an LLM call before every affected retrieval. Measure retrieval quality against the added latency and model cost.

## How to use it

Implement a custom transformer:

```java
final class HydeQueryTransformer implements QueryTransformer {
    private final ChatClient chatClient;

    HydeQueryTransformer(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @Override
    public Query transform(Query query) {
        String hypotheticalDocument = chatClient.prompt()
            .user("""
                Write a short passage that would appear in a technical document
                answering this question. Use terminology the relevant source
                documentation would likely use. Do not explain your reasoning.

                Question: %s
                """.formatted(query.text()))
            .call()
            .content();

        return query.mutate()
            .text(hypotheticalDocument)
            .build();
    }
}
```

Register it before document retrieval:

```java
QueryTransformer hyde = new HydeQueryTransformer(chatClientBuilder);

Advisor rag = RetrievalAugmentationAdvisor.builder()
    .queryTransformers(hyde)
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .build())
    .build();
```

Spring AI allows multiple query transformers. For conversational RAG, a useful order is often:

```text
conversation compression -> HyDE -> vector retrieval
```

For example, use `CompressionQueryTransformer` first to turn a follow-up into a standalone question, then HyDE to translate that question into document-like vocabulary.

Spring AI recommends low model temperature for query transformation so retrieval transformations are more deterministic.

## Validation recipe

Treat HyDE as a retrieval experiment, not a prompt tweak. Build a small labeled query set containing questions that currently miss relevant documents and compare at least:

- relevant-document recall@k / hit@k;
- downstream answer correctness or groundedness;
- p50/p95 latency;
- model/token cost per request.

Keep raw-query retrieval as the baseline. Adopt HyDE only if the recall/answer improvement pays for the extra model call.

Also log the original query, generated HyDE query, and retrieved document ids in development/eval environments. Do not log sensitive user text blindly in production.

## Why it is useful

A user may ask “my service becomes slow after a few hours; how can I see what is happening?” while the relevant documentation talks about “observability”, “metrics”, “tracing”, and “Actuator”. A hypothetical answer-like passage is more likely to contain those document-side terms and therefore retrieve the right chunks.

This is especially useful for internal knowledgebases where users know the problem but not the vocabulary used by platform documentation.

## Caveats / when not to use

- Adds one LLM call before retrieval, increasing latency and cost.
- A hallucinated hypothetical passage can push retrieval in the wrong direction. Never treat it as authoritative evidence.
- If users already use the same terminology as the documents, plain vector search may perform just as well.
- Hybrid lexical/vector search or re-ranking may solve a different retrieval failure mode; evaluate them separately rather than stacking every RAG technique at once.
- Keep the final generation grounded in the actual retrieved documents, not the hypothetical document.

## Version / compatibility

The recipe uses Spring AI's stable `QueryTransformer` / `RetrievalAugmentationAdvisor` APIs available in Spring AI 2.0.1. HyDE itself is implemented as a custom transformer rather than an out-of-the-box `HydeQueryTransformer` in Spring AI 2.0.1.

## Sources

- https://www.linkedin.com/pulse/spring-ai-recipe-better-rag-results-hypothetical-document-craig-walls-o9v3c
- https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html
- https://docs.spring.io/spring-ai/docs/current/api/org/springframework/ai/rag/preretrieval/query/transformation/QueryTransformer.html
- Discovery: https://spring.io/blog/2026/09/01/this-week-in-spring-september-1-2026/
