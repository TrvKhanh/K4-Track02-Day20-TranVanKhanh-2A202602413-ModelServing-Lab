# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 109 | 1.85 | 1500 | 33000 | 33000 | 8.1 | 0.0% |
| 50 | 343 | 5.84 | 7300 | 8800 | 9600 | 41.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **3.15x** (63% of linear) |
| P95 latency | **0.27x** |
| Effective concurrency at 50 users | 41.7 vs `--parallel 4` slots (occupancy/slot ratio 10.43) |

**At capacity, still scaling.** All 4 decode slots are busy (effective concurrency 41.7) but throughput still rose 3.15x. You are at the knee -- the next increment of load is where P95 starts to run away.

P95 grew no faster than throughput (0.27x vs 3.15x), so this server still has headroom at 50 users.

## Your reading

Server đạt điểm bão hoà (saturate) khi tải tăng lên 50 user. Bằng chứng là `Effective concurrency` đạt 41.7, lớn gấp 10 lần so với số slot phục vụ tối đa (`--parallel 4`). Điều này có nghĩa là có một lượng lớn request đang phải nằm chờ trong hàng đợi thay vì được xử lý (queue time tăng). Để tăng throughput và giữ P95 ở mức an toàn cho SLO, tôi sẽ tinh chỉnh tham số `--parallel` (tăng lên 8 hoặc 16). Việc tăng `--parallel` (cùng với `ctx` đủ lớn) sẽ cho phép GPU gom nhiều request vào cùng một batch (continuous batching) để giải quyết cùng lúc. RTX 3090 có 24GB VRAM và băng thông rộng, hoàn toàn có dư tài nguyên để chạy batch lớn hơn, qua đó sẽ tăng goodput mà không làm tăng đáng kể độ trễ.
