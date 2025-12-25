# 7-Day Recommendation Engine Roadmap

| Day | Phase | Tasks |
|:---|:---|:---|
| **Day 1** | **Embedding Pipeline** | <ul><li>Setup environment (Torch/Transformers)</li><li>Script batch processing for posts</li><li>Store embeddings (.pt/.h5)</li></ul> |
| **Day 2** | **Zero-Shot Tagging** | <ul><li>Define label taxonomy</li><li>Prompt engineering ("A photo of...")</li><li>Compute similarity scores</li></ul> |
| **Day 3** | **Vector Infrastructure** | <ul><li>Integrate Vector DB (FAISS/Pinecone)</li><li>Index Day 1 & 2 data</li><li>Test top-k retrieval</li></ul> |
| **Day 4** | **Content Similarity** | <ul><li>Build 'More Like This' logic</li><li>Implement diversity filters</li></ul> |
| **Day 5** | **User Personalization** | <ul><li>Track user 'Like' history</li><li>Generate User Interest Vectors</li></ul> |
| **Day 6** | **API & UI Integration** | <ul><li>Build `/suggestions` endpoint</li><li>Add 'Discover' shelf in UI</li></ul> |
| **Day 7** | **Tuning & Polish** | <ul><li>Relevance audit</li><li>Latency benchmarking</li><li>Feedback loop tracking</li></ul> |