---
layout: post
title: "Building Intelligent Agentic RAG Systems: A Comprehensive Tutorial"
date: 2026-02-28 23:30:00 +0530
categories: [ai, rag, langgraph, erp]
tags: [agentic-rag, langgraph, langchain, oracle-cloud, erp, procurement, python]
excerpt: "Learn how to build production-grade Retrieval-Augmented Generation agents with LangGraph — with a real-world Oracle Cloud ERP procurement use case."
---

Most RAG (Retrieval-Augmented Generation) tutorials show you the basics: embed documents, search vectors, generate answers. But production systems need more than simple retrieval — they need **intelligence**.

This tutorial walks through building a modular Agentic RAG system that:
- **Understands context** across multi-turn conversations
- **Clarifies ambiguous queries** before searching
- **Self-corrects** when initial results are insufficient
- **Orchestrates multiple agents** for complex questions
- **Manages memory** efficiently to avoid redundant work

We'll use **LangGraph** to orchestrate the workflow and build a system that thinks before it retrieves.

> 📚 **Based on:** [agentic-rag-for-dummies](https://github.com/GiovanniPasq/agentic-rag-for-dummies) by GiovanniPasq (2.2k ⭐)

---

## Why Agentic RAG?

Traditional RAG systems are **reactive**: user asks → retrieve docs → generate answer.

Agentic RAG is **proactive**:
1. **Analyze** the question (Is it clear? Does it need context?)
2. **Plan** the retrieval strategy (What documents? How many searches?)
3. **Execute** with self-correction (Did we find enough? Should we try again?)
4. **Synthesize** coherent answers from multiple sources

This mirrors how humans research: we don't just grab the first result — we iterate, clarify, and cross-reference.

---

## System Architecture

The system has **two main workflows**:

### 1. Conversation Understanding Pipeline
```
User Query
  ↓
Conversation Summary ← (analyzes recent history)
  ↓
Query Rewriting ← (resolves references, splits multi-part questions)
  ↓
Query Clarification ← (detects unclear inputs, asks for details)
  ↓
Multi-Agent Map-Reduce
```

### 2. Intelligent Retrieval (Per Agent)
```
Parallel Agent Subgraphs (one per sub-query)
  ↓
Search child chunks (small, precise excerpts)
  ↓
Retrieve parent chunks (large, contextual sections)
  ↓
Self-correct if results insufficient
  ↓
Compress context (avoid redundant fetches)
  ↓
Aggregate responses → Final Answer
```

---

## Key Features

- 🔍 **Hierarchical Indexing** — Search small chunks for precision, retrieve large parent chunks for context
- 💬 **Conversation Memory** — Maintains context across questions for natural dialogue
- 🔄 **Query Clarification** — Rewrites ambiguous queries or pauses to ask the user for details
- 🤖 **Agent Orchestration** — LangGraph coordinates the full retrieval and reasoning workflow
- 🔀 **Multi-Agent Map-Reduce** — Decomposes complex queries into parallel sub-queries
- ✅ **Self-Correction** — Re-queries automatically if initial results are insufficient
- 🧠 **Context Compression** — Keeps working memory lean across long retrieval loops

---

## Implementation Guide

### Step 1: Hierarchical Document Indexing

The secret to precision + context: **split documents twice**.

**Parent Chunks** — Large sections based on Markdown headers (H1, H2, H3). Provide rich context for answer generation.

**Child Chunks** — Small, fixed-size pieces (500 chars, 100 overlap). Optimized for search precision.

```python
# Parent splitting (by headers)
headers_to_split_on = [("#", "H1"), ("##", "H2"), ("###", "H3")]
parent_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on,
    strip_headers=False
)

# Child splitting (fixed size)
child_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=100
)

# Process documents
parent_chunks = parent_splitter.split_text(markdown_text)
for parent in parent_chunks:
    parent_id = f"{doc_name}_parent_{i}"
    children = child_splitter.split_documents([parent])
    # Store children in vector DB, parents in JSON files
```

**Why this works:** Child chunks return **precise matches** (high relevance). Parent chunks provide **full context** (no fragmentation). Best of both worlds!

---

### Step 2: Conversation-Aware Query Rewriting

Handles follow-up questions like "What about Python?" after asking about JavaScript.

**Rules:**
1. **Self-contained queries:** Always rewrite to be clear standalone
2. **Minimal context injection:** Use conversation summary only when necessary
3. **Grammar fixes:** Correct spelling, remove filler words
4. **Split multi-part questions:** "What is X? What is Y?" → 2 separate queries

```python
class QueryAnalysis(BaseModel):
    is_clear: bool
    questions: List[str]
    clarification_needed: str

llm_with_structure = llm.with_structured_output(QueryAnalysis)
response = llm_with_structure.invoke([
    SystemMessage(content=get_rewrite_query_prompt()),
    HumanMessage(content=f"Context: {conversation_summary}\nQuery: {user_query}")
])
```

**Example:**
- **User:** "How do I update it?" (after discussing SQL)
- **Rewritten:** "How do I update SQL databases?"

---

### Step 3: Multi-Agent Map-Reduce

Complex questions spawn **parallel agent subgraphs** — one per sub-query.

```python
def route_after_rewrite(state: State):
    if not state["questionIsClear"]:
        return "request_clarification"
    
    # Spawn parallel agents
    return [
        Send("agent", {
            "question": query,
            "question_index": idx,
            "messages": []
        })
        for idx, query in enumerate(state["rewrittenQuestions"])
    ]
```

**Example:** "What is JavaScript? What is Python?" → Spawns 2 parallel agents → executes simultaneously → aggregates results.

---

### Step 4: Self-Correcting Retrieval

Each agent follows a **research loop** with hard limits to prevent infinite loops:

```python
MAX_TOOL_CALLS = 8   # Maximum tool calls per agent
MAX_ITERATIONS = 10  # Maximum loop iterations

def route_orchestrator(state: AgentState):
    iteration = state.get("iteration_count", 0)
    tool_count = state.get("tool_call_count", 0)
    
    if iteration >= MAX_ITERATIONS or tool_count > MAX_TOOL_CALLS:
        return "fallback_response"
    
    last_message = state["messages"][-1]
    if not last_message.tool_calls:
        return "collect_answer"
    
    return "tools"
```

**Workflow:**
1. Search child chunks (5-7 results)
2. If **NO_RELEVANT_CHUNKS** → broaden query and retry
3. For each relevant child → retrieve parent chunk
4. If context insufficient → search again with new query
5. Repeat until satisfied or budget exhausted

---

### Step 5: Context Compression

Long retrieval loops = token explosion. **Compress context** to keep memory lean.

```python
def compress_context(state: AgentState):
    existing_summary = state.get("context_summary", "")
    
    conversation_text = f"USER QUESTION: {state['question']}\n\n"
    if existing_summary:
        conversation_text += f"[PRIOR COMPRESSED CONTEXT]\n{existing_summary}\n\n"
    
    # Compress via LLM
    summary = llm.invoke([
        SystemMessage(content=get_context_compression_prompt()),
        HumanMessage(content=conversation_text)
    ])
    
    return {
        "context_summary": summary,
        "messages": [RemoveMessage(id=m.id) for m in state["messages"][1:]]
    }
```

**Why it matters:** Prevents redundant fetches, reduces token costs, and maintains continuity across retrieval loops.

---

### Step 6: LangGraph Orchestration

```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(State)

workflow.add_node("summarize_history", summarize_history)
workflow.add_node("rewrite_query", rewrite_query)
workflow.add_node("request_clarification", request_clarification)
workflow.add_node("agent", agent_graph)
workflow.add_node("aggregate_answers", aggregate_answers)

workflow.add_edge(START, "summarize_history")
workflow.add_edge("summarize_history", "rewrite_query")
workflow.add_conditional_edges(
    "rewrite_query", route_after_rewrite,
    ["request_clarification", "agent"]
)
workflow.add_edge("request_clarification", END)
workflow.add_edge("agent", "aggregate_answers")
workflow.add_edge("aggregate_answers", END)

graph = workflow.compile()
```

---

## LLM Provider Configuration

The system is **provider-agnostic** — switch models in one line:

```python
# Ollama (Local, Free)
llm = ChatOllama(model="qwen3:4b-instruct-2507-q4_K_M", temperature=0)

# OpenAI GPT
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Anthropic Claude
llm = ChatAnthropic(model="claude-sonnet-4-5-20250929", temperature=0)

# Google Gemini
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash", temperature=0)
```

⚠️ For reliable tool calling, prefer **7B+ models**.

---

## Real-World Use Case: ERP Procurement & Order Management

Let's ground this architecture in a **real enterprise scenario** — an Agentic RAG system that helps procurement teams navigate Oracle Cloud ERP documentation, supplier contracts, purchase order policies, and order management workflows.

### The Problem

A typical Oracle Cloud ERP implementation generates **thousands of pages** of documentation:
- Functional design documents (FDDs) for Procurement, Order Management, AP/AR
- Supplier contracts and master service agreements
- Standard operating procedures (SOPs) for PO creation, approval workflows, 3-way matching
- Integration specs (NiFi flows, FBDI templates, REST API guides)
- Custom extension documentation (VBCS pages, BIP reports, ESS jobs)

**Procurement analysts and support teams waste hours** searching across these documents for answers like:
- *"What's the approval hierarchy for POs above $50K?"*
- *"How does the 3-way match tolerance work for this supplier?"*
- *"What FBDI template do I use for open PO imports?"*
- *"Why is this receipt failing? What are the matching rules?"*

### The Solution: Procurement RAG Agent

#### Document Indexing Strategy

```python
DOCS = {
    "procurement": [
        "PO_Creation_SOP.pdf",
        "Supplier_Qualification_Policy.pdf",
        "3Way_Match_Rules.pdf",
        "Approval_Hierarchy_Matrix.pdf"
    ],
    "order_management": [
        "Order_to_Cash_Process.pdf",
        "Fulfillment_Rules_Config.pdf",
        "Pricing_Rules_Guide.pdf",
        "Hold_Management_Procedures.pdf"
    ],
    "integrations": [
        "FBDI_Import_Templates.pdf",
        "REST_API_Procurement_Guide.pdf",
        "NiFi_Integration_Flows.pdf",
        "BIP_Report_Catalog.pdf"
    ]
}

# Hierarchical indexing works perfectly for ERP docs:
# Parent: "3-Way Matching" section (full context: tolerances, exceptions, config)
# Child: "Invoice tolerance for amount variance is 5%" (precise, searchable)
```

#### Multi-Turn Conversation Example

```
👤 User: "We have a PO for 10,000 units of steel from Tata Steel.
          The invoice shows 10,200 units. Will it pass matching?"

🤖 Agent:
  Step 1 → Summarize: User asking about quantity tolerance for Tata Steel PO
  Step 2 → Rewrite: "What is the quantity receipt tolerance for 3-way
            matching in Oracle Cloud Procurement?"
  Step 3 → Search child chunks → finds tolerance config docs
  Step 4 → Retrieve parent chunk → full 3-way match rules section
  
  Answer: "Based on the 3-Way Match Rules document, the default quantity
  tolerance is set to 2% for standard suppliers. A 10,200 unit receipt
  against a 10,000 unit PO represents a 2% variance, which is AT the
  tolerance threshold. However, Tata Steel is classified as a Strategic
  Supplier with an extended tolerance of 5%. The invoice will PASS
  matching. Note: Any quantity beyond 10,500 units (5%) would trigger
  a matching hold requiring AP manager approval per the Approval
  Hierarchy Matrix."

👤 User: "What about the price variance?"

🤖 Agent:
  Step 1 → Context: Previous question about Tata Steel PO matching
  Step 2 → Rewrite: "What are the price tolerance rules for invoice
            matching in Oracle Cloud Procurement for Strategic Suppliers?"
  
  Answer: "Price tolerance for Strategic Suppliers is 3%. If the invoiced
  unit price differs by more than 3% from the PO unit price, the invoice
  will be placed on Price Hold. The hold can be released by the
  Procurement Manager (Level 2) per Section 4.2 of the Approval
  Hierarchy Matrix."
```

#### Multi-Agent Parallel Query

```
👤 User: "Compare the approval process for direct POs vs blanket
          purchase agreements, and what FBDI template do I use to
          import open POs from the legacy system?"

🤖 Agent splits into 3 parallel sub-queries:
  
  Agent 1: "Approval workflow for direct purchase orders"
    → Searches PO_Creation_SOP.pdf, Approval_Hierarchy_Matrix.pdf
  
  Agent 2: "Approval workflow for blanket purchase agreements"  
    → Searches Supplier_Qualification_Policy.pdf
  
  Agent 3: "FBDI template for importing open purchase orders"
    → Searches FBDI_Import_Templates.pdf, REST_API_Procurement_Guide.pdf

  → Aggregated into single coherent response with comparison table
    and exact FBDI template name (PO_HEADERS_INTERFACE / PO_LINES_INTERFACE)
```

#### Self-Correction in Action

```
👤 User: "What's the receiving process for drop-ship orders?"

🤖 Agent:
  Search 1: "drop-ship receiving process" → NO_RELEVANT_CHUNKS
  (Drop-ship docs use different terminology)
  
  Self-correct → broaden query
  Search 2: "direct delivery order fulfillment receiving" → Found!
  
  Retrieves parent chunk from Order_to_Cash_Process.pdf
  → Provides complete drop-ship/direct-delivery receiving workflow
```

### Why This Matters for ERP Teams

- **Traditional:** Search SharePoint, scroll through 200-page PDFs → **Agentic RAG:** Natural language query, instant precise answers
- **Traditional:** Ask the SME (who's in a meeting) → **Agentic RAG:** 24/7 availability, consistent answers
- **Traditional:** Copy-paste from multiple docs manually → **Agentic RAG:** Auto-aggregates across all relevant documents
- **Traditional:** New team members take weeks to ramp up → **Agentic RAG:** Instant access to institutional knowledge
- **Traditional:** "I think the tolerance is 5%..." → **Agentic RAG:** "Per document X, Section 4.2, tolerance is 5% for Strategic Suppliers"

### Implementation Notes for Oracle Cloud ERP

**Document preparation:**
- Export Oracle Fusion process guides as PDF → convert to Markdown
- Include FBDI template headers as separate documents
- Maintain a glossary document (ERP acronyms → full terms) to help the query rewriter

**Embedding model:** `sentence-transformers/all-mpnet-base-v2` for ERP docs. For multi-language, consider `multilingual-e5-large`.

**Chunk size tuning:** Parent 2000-4000 chars, child 500-800 chars, overlap 150 for approval workflow docs.

---

## Key Takeaways

1. **Hierarchical indexing** balances precision and context
2. **Query clarification** prevents garbage-in-garbage-out
3. **Multi-agent map-reduce** parallelizes complex queries
4. **Self-correction** improves quality through iterative retrieval
5. **Context compression** reduces costs and prevents redundancy
6. **LangGraph** makes the workflow modular and debuggable
7. **ERP documentation** is a natural fit for agentic RAG

---

## Resources

- [agentic-rag-for-dummies](https://github.com/GiovanniPasq/agentic-rag-for-dummies) — Original repo
- [LangGraph docs](https://python.langchain.com/docs/langgraph) — Framework documentation
- [Interactive Notebook](https://colab.research.google.com/gist/GiovanniPasq/a74f077444ba21fc917dd8828bd92f23/agentic_rag_for_dummies.ipynb) — Try it yourself

---

*By [Prasenjit Gupta](https://github.com/pgupta1980) — Exploring AI-assisted enterprise solutions*
