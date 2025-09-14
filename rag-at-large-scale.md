Been building RAG systems for mid-size enterprise companies in the regulated space (100-1000 employees) for the past year and to be honest, this stuff is way harder than any tutorial makes it seem. Worked with around 10+ clients now - pharma companies, banks, law firms, consulting shops. Thought I'd share what actually matters vs all the basic info you read online.

Quick context: most of these companies had 10K-50K+ documents sitting in SharePoint hell or document management systems from 2005. Not clean datasets, not curated knowledge bases - just decades of business documents that somehow need to become searchable.

Document quality detection: the thing nobody talks about

This was honestly the biggest revelation for me. Most tutorials assume your PDFs are perfect. Reality check: enterprise documents are absolute garbage.

I had one pharma client with research papers from 1995 that were scanned copies of typewritten pages. OCR barely worked. Mixed in with modern clinical trial reports that are 500+ pages with embedded tables and charts. Try applying the same chunking strategy to both and watch your system return complete nonsense.

Spent weeks debugging why certain documents returned terrible results while others worked fine. Finally realized I needed to score document quality before processing:

Clean PDFs (text extraction works perfectly): full hierarchical processing

Decent docs (some OCR artifacts): basic chunking with cleanup

Garbage docs (scanned handwritten notes): simple fixed chunks + manual review flags

Built a simple scoring system looking at text extraction quality, OCR artifacts, formatting consistency. Routes documents to different processing pipelines based on score. This single change fixed more retrieval issues than any embedding model upgrade.

Why fixed-size chunking is mostly wrong

Every tutorial: "just chunk everything into 512 tokens with overlap!"

Reality: documents have structure. A research paper's methodology section is different from its conclusion. Financial reports have executive summaries vs detailed tables. When you ignore structure, you get chunks that cut off mid-sentence or combine unrelated concepts.

Had to build hierarchical chunking that preserves document structure:

Document level (title, authors, date, type)

Section level (Abstract, Methods, Results)

Paragraph level (200-400 tokens)

Sentence level for precision queries

The key insight: query complexity should determine retrieval level. Broad questions stay at paragraph level. Precise stuff like "what was the exact dosage in Table 3?" needs sentence-level precision.

I use simple keyword detection - words like "exact", "specific", "table" trigger precision mode. If confidence is low, system automatically drills down to more precise chunks.

Metadata architecture matters more than your embedding model

This is where I spent 40% of my development time and it had the highest ROI of anything I built.

Most people treat metadata as an afterthought. But enterprise queries are crazy contextual. A pharma researcher asking about "pediatric studies" needs completely different documents than someone asking about "adult populations."

Built domain-specific metadata schemas:

For pharma docs:

Document type (research paper, regulatory doc, clinical trial)

Drug classifications

Patient demographics (pediatric, adult, geriatric)

Regulatory categories (FDA, EMA)

Therapeutic areas (cardiology, oncology)

For financial docs:

Time periods (Q1 2023, FY 2022)

Financial metrics (revenue, EBITDA)

Business segments

Geographic regions

Avoid using LLMs for metadata extraction - they're inconsistent as hell. Simple keyword matching works way better. Query contains "FDA"? Filter for regulatory_category: "FDA". Mentions "pediatric"? Apply patient population filters.

Start with 100-200 core terms per domain, expand based on queries that don't match well. Domain experts are usually happy to help build these lists.

When semantic search fails (spoiler: a lot)

Pure semantic search fails way more than people admit. In specialized domains like pharma and legal, I see 15-20% failure rates, not the 5% everyone assumes.

Main failure modes that drove me crazy:

Acronym confusion: "CAR" means "Chimeric Antigen Receptor" in oncology but "Computer Aided Radiology" in imaging papers. Same embedding, completely different meanings. This was a constant headache.

Precise technical queries: Someone asks "What was the exact dosage in Table 3?" Semantic search finds conceptually similar content but misses the specific table reference.

Cross-reference chains: Documents reference other documents constantly. Drug A study references Drug B interaction data. Semantic search misses these relationship networks completely.

Solution: Built hybrid approaches. Graph layer tracks document relationships during processing. After semantic search, system checks if retrieved docs have related documents with better answers.

For acronyms, I do context-aware expansion using domain-specific acronym databases. For precise queries, keyword triggers switch to rule-based retrieval for specific data points.

Why I went with open source models (Qwen specifically)
Most people assume GPT-4o or o3-mini are always better. But enterprise clients have weird constraints:

Cost: API costs explode with 50K+ documents and thousands of daily queries

Data sovereignty: Pharma and finance can't send sensitive data to external APIs

Domain terminology: General models hallucinate on specialized terms they weren't trained on

Qwen QWQ-32B ended up working surprisingly well after domain-specific fine-tuning:

85% cheaper than GPT-4o for high-volume processing

Everything stays on client infrastructure

Could fine-tune on medical/financial terminology

Consistent response times without API rate limits

Fine-tuning approach was straightforward - supervised training with domain Q&A pairs. Created datasets like "What are contraindications for Drug X?" paired with actual FDA guideline answers. Basic supervised fine-tuning worked better than complex stuff like RAFT. Key was having clean training data.

Table processing: the hidden nightmare

Enterprise docs are full of complex tables - financial models, clinical trial data, compliance matrices. Standard RAG either ignores tables or extracts them as unstructured text, losing all the relationships.

Tables contain some of the most critical information. Financial analysts need exact numbers from specific quarters. Researchers need dosage info from clinical tables. If you can't handle tabular data, you're missing half the value.

My approach:

Treat tables as separate entities with their own processing pipeline

Use heuristics for table detection (spacing patterns, grid structures)

For simple tables: convert to CSV. For complex tables: preserve hierarchical relationships in metadata

Dual embedding strategy: embed both structured data AND semantic description

For the bank project, financial tables were everywhere. Had to track relationships between summary tables and detailed breakdowns too.

Production infrastructure reality check

Tutorials assume unlimited resources and perfect uptime. Production means concurrent users, GPU memory management, consistent response times, uptime guarantees.

Most enterprise clients already had GPU infrastructure sitting around - unused compute or other data science workloads. Made on-premise deployment easier than expected.

Typically deploy 2-3 models:

Main generation model (Qwen 32B) for complex queries

Lightweight model for metadata extraction

Specialized embedding model

Used quantized versions when possible. Qwen QWQ-32B quantized to 4-bit only needed 24GB VRAM but maintained quality. Could run on single RTX 4090, though A100s better for concurrent users.

Biggest challenge isn't model quality - it's preventing resource contention when multiple users hit the system simultaneously. Use semaphores to limit concurrent model calls and proper queue management.

Key lessons that actually matter
1. Document quality detection first: You cannot process all enterprise docs the same way. Build quality assessment before anything else.

2. Metadata > embeddings: Poor metadata means poor retrieval regardless of how good your vectors are. Spend the time on domain-specific schemas.

3. Hybrid retrieval is mandatory: Pure semantic search fails too often in specialized domains. Need rule-based fallbacks and document relationship mapping.

4. Tables are critical: If you can't handle tabular data properly, you're missing huge chunks of enterprise value.

5. Infrastructure determines success: Clients care more about reliability than fancy features. Resource management and uptime matter more than model sophistication.

The real talk

Enterprise RAG is way more engineering than ML. Most failures aren't from bad models - they're from underestimating the document processing challenges, metadata complexity, and production infrastructure needs.

The demand is honestly crazy right now. Every company with substantial document repositories needs these systems, but most have no idea how complex it gets with real-world documents.

Anyway, this stuff is way harder than tutorials make it seem. The edge cases with enterprise documents will make you want to throw your laptop out the window. But when it works, the ROI is pretty impressive - seen teams cut document search from hours to minutes.

