# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9243 | 76 / 1876 | 7.7 / 10.0 | 554 / 2455 / 2455 | 130.3 |
| UD-Q2_K_XL | 2.24 | 5080 | 77 / 3054 | 8.1 / 10.3 | 564 / 3702 / 3702 | 123.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.06x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy của mình (RTX 3090), lượng tử hóa UD-Q2_K_XL không những không tăng tốc mà còn làm giảm decode speed (123.1 tok/s so với 130.3 tok/s của UD-Q4_K_XL). Lý do là GPU RTX 3090 có băng thông bộ nhớ (memory bandwidth) rất lớn, nên quá trình decode không bị giới hạn bởi IO mà chuyển sang bị giới hạn bởi năng lực tính toán (compute-bound) của bước giải nén (dequantization). Các phép toán giải nén 2-bit phức tạp hơn 4-bit, khiến tốc độ giảm xuống. Do đó, việc dùng 2-bit trên máy này là không đáng vì vừa chậm hơn vừa làm giảm chất lượng mô hình.
