# Lab 17 - Báo Cáo Thực Hành Multi-Memory Agent

## 1. Trả lời câu hỏi lý thuyết & thiết kế

- **Layer quan trọng nhất trong bộ test:** **Long-term Memory** là layer cốt lõi nhất vì đảm nhiệm 4/11 case (`E02`, `E03`, `E08`, `E09`). Layer này quyết định khả năng nhớ sở thích người dùng qua nhiều thread (`E02`), theo dõi open loops/TODOs (`E03`), xử lý xung đột cập nhật thông tin theo thời gian (*recency*) (`E08`), và đảm bảo cách ly dữ liệu giữa các user (`E09`).
- **Đánh đổi giữa Zep Context Block vs Tự dựng Redis + Qdrant:**
  - *Zep (Managed):* Tự động trích xuất entity/graph relation, quản lý temporal validity (recency) và lắp ráp Context Block liên quan tự động; đánh đổi là phụ thuộc dịch vụ ngoài và tính bất đồng bộ khi ingest.
  - *Redis + Qdrant (Self-managed):* Kiểm soát hoàn toàn dữ liệu, độ trễ thấp, chi phí local; nhưng phải tự code pipeline trích xuất quan hệ, xử lý mâu thuẫn fact và thuật toán merge context.
- **Guardrail chống Memory Poisoning:** Bắt buộc kiểm tra quyền ghi (*consent opt-in*), chuẩn hóa schema/provenance, khử thông tin nhạy cảm (*PII redaction*), và cô lập strictly theo `user_id`. Các tiến trình nền (như *heartbeat*) chỉ được dọn dẹp/dedup, tuyệt đối không tự cấp quyền hay nạp chỉ thị mới vào durable memory.

## 2. Phân tích kết quả Benchmark

1. **Layer có hit rate thấp nhất:** Ở baseline *no-memory*, toàn bộ các tầng `long_term`, `episodic`, `semantic`, `mixed` đều rớt về **0%** vì không có bộ nhớ bền vững. Với *student memory*, sau khi cấu hình Zep, tất cả các tầng đều đạt **100% hit rate**.
2. **Query retrieve nhiều token nhất:** Case mixed `E07` và các query `long_term` dài do cần gom đồng thời Context Block và đồ thị tri thức/facts.
3. **Case mixed (`E07`):** Cần kết hợp **Long-term** (sở thích ngôn ngữ `Python` của user) và **Semantic Memory** (quy tắc retry thanh toán với `Idempotency-Key`).
4. **Token reduction vs Hit rate:** Baseline *no-memory* có token reduction rất cao (~98%) vì không đưa context vào prompt, nhưng hit rate rớt về 0. Memory-enabled agent tối ưu token bằng *token budget (10/4/3/3)* và *compaction*, vừa giảm ~70-80% token so với raw transcript vừa đạt 100% hit rate.

## 3. Phân tích Recency & Compaction

- **Recency (`E08`):** Khi user đổi sang `TypeScript`/`NestJS` ở project `BLUEBIRD-42`, Zep cập nhật validity timestamp để fact mới chiến thắng preference `Python` cũ trong ngữ cảnh mới.
- **Compaction (`E10`):** Khi hội thoại vượt quá cửa sổ trượt, compaction trích xuất và bảo toàn các *durable notes* (`REVIEW-DEADLINE-1600`, `Friday`, `16:00`), tránh mất mát ràng buộc quan trọng.
