# Multimodal Lakehouse RAG

Multimodal search at scale.

## Context

- Managing up to 20 TB of multimodal data (images, videos, text, audio) across storage mediums for AI-driven search, deduplication, indexing, and categorization
- Multimodal lakehouse architecture unifies storage, governance, and processing for up to 20TB of diverse data (images, videos, text) on consumer hardware
- Open formats like Lance or Apache Iceberg on cheap object storage (e.g., local HDDs/NAS) with vector-native tables for embeddings, support hybrid SQL-vector queries without data duplication
- Centralize raw assets, metadata, and AI features while optimizing I/O for large video files via columnar layouts and zero-copy access
- Ollama or LM Studio to run multimodal LLMs (e.g., LLaVA, Phi-3 Vision) locally for tagging and search
- Milvus or FAISS for vector indexing of embeddings from CLIP or BLIP models
- Deduplication via perceptual hashing (pHash for images/videos) or MinHash
- Batch processing with DVC or Ray for parallelism
- LanceDB as the multimodal lakehouse engine: store raw videos/images in Lance format (columnar, versioned, with Delta logs for mutability) on a local ZFS RAID or TrueNAS Scale NAS with 40TB+ HDDs tiered to NVMe cache
- Use medallion layers—bronze (raw ingestion), silver (deduped/embedded), gold (tagged/indexed)—enforced via dbt or Spark for progressive refinement
- Integrate Ray for distributed processing: parallelize embedding generation with CLIP/BLIP on your RTX GPU cluster, feeding directly into Lance tables for RAG-ready search
- Ingest via FUSE-mounted directories or MinIO (S3-compatible on localhost) to handle multi-medium sources; dedupe large videos with video hashing (e.g., pHash on keyframes via FFmpeg extracts) before embedding
- Process in Ray jobs: chunk videos into 10-60s clips, extract features (audio via Whisper, visuals via YOLO/SigLIP), and append embeddings/metadata atomically to Lance tables—no rewrites needed
- Tag/categorize with batched LLaVA inference, versioning datasets for reproducibility
- Lance's layered disk layout and object storage semantics minimize seeks for large files: co-locate thumbnails, embeddings, and keyframes contiguously, enabling GPU-direct access (zero-copy via CUDA memory mapping)
- For 4K videos, use adaptive chunking (e.g., 1GB fragments) and prefetching in Ray actors; NVMe SSDs as L2ARC cache hit 90%+ for random reads during search
- Avoid Parquet pitfalls—Lance supports streaming upserts and compaction, sustaining 10GB/s throughput on consumer NVMe arrays vs. traditional lakes' batch bottlenecks

---

Tagging

- Extract features hierarchically: use FFmpeg to pull keyframes/transcripts from videos, then run BLIP-2 or LLaVA 1.6 (13B quantized to 4-bit) for image captions/objects, Whisper for audio transcription, and CLIP for unified embeddings
- Batch-process 1,000 clips/hour on dual RTX 5090s, appending JSON metadata (e.g., {"objects": ["car", "child"], "scene": "outdoor"}) directly to Lance columns
- Human-in-loop via Gradio UI refines edge cases, with 90%+ auto-accuracy after fine-tuning on 1% sampled data

Categorization

- Cluster embeddings with FAISS or HNSW in LanceDB: group similar assets via cosine similarity on joint text-vision vectors from SigLIP + sentence-transformers
- Apply zero-shot classification with GPT-4o-mini local proxies (e.g., "categorize as family, work, or travel") or fine-tuned LoRA on Phi-3 Vision for domains like "sports" or "recipes"
- Time-sync tags for videos (e.g., scene changes every 30s) using S3D embeddings, enabling temporal queries

Indexing

- Build hybrid indexes in LanceDB: full-text on metadata/tags via Tantivy, vector search on 768-dim embeddings (fits 128GB RAM), and spatial on object bounding boxes from YOLOv10
- Upsert incrementally during processing—no full rebuilds—with compaction handling 20TB scale via Z-order partitioning by file type/date
- Query fuses results (e.g., SQL JOIN vector similarity >0.8), hitting <50ms latency on consumer NVMe

Optimization

- Quantize to AWQ/GPTQ for 2x speed on your GPUs; use vLLM for serving during indexing
- Dedupe tags pre-storage with embedding thresholds to cut index size 40%
- Monitor via Prometheus on TrueNAS, scaling to 4x GPUs ($1K used) for 2-day full reindex

---



## Learning outcomes

- [ ] Lakehouse architecture (medallion layers: bronze/silver/gold)
- [ ] Multimodal data formats (Lance, Parquet, Iceberg for versioning/ACID)
- [ ] Vector databases (LanceDB, FAISS, HNSW indexing)
- [ ] Embedding models (CLIP, BLIP-2, SigLIP for images/videos)
- [ ] Vision-language models (LLaVA, Phi-3 Vision for tagging)
- [ ] Audio processing (Whisper for transcription)
- [ ] Object detection (YOLOv10 for bounding boxes)
- [ ] Deduplication (perceptual hashing, MinHash)
- [ ] Distributed processing (Ray, Dask for parallelism)
- [ ] Local storage (ZFS/TrueNAS, MinIO S3 gateway)
- [ ] GPU optimization (quantization: AWQ/GPTQ, vLLM inference)
- [ ] Hybrid querying (SQL + vector search, RAG pipelines)
- [ ] Data ingestion (FFmpeg chunking, FUSE mounts)
- [ ] Feature stores (materialized views, UDFs)
- [ ] Observability (Prometheus for I/O monitoring)

---

- [ ] Open-table formats like Lance for ACID-compliant, multimodal storage with zero-ETL querying
- [ ] Medallion architecture (raw, enriched, served layers)
- [ ] Hybrid indexes (SQL + vectors)
- [ ] Columnar partitioning
- [ ] Distributed DAGs for embedding/tagging pipelines
- [ ] Autoscaling across GPUs/CPUs with fault-tolerant workflows
- [ ] Serve endpoints, actor pools, and integration to LanceDB UDFs
- [ ] Deploy quantized VLMs via vLLM/Ollama with OpenAI APIs
- [ ] Optimize throughput via tensor parallelism and PagedAttention
- [ ] GPU memory management critical for production RAG serving

---

- [ ] 

## Architecture layers

- Decoupled, modular layers optimized for local consumer hardware
- Production-grade serving without cloud dependencies

Storage

- Lance format tables on local ZFS/TrueNAS or MinIO (S3 gateway) expose data via filesystem mounts and HTTP endpoints
- Zero-copy access serves raw videos/images directly to GPUs
- NVMe caching (L2ARC) and columnar partitioning enable 10GB/s reads for 20TB datasets, with Delta logs for ACID upserts served atomically (no external services needed beyond a single-node LanceDB server process)

Compute

- Ray clusters (head + worker nodes on multi-core Ryzen/EPYC) distribute embedding/tagging workloads across CPUs/GPUs
- Serve jobs via Ray Serve HTTP endpoints (e.g., /embed POST for batch inference), hitting 1K clips/sec on dual RTX 5090s; integrates natively with LanceDB for read/write without data movement

Orchestration

- Ray workflows or Airflow (lightweight on localhost) schedule pipelines as DAGs, exposing status APIs at `localhost:8265`
- UDFs in LanceDB's `geneva` package run serverlessly on Ray actors
- Trigger via CLI (`ray job submit`) or Streamlit dashboard for monitoring I/O, GPU util, and reindexing—fully local, no external orchestrators required

Model serving

- Quantized VLMs (LLaVA, BLIP-2) via vLLM or Ollama serve at `localhost:8000/v1` with OpenAI-compatible APIs; batch requests from Ray jobs for 100+ tps
- GPU-direct inference offloads tagging/categorization, with model weights cached in Lance tables for versioning—consumer hardware sustains continuous serving post-initial indexing

Search/index

- LanceDB's embedded server (`lancedb.connect()`) provides hybrid SQL+vector endpoints over HTTP/gRPC; HNSW indexes query in <50ms with filters (e.g., `table.search("cooking scenes").where("date>2025")`)
- Expose via FastAPI/Gradio wrapper for RAG UI, scaling reads across Ray for concurrent users—all on single box with 128GB RAM

## System design

- Decoupled storage, compute, and serving layers (scale independently)
- Columnar zero-copy (GPU-direct access)
- Incremental indexing (no full rebuilds)
- Local-first, cloud-portable (S3/EKS ready)
- Batch-optimized (1000 clips/hour RTX 5090)

Storage

- LanceDB tables (ACID, columnar, multimodal)
- ZFS/TrueNAS RAID (caching, snapshots)
- MinIO S3 gateway (object compatibility)
- Medallion architecture (bronze/silver/gold)

Compute/processing

- Ray clusters (distributed tasks, autoscaling)
- FFmpeg chunking (video keyframes)
- Embedding pipelines (CLIP/BLIP parallelization)
- Deduplication (pHash, MinHash)

AI/model

- Quantized VLMs (LLaVA, Phi-3 Vision 4-bit)
- vLLM/Ollama serving (OpenAI API)
- Feature extraction (YOLOv10, Whisper)
- Zero-shot classification (scene/object tags)

Indexing/search

- HNSW vectors (FAISS/LanceDB hybrid)
- SQL+vector queries (full-text metadata)
- RAG pipelines (context retrieval)
- Temporal indexing (video timestamps)

Orchestration/monitoring

- Ray workflows (DAG scheduling)
- dbt transformations (data quality)
- Prometheus/Grafana (I/O, GPU metrics)
- Gradio/Streamlit UI (query interface)

## Hardware setup

- NAS with a modern GPU (e.g., NVIDIA RTX 4090 with 24GB VRAM or RTX 5090)
- 128GB+ system RAM
- NVMe SSDs for fast indexing

## Checklist

- [ ] Mount drives in a Linux NAS (TrueNAS Scale),use `fdupes` or `rdfind` for exact dupes, AI-based near-dupes with imagehash library
- [ ] Batch-generate embeddings with `sentence-transformers` (multimodal) and store in ChromaDB or Qdrant for semantic search
- [ ] Prompt local Llama 3.2 Vision or Bakllava on GPU for auto-labeling; use YOLOv8 for object detection in videos/images
- [ ] Expose via Gradio or Streamlit app with RAG over the index for querying
- [ ] Quantize models to 4-bit with llama.cpp for consumer GPU fit; distribute across multiple cheap GPUs via vLLM
