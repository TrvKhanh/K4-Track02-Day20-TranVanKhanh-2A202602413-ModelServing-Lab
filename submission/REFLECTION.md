# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Tran Van Khanh
**MSSV:** 202602413
**Cohort:** K4-Track02
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 7.0.0-15-generic
- **CPU:** Intel(R) Core(TM) i5-14400F
- **Cores:** 10 / 16
- **CPU extensions:** AVX2
- **RAM:** 30.6 GB
- **Accelerator:** NVIDIA GeForce RTX 3090, 24576 MiB
- **llama.cpp asset đã tải:** prebuilt release b10488
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=`gemma4-e2b)
- **Quantization:** gemma-4-E2B-it-UD-Q4_K_XL.gguf + gemma-4-E2B-it-UD-Q2_K_XL.gguf (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story**: Cài đặt chạy mượt mà ngay lần đầu. Phần cứng mạnh (RTX 3090, 32GB RAM) nên tải mô hình và cấu hình prebuilt llama.cpp rất nhanh, không cần biên dịch lại từ mã nguồn.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9243 | 76 / 1876 | 7.7 / 10.0 | 554 / 2455 / 2455 | 130.3 |
| UD-Q2_K_XL | 2.24 | 5080 | 77 / 3054 | 8.1 / 10.3 | 564 / 3702 / 3702 | 123.1 |

**Quan sát**: Tốc độ Q2_K_XL chậm hơn Q4_K_XL (123.1 vs 130.3 tok/s). Nguyên nhân do GPU RTX 3090 có memory bandwidth lớn, khiến giới hạn tốc độ chuyển từ I/O sang năng lực giải nén (compute). Q2 giải nén phức tạp hơn nên chạy chậm hơn, và chất lượng cũng kém hơn nên không đáng dùng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 109 | 1.85 | 1500 | 33000 | 33000 | 8.1 | 0.0% |
| 50 | 343 | 5.84 | 7300 | 8800 | 9600 | 41.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 3.15×
- **P95 tăng:** 0.27×
- **Effective concurrency ở 50 users:** 41.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`**: 3.96 / 4 slots

**Saturation reading**: Server bão hoà ở mức 50 users. Bằng chứng là Peak busy slots đạt 3.96/4, cho thấy engine đã làm việc hết công suất song song (4 slots). Đồng thời, Effective concurrency là 41.7 (rất cao so với 4 slot), nghĩa là phần lớn các request đang phải chờ trong hàng đợi. Để tăng throughput/goodput, tôi sẽ tăng số lượng parallel slots (`--parallel` = 8 hoặc 16) do VRAM và GPU vẫn còn dư dả năng lực tính toán theo batch.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub |
| N17 Data pipeline | stub |
| N18 Lakehouse | stub |
| N19 Vector + features | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 4076.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection**: Thời gian xử lý chủ yếu nằm ở phần LLM (100%), hoàn toàn khớp với dự đoán do thuật toán search (keyword) chạy siêu nhanh, còn LLM phải tính toán khối lượng lớn. Để giảm độ trễ một nửa, tôi sẽ tấn công vào LLM bằng cách dùng model nhẹ hơn hoặc áp dụng semantic caching/prompt caching.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số lượng thread (`-t`) từ mặc định 10 xuống 5.

```
before:  167.2 tok/s
after:   168.3 tok/s
speedup: 1.01×
```

**Tại sao nó work**:

Việc hạ số luồng từ 10 xuống 5 (tương đương với số Performance-cores của CPU) lại giúp tăng nhẹ tốc độ sinh token. Nguyên nhân cơ học là do model đang offload toàn bộ các layer tính toán nặng sang GPU (`ngl=99`). Ở đây CPU chủ yếu làm nhiệm vụ điều phối và đồng bộ tiến trình (synchronization). Tăng thêm luồng (từ 5 lên 10 hoặc 16) khiến CPU phải chịu chi phí overhead cho context switching và giao tiếp giữa các lõi (cụ thể là E-cores yếu hơn bị kéo vào), dẫn đến tốc độ bị suy giảm thay vì tăng lên.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

_(Công cụ nào, dùng vào việc gì. Ghi "Không dùng" nếu không dùng.)_
