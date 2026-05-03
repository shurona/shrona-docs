---
tags:
  - RAG
---
# 내 문서를 임베딩 해보자
## 문서를 청킹 하기
```python
doc_chunks: list[tuple[Document, list[Chunk]]] = []
all_chunks: list[Chunk] = []
for doc in documents:
    chunks = chunk_document(doc, settings.chunk_size, settings.chunk_overlap)
    doc_chunks.append((doc, chunks))  # 문서-청크 쌍으로 보관
    all_chunks.extend(chunks)         # 전체 청크 리스트에 추가
```
## 임베딩할 문서 추출
- 각 청크의 embedding_text (메타데이터 + 본문) 를 리스트로 생성한다.
```
제목: Redis란 | 카테고리: Spring | 섹션: ## 특징\n\nRedis는 ...
제목: Redis란 | 카테고리: Spring | 섹션: ## 사용법\n\n다음과 같이 ...
```
- 
```python
texts = [chunk.embedding_text for chunk in all_chunks]
```
### 임베딩 생성
- 생성 코드
```python
vectors = embedder.embed_batch(texts, batch_size=32)
```
- Output 예시
```txt
"Redis는 캐시 DB입니다" → [0.12, -0.34, 0.87, ... (768개 숫자)]
```
