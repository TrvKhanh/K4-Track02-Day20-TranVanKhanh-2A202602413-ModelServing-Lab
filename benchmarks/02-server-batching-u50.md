# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 44 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 62519 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width đạt 3.96 (gần sát giới hạn của `--parallel 4`), trong khi Effective concurrency trong báo cáo trước là 41.7. Hai con số này khác nhau do chúng đo lường các khía cạnh khác nhau. Peak batch width cho biết số lượng request đang được *thực sự xử lý đồng thời* trong GPU (bị giới hạn bởi `--parallel 4`). Ngược lại, Effective concurrency (tính theo định luật Little) đo lường *tổng lượng request đang nằm trong hệ thống*, bao gồm cả những request đang phải xếp hàng chờ (`requests_deferred` lên tới 44). Tôi tin cậy cả hai: batch width cho thấy server đang tận dụng 100% khả năng xử lý của mình, còn effective concurrency phản ánh đúng áp lực thực tế mà các client đang gây ra cho server.
