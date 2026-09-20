# Individual contribution report

Mỗi thành viên copy template này thành:

```text
reports/<student-id>-<short-name>.md
```

Giới hạn khuyến nghị: 1 trang, không chép lại README hoặc mô tả lý thuyết chung. Báo cáo không phải một bài pipeline cá nhân; mục đích là ghi nhận ownership và bằng chứng đóng góp trong sản phẩm nhóm.

---

## Thông tin

- Họ và tên: Đào Thanh Trường
- Mã học viên: 2A202602683
- Nhóm: SaBiChuong
- Repository/branch: 2A202602683-DaoThanhTruong

## Phần việc đã thực hiện

| Module/deliverable | Việc tôi trực tiếp làm | File/commit/PR | Trạng thái |
|---|---|---|---|
| Hybrid Retrieval & Fusion | Thiết kế và tích hợp Dense Search (ChromaDB) và Lexical Search (BM25); cài đặt giải thuật Reciprocal Rank Fusion (RRF) với $k=60$ để hợp nhất xếp hạng đa nguồn | `src/task5_semantic_search.py`, `src/task6_lexical_search.py`, `src/task7_reranking.py` | Done |
| Pipeline Orchestration & Fallback | Xây dựng pipeline truy hồi end-to-end `retrieve()`, cơ chế kiểm tra ngưỡng điểm cosine thô (`SCORE_THRESHOLD = 0.3`) để kích hoạt fallback, lọc từ khóa BM25 giả mạo (`_has_adequate_lexical_evidence`) | `src/task9_retrieval_pipeline.py`, `src/task8_pageindex_vectorless.py` | Done |
| Generation with Citations & UI | Xây dựng pipeline sinh câu trả lời có trích dẫn nguồn `[Document n]`, re-order context chống suy giảm tập trung (Lost in the middle), fallback Safe Refusal và hiển thị đầy đủ sources/scores trên Streamlit UI | `src/task10_generation.py`, `app.py` | Done |

Chỉ kê khai công việc có thể đối chiếu bằng file, commit, pull request, test hoặc kết quả evaluation.

## Quyết định kỹ thuật quan trọng

Mô tả tối đa hai quyết định mà bạn trực tiếp tham gia:

1. **Quyết định:** Sử dụng Reciprocal Rank Fusion (RRF) với hằng số làm mượt $k=60$ để dung hợp Dense Retrieval (ChromaDB) và Sparse Retrieval (BM25) thay vì Weighted Score Sum.  
   **Lý do/evidence:** Dense retrieval (cosine similarity $\in [0, 1]$) và BM25 (điểm không chặn trên, phụ thuộc vào độ dài chunk và tần suất từ) có phân phối điểm số và khoảng biến thiên hoàn toàn khác biệt. Việc chuẩn hóa min-max thường bị méo mó khi gặp outlier hoặc corpus nhỏ. RRF tính điểm dựa trên thứ hạng tương đối ($1 / (k + \text{rank})$) độc lập với thang đo phân phối, đem lại sự ổn định cao và ưu tiên các tài liệu cùng xuất hiện ở thứ hạng tốt ở cả hai bộ tìm kiếm.  
   **Trade-off:** Đánh đổi thông tin về khoảng cách điểm số tuyệt đối (độ chênh lệch score giữa các vị trí liên tiếp bị san phẳng thành bước nhảy hạng tương đối), nhưng loại bỏ hoàn toàn việc phải hiệu chỉnh thủ công trọng số $\alpha$ hay scale normalization cho từng loại truy vấn.

2. **Quyết định:** Mở rộng truy vấn tiếng Việt qua từ điển intent aliases (`expand_query`) và thiết lập cơ chế lọc bằng chứng từ khóa tối thiểu (`_has_adequate_lexical_evidence`).  
   **Lý do/evidence:** Corpus tài liệu IELTS chính thống (Band Descriptors, Assessment Criteria) chủ yếu bằng tiếng Anh, trong khi người dùng tương tác bằng tiếng Việt (ví dụ: "lịch thi", "lệ phí", "tiêu chí chấm điểm"). Nếu không có intent mapping, BM25 hoàn toàn thất bại (score = 0). Tuy nhiên, khi BM25 mở rộng từ khóa, dễ gặp hiện tượng match ngẫu nhiên 1 từ rác trong footer/navigation block khi vector store trống/điểm thấp. Việc bổ sung kiểm tra độ phủ từ khóa tối thiểu (100% cho query $\le 2$ từ và 75% cho query dài hơn) bảo đảm chỉ chấp nhận tài liệu có bằng chứng từ khóa xác thực.  
   **Trade-off:** Phải xây dựng và bảo trì danh mục ánh xạ từ khóa theo nghiệp vụ (domain-specific intent dictionary); các biến thể cách diễn đạt chưa nằm trong từ điển vẫn phải phụ thuộc chính vào khả năng multilingual của dense embedding.

## Kiểm thử và kết quả

- Test hoặc query tôi đã dùng:
  - Unit/Contract Tests: Chạy bộ kiểm thử hợp đồng dữ liệu và schema trong `tests/test_contracts.py` (kiểm tra `SearchResult`, metadata, thứ hạng RRF, format context và `GenerationResult`).
  - Query in-domain: "Tiêu chí chấm điểm IELTS Writing là gì?", "Band descriptors Speaking IELTS gồm những gì?", "Cách đăng ký thi IELTS như thế nào?".
  - Query out-of-domain / adversarial: "Thời tiết Hà Nội hôm nay thế nào?", "Công thức nấu phở bò gia truyền".
- Kết quả trước/sau nếu có:
  - Khi chỉ dùng Dense retrieval: Các câu hỏi thủ tục có từ khóa đặc thù ("lệ phí", "lịch thi") đạt điểm cosine dao động 0.4 - 0.55, đôi khi bị lẫn các văn bản học thuật tổng quan.
  - Khi áp dụng Hybrid (Dense + BM25 + RRF): Các chunk tài liệu chính xác từ bài viết hướng dẫn đăng ký IDP/BC nhảy lên top 1-2 với citation rõ ràng `[Document 1]`.
  - Với câu hỏi ngoài miền (out-of-domain): Hệ thống kích hoạt Safe Refusal đúng cam kết: *"Tôi không thể xác minh thông tin này từ nguồn hiện có."*, không bị ảo giác sinh nội dung sai.
- Lỗi đã phát hiện và cách xử lý:
  - Lỗi 1: Model LLM đôi khi trích dẫn số Document không có trong context được cung cấp (out-of-bound citation hallucination).  
    *Cách xử lý:* Thêm hàm lọc bằng biểu thức chính quy `CITATION_PATTERN`, kiểm tra index hợp lệ đối chiếu với context thực tế. Nếu không có trích dẫn hợp lệ hoặc trích dẫn sai, tự động kích hoạt Safe Refusal.
  - Lỗi 2: ChromaDB lưu trữ `url: None` thành chuỗi rỗng `""`, vi phạm contract test `assert item["metadata"]["url"] is None`.  
    *Cách xử lý:* Thêm hàm chuẩn hóa `_normalize_metadata` để chuyển đổi chuỗi rỗng trở lại `None` trước khi trả kết quả ra pipeline.

## Điều còn hạn chế

- Một hạn chế cụ thể của phần tôi làm: Giai đoạn re-ranking mới chỉ dựa trên giải thuật vị trí RRF (rank fusion tĩnh), chưa áp dụng mô hình Cross-Encoder (như BGE-Reranker hay Cohere Rerank) do cân nhắc về độ trễ và tài nguyên máy trạm.
- Nếu có thêm thời gian, thay đổi đầu tiên tôi sẽ thực hiện: Tích hợp một cross-encoder nhẹ (ví dụ `BAAI/bge-reranker-base`) cho top-10 candidates sau RRF, đồng thời bổ sung bộ nhớ hội thoại đa lượt (Conversation Memory) trên giao diện Streamlit.

## Xác nhận đóng góp

Tôi xác nhận nội dung trên phản ánh đúng phần việc của mình và có thể giải thích hoặc chạy lại trong buổi demo.

- Ngày: 2026-09-21
- Tên thành viên: Đào Thanh Trường
