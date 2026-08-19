# Failure Analysis — Lab 19

## Case 1 — G5000-01 Aeris–Ericsson

Flat RAG và GraphRAG đều trả lời thiếu evidence. Reference yêu cầu hai business IoT của Ericsson và ba chỉ số scale, nhưng context không chứa các chunk tương ứng. Root cause là Golden Set tham chiếu các source row cụ thể trong 5.000 dòng đầu, trong khi lần benchmark hiện tại không chứng minh các row đó đã được đưa vào chunk/extraction scope. Khắc phục bằng `source_row_id` và `select_extraction_source()` trước coreference/extraction.

## Case 2 — G5000-09 Now Assist for Virtual Agent

Flat RAG đạt comprehensiveness 2, GraphRAG đạt 1. Seed extraction/graph extraction không tạo được subgraph chứa event/capability yêu cầu. Khắc phục bằng kiểm tra seed diagnostics, bổ sung relation phù hợp vào allowlist khi có evidence, và fallback vector nếu graph context rỗng hoặc insufficient.

## Kết luận

Điểm benchmark thấp là tín hiệu coverage failure, không phải bằng chứng GraphRAG kém hơn mặc định. Evaluation chỉ có giá trị khi Golden evidence, chunk store và graph extraction cùng một source scope.
