# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 11822.1 | 11822.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 207.0 | 207.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 201.3 | 201.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **4076.8** · total **4076.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- N16 Cloud/IaC: Stub
- N17 Data pipeline: Stub
- N18 Lakehouse: Stub
- N19 Vector + features: Stub
- N20 Serving: Real

Việc LLM chiếm 100% thời gian (dominant stage) là hoàn toàn đúng với kỳ vọng, do thuật toán tìm kiếm từ khoá (keyword overlap) chạy cực nhanh trên CPU, trong khi LLM phải chạy qua hàng tỷ tham số để suy luận và sinh văn bản. Nếu cần giảm một nửa độ trễ của pipeline này, tôi sẽ tấn công vào stage LLM. Có thể áp dụng Semantic Caching (như trong bài lab bonus) để tránh việc phải gọi LLM cho các câu hỏi trùng lặp, hoặc sử dụng Prompt Caching để tận dụng KV cache cho các context tĩnh.
