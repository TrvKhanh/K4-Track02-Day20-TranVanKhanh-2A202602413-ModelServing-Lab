# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **10 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 132.1 | 78% |
| 5 | 168.3 | 100% |
| 10 | 167.2 | 99% |
| 16 | 167.0 | 99% |
| 32 | 165.1 | 98% |

**Best**: `-t 5` at 168.3 tok/s
**Slowest tested**: `-t 1` at 132.1 tok/s (1.27x spread)
**Against the physical-core default** (`-t 10`, 167.2 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=5 make bench
```

## Your explanation

Đỉnh (knee) đạt được ở `-t 5` (168.3 tok/s), sau đó tốc độ gần như đi ngang và giảm nhẹ khi tăng thêm luồng (chỉ còn 165.1 tok/s ở `-t 32`). Lý do chính yếu là model đã offload toàn bộ các layer tính toán nặng sang GPU (`ngl=99`). CPU bây giờ chỉ đóng vai trò điều phối luồng và thực hiện những tác vụ còn sót lại, nên không cần nhiều tài nguyên tính toán. Việc tăng số thread lớn hơn 5 không mang lại hiệu năng cao hơn mà trái lại, gây ra overhead trong việc đồng bộ hóa (synchronization) giữa các luồng (context switching) và làm chậm toàn bộ hệ thống. Ngoài ra, CPU i5-14400F sử dụng kiến trúc hybrid (6 P-cores, 4 E-cores), việc dùng quá nhiều thread sẽ lan sang các E-core yếu hơn, kéo tốc độ tổng thể đi xuống.
