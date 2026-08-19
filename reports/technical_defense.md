# Technical Defense — Lab 19

1. Coreference chỉ thay span khi antecedent có trong cùng chunk; ambiguity được log để tránh false edge.
2. Entity vector threshold là 0.90; HNSW ANN chỉ tạo candidate, lexical/type guard quyết định merge.
3. `Person` non-exact và `Technology` non-exact bị chặn; điều này tránh `Sam Altman`/`Steve Altman` và `Apple`/`Apple Watch` false merge.
4. Graph hiện có 109 nodes, 134 edges; top degree: Apple (7), Railergy (5), Meeno (3).
5. Không có node degree >100; policy vẫn áp dụng cap 50 cạnh mới nhất và global cap 250.
6. Provenance check trả 0 cạnh thiếu `source_chunk_id` hoặc `published_date`.
7. Golden benchmark hoàn thành 50 câu; overall Flat/Graph comprehensiveness lần lượt là 1.66/1.64.
8. G5000-01 thất bại do context không có Aeris–Ericsson evidence, cho thấy mismatch data/chunk scope với Golden source rows.
9. Pairwise cosine O(N²) bị từ chối; dùng HNSW ANN + Union-Find để scale.
10. Scale 350 MB cần async extraction, checkpoint, batch `UNWIND`, HNSW và community partitioning.
