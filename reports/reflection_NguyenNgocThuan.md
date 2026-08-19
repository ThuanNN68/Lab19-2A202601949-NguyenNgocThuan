# Reflection — Nguyễn Ngọc Thuận

Lab cho thấy GraphRAG chỉ hữu ích khi entity/relation extraction và Golden source alignment đáng tin cậy. Bài học debugging quan trọng nhất là kiểm tra provenance từ raw source row đến chunk, triple và context trước khi đọc điểm Judge.

Trong đồ án thực tế, tôi sẽ chọn Hybrid RAG khi câu hỏi cần nối nhiều tài liệu. Thiết kế ban đầu gồm Document, Organization, Person, Technology và Policy; relation gồm OWNS, USES, ANNOUNCED, APPLIES_TO và SUPERSEDES. Candidate entity gần ngưỡng sẽ vào audit/human-review queue; node có degree cao sẽ dùng temporal cap, relation ranking và community-level summary.
