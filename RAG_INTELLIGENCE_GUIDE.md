# DIGO Intelligence Layer - RAG Implementation Guide

## Overview

Add AI-powered retrieval and intelligence features to help DIGO facilitators, municipality staff, and SDC reporting teams work faster and smarter.

**Core Concept**: Use Retrieval-Augmented Generation (RAG) to query across documents, data sources, and platform outputs — similar to Yatrī's architecture but focused on policy, statistics, and workshop data instead of travel content.

---

## 🚀 Production Version: Planning Intelligence Assistant

For the **production-ready implementation** with multi-criteria ranking, see:

### 📄 PLANNING_INTELLIGENCE_ASSISTANT.md

The Planning Intelligence Assistant builds on this RAG foundation with a **ranking layer** that scores results by:
- **Policy Alignment Score** (0-1) - Does it align with Nepal Tourism Strategy & DIGO criteria?
- **Data Confidence Tier** (0.2-1.0) - Tier 1 (validated sources) > Tier 2 (consensus) > Tier 3 (single source)
- **SDC Reporting Relevance** (0-1) - Does it include metrics useful for donor reporting?

**Why this matters**: Standard RAG only uses semantic similarity. The Planning Intelligence Assistant prioritizes **high-quality, policy-aligned, SDC-relevant** sources even if they have slightly lower text similarity.

**This guide** covers basic RAG implementation. **PLANNING_INTELLIGENCE_ASSISTANT.md** covers the production version with intelligent ranking.

---

## Three Intelligence Features

### 1. Workshop Preparation Intelligence 📊

**Problem**: Facilitators spend hours manually gathering evidence before workshops.

**Solution**: Ask natural language questions, get sourced answers instantly.

**Example Queries**:
- "What does official data show about Lakeside visitor volumes in Q3 2025?"
- "What are ACAP permit volumes for Annapurna region in last 6 months?"
- "Show me hotel occupancy rates in Pokhara during October"
- "What does NTB data say about international vs domestic tourist split?"

**Data Sources**:
- Nepal Tourism Board (NTB) arrival statistics
- ACAP (Annapurna Conservation Area Project) permit databases
- Hotel association occupancy reports
- Municipal tourism budget documents
- Previous DIGO workshop findings
- Government tourism survey data

---

### 2. Policy Alignment Checking ✅

**Problem**: Staff need to manually verify if action plans align with multiple policy frameworks.

**Solution**: Automatic validation against policy documents.

**Example Queries**:
- "Does this action plan align with Nepal's visitor dispersal objectives?"
- "Check if our homestay initiative meets DIGO sustainable development criteria"
- "Is this overtourism mitigation plan consistent with National Tourism Strategy 2022-2032?"
- "Does this infrastructure proposal comply with SDC environmental safeguards?"

**Document Corpus**:
- Nepal National Tourism Strategy 2022-2032
- Ministry of Culture, Tourism & Civil Aviation (MoCTCA) policy papers
- DIGO sustainable tourism criteria documents
- SDC (Swiss Agency for Development) reporting frameworks
- Swisscontact programme guidelines
- UNESCO sustainable tourism guidelines
- UNWTO overtourism management principles

---

### 3. SDC Reporting Assistance 📝

**Problem**: Writing SDC progress reports is time-consuming and repetitive.

**Solution**: Auto-generate report drafts from structured platform data.

**Example Queries**:
- "Draft Q2 2026 progress report for SDC in standard format"
- "Summarize action plan outcomes for Pokhara with evidence"
- "Generate narrative report on visitor dispersal achievements"
- "Create impact summary for women-led homestay initiatives"

**Data Sources**:
- DIGO platform workshop outputs (action plans, flow maps, pressure analyses)
- Flow data and analytics (visitor counts, pressure points, economic leakage)
- Municipal action plan implementation tracking
- Stakeholder workshop participation records
- Economic impact calculations
- Evidence quality metadata

---

## Architecture

### Tech Stack

```
┌─────────────────────────────────────────┐
│         Next.js Frontend                │
│  - RAG Query Interface                  │
│  - Document Upload UI                   │
│  - Report Generation Dashboard          │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       Next.js API Routes                │
│  - /api/rag/query                       │
│  - /api/rag/upload                      │
│  - /api/rag/generate-report             │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│         RAG Pipeline                    │
│  1. Query → Embedding                   │
│  2. Vector Similarity Search            │
│  3. Context Retrieval                   │
│  4. LLM Generation with Sources         │
└─────────────────────────────────────────┘
                   ↓
┌──────────────────┬──────────────────────┐
│  Supabase        │   OpenAI API         │
│  - pgvector      │   - text-embedding   │
│  - Documents     │   - GPT-4            │
│  - Metadata      │   - Streaming        │
└──────────────────┴──────────────────────┘
```

### Dependencies

```bash
npm install openai @supabase/supabase-js
npm install pdf-parse mammoth  # For PDF and Word doc parsing
npm install langchain @langchain/openai @langchain/community
```

**Alternative (Open Source Stack)**:
```bash
npm install ollama  # Local LLMs (llama3, mistral)
npm install @xenova/transformers  # Local embeddings
# No API costs, runs locally, but slower and needs GPU
```

---

## Implementation Phases

### Phase 1: Document Ingestion (Week 3-4)

**Goal**: Build system to upload and index documents.

#### Step 1: Database Schema

```sql
-- Documents table
CREATE TABLE rag_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  source_type TEXT NOT NULL, -- 'policy', 'statistic', 'workshop', 'report'
  source_org TEXT NOT NULL,  -- 'NTB', 'MoCTCA', 'SDC', 'DIGO', etc.
  file_url TEXT,
  upload_date TIMESTAMP DEFAULT NOW(),
  uploaded_by TEXT,
  metadata JSONB,  -- Additional info like date range, destination, etc.
  is_active BOOLEAN DEFAULT TRUE
);

-- Document chunks (for RAG retrieval)
CREATE TABLE rag_chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES rag_documents(id) ON DELETE CASCADE,
  chunk_text TEXT NOT NULL,
  chunk_index INTEGER NOT NULL,  -- Position in original document
  embedding vector(1536),  -- OpenAI ada-002 produces 1536-dim vectors
  metadata JSONB
);

-- Enable vector similarity search
CREATE INDEX ON rag_chunks USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Query logs (for improving system)
CREATE TABLE rag_query_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_text TEXT NOT NULL,
  user_id TEXT,
  use_case TEXT,  -- 'workshop_prep', 'policy_check', 'report_gen'
  chunks_retrieved INTEGER,
  response_generated TEXT,
  sources_cited JSONB,
  query_time TIMESTAMP DEFAULT NOW(),
  feedback_rating INTEGER  -- User can rate answer quality 1-5
);
```

#### Step 2: Document Upload API

**File**: `src/app/api/rag/upload/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@supabase/supabase-js';
import OpenAI from 'openai';
import pdf from 'pdf-parse';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // Use service role for uploads
);

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY!
});

// Chunk text into smaller pieces for better retrieval
function chunkText(text: string, chunkSize: number = 1000, overlap: number = 200): string[] {
  const chunks: string[] = [];
  let start = 0;
  
  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);
    chunks.push(text.slice(start, end));
    start = end - overlap;  // Overlap chunks to maintain context
  }
  
  return chunks;
}

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;
    const title = formData.get('title') as string;
    const sourceType = formData.get('sourceType') as string;
    const sourceOrg = formData.get('sourceOrg') as string;
    const metadata = JSON.parse(formData.get('metadata') as string || '{}');
    
    // Parse document based on file type
    const buffer = await file.arrayBuffer();
    let documentText = '';
    
    if (file.type === 'application/pdf') {
      const pdfData = await pdf(Buffer.from(buffer));
      documentText = pdfData.text;
    } else if (file.type === 'text/plain') {
      documentText = new TextDecoder().decode(buffer);
    }
    // Add more parsers for Word, Excel, etc.
    
    // Upload file to Supabase Storage
    const fileName = `${Date.now()}-${file.name}`;
    const { data: uploadData, error: uploadError } = await supabase.storage
      .from('rag-documents')
      .upload(fileName, file);
    
    if (uploadError) throw uploadError;
    
    // Create document record
    const { data: doc, error: docError } = await supabase
      .from('rag_documents')
      .insert({
        title,
        source_type: sourceType,
        source_org: sourceOrg,
        file_url: uploadData.path,
        metadata
      })
      .select()
      .single();
    
    if (docError) throw docError;
    
    // Chunk the document
    const chunks = chunkText(documentText);
    
    // Generate embeddings for each chunk
    const chunkRecords = await Promise.all(
      chunks.map(async (chunkText, index) => {
        const embeddingResponse = await openai.embeddings.create({
          model: 'text-embedding-ada-002',
          input: chunkText
        });
        
        const embedding = embeddingResponse.data[0].embedding;
        
        return {
          document_id: doc.id,
          chunk_text: chunkText,
          chunk_index: index,
          embedding: embedding,
          metadata: {
            ...metadata,
            chunk_size: chunkText.length
          }
        };
      })
    );
    
    // Insert chunks into database
    const { error: chunksError } = await supabase
      .from('rag_chunks')
      .insert(chunkRecords);
    
    if (chunksError) throw chunksError;
    
    return NextResponse.json({
      success: true,
      document_id: doc.id,
      chunks_created: chunks.length
    });
    
  } catch (error: any) {
    console.error('Document upload error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}
```

---

### Phase 2: RAG Query System (Week 5-6)

#### Step 1: Query API

**File**: `src/app/api/rag/query/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@supabase/supabase-js';
import OpenAI from 'openai';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY!
});

export async function POST(request: NextRequest) {
  try {
    const { query, useCase, filters } = await request.json();
    
    // 1. Generate embedding for the query
    const embeddingResponse = await openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: query
    });
    const queryEmbedding = embeddingResponse.data[0].embedding;
    
    // 2. Vector similarity search in Supabase
    const { data: chunks, error } = await supabase.rpc('match_documents', {
      query_embedding: queryEmbedding,
      match_threshold: 0.7,  // Minimum similarity score
      match_count: 5,  // Top 5 most relevant chunks
      filter_source_type: filters?.sourceType || null
    });
    
    if (error) throw error;
    
    // 3. Build context from retrieved chunks
    const context = chunks
      .map((chunk: any) => `[Source: ${chunk.source_org} - ${chunk.title}]\n${chunk.chunk_text}`)
      .join('\n\n---\n\n');
    
    // 4. Generate answer with GPT-4
    const systemPrompt = getSystemPrompt(useCase);
    
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo',
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: `Context:\n${context}\n\nQuestion: ${query}` }
      ],
      temperature: 0.3,  // Lower temperature for factual answers
      stream: false
    });
    
    const answer = completion.choices[0].message.content;
    
    // 5. Extract sources
    const sources = chunks.map((chunk: any) => ({
      title: chunk.title,
      source_org: chunk.source_org,
      source_type: chunk.source_type,
      relevance_score: chunk.similarity
    }));
    
    // 6. Log query for analytics
    await supabase.from('rag_query_logs').insert({
      query_text: query,
      use_case: useCase,
      chunks_retrieved: chunks.length,
      response_generated: answer,
      sources_cited: sources
    });
    
    return NextResponse.json({
      answer,
      sources,
      chunks_retrieved: chunks.length
    });
    
  } catch (error: any) {
    console.error('RAG query error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}

function getSystemPrompt(useCase: string): string {
  const prompts = {
    workshop_prep: `You are a DIGO workshop facilitator assistant. Answer questions about tourism statistics, visitor data, and evidence to help prepare for Sustainable Visitor Flow workshops. 

Always cite your sources. If the context doesn't contain the answer, say "I don't have that information in the available documents." Be specific with numbers, dates, and locations.`,

    policy_check: `You are a policy alignment checker for Nepal tourism initiatives. Evaluate whether proposed action plans align with official policies, strategies, and frameworks.

Cite specific sections of policy documents. If there's a conflict, explain it clearly. If alignment cannot be determined from available documents, state that explicitly.`,

    report_gen: `You are an SDC report writer assistant. Generate progress reports based on DIGO platform data, workshop outcomes, and action plan implementations.

Use official SDC reporting format. Include specific metrics, evidence quality tiers, and stakeholder participation data. Maintain professional tone suitable for donor reporting.`
  };
  
  return prompts[useCase as keyof typeof prompts] || prompts.workshop_prep;
}
```

#### Step 2: PostgreSQL Vector Search Function

```sql
-- Function for vector similarity search
CREATE OR REPLACE FUNCTION match_documents (
  query_embedding vector(1536),
  match_threshold float,
  match_count int,
  filter_source_type text DEFAULT NULL
)
RETURNS TABLE (
  id uuid,
  document_id uuid,
  chunk_text text,
  chunk_index int,
  similarity float,
  title text,
  source_type text,
  source_org text,
  metadata jsonb
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    rc.id,
    rc.document_id,
    rc.chunk_text,
    rc.chunk_index,
    1 - (rc.embedding <=> query_embedding) as similarity,
    rd.title,
    rd.source_type,
    rd.source_org,
    rc.metadata
  FROM rag_chunks rc
  JOIN rag_documents rd ON rc.document_id = rd.id
  WHERE 
    rd.is_active = true
    AND (filter_source_type IS NULL OR rd.source_type = filter_source_type)
    AND 1 - (rc.embedding <=> query_embedding) > match_threshold
  ORDER BY rc.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

---

### Phase 3: UI Components (Week 7)

#### Component 1: RAG Query Interface

**File**: `src/components/rag/RAGQueryInterface.tsx`

```typescript
'use client';

import { useState } from 'react';
import { Loader2, Search, FileText, CheckCircle, TrendingUp } from 'lucide-react';

type UseCase = 'workshop_prep' | 'policy_check' | 'report_gen';

interface Source {
  title: string;
  source_org: string;
  source_type: string;
  relevance_score: number;
}

export default function RAGQueryInterface() {
  const [query, setQuery] = useState('');
  const [useCase, setUseCase] = useState<UseCase>('workshop_prep');
  const [loading, setLoading] = useState(false);
  const [answer, setAnswer] = useState('');
  const [sources, setSources] = useState<Source[]>([]);

  const useCaseOptions = [
    { 
      value: 'workshop_prep', 
      label: 'Workshop Preparation', 
      icon: Search,
      color: 'blue',
      examples: [
        'What does NTB data show about Pokhara visitor arrivals in Q3 2025?',
        'Show me ACAP permit volumes for Annapurna trekking routes',
        'What are hotel occupancy rates in Lakeside during peak season?'
      ]
    },
    { 
      value: 'policy_check', 
      label: 'Policy Alignment Check', 
      icon: CheckCircle,
      color: 'green',
      examples: [
        'Does this homestay initiative align with Nepal Tourism Strategy?',
        'Check if our visitor dispersal plan meets DIGO criteria',
        'Is this infrastructure proposal consistent with SDC guidelines?'
      ]
    },
    { 
      value: 'report_gen', 
      label: 'Report Generation', 
      icon: TrendingUp,
      color: 'purple',
      examples: [
        'Draft Q2 2026 SDC progress report',
        'Summarize Pokhara action plan outcomes with evidence',
        'Generate impact narrative for women-led tourism initiatives'
      ]
    }
  ];

  const currentUseCase = useCaseOptions.find(uc => uc.value === useCase)!;

  const handleQuery = async () => {
    if (!query.trim()) return;
    
    setLoading(true);
    setAnswer('');
    setSources([]);
    
    try {
      const response = await fetch('/api/rag/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query, useCase })
      });
      
      const data = await response.json();
      setAnswer(data.answer);
      setSources(data.sources);
    } catch (error) {
      console.error('Query error:', error);
      setAnswer('Error processing query. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-5xl mx-auto p-6 space-y-6">
      {/* Use Case Selector */}
      <div className="grid grid-cols-3 gap-4">
        {useCaseOptions.map((option) => {
          const Icon = option.icon;
          const isSelected = useCase === option.value;
          
          return (
            <button
              key={option.value}
              onClick={() => setUseCase(option.value as UseCase)}
              className={`p-4 rounded-lg border-2 transition-all ${
                isSelected
                  ? `border-${option.color}-500 bg-${option.color}-50`
                  : 'border-gray-200 hover:border-gray-300'
              }`}
            >
              <div className="flex items-center gap-2 mb-2">
                <Icon className={`w-5 h-5 ${isSelected ? `text-${option.color}-600` : 'text-gray-600'}`} />
                <span className="font-semibold text-sm">{option.label}</span>
              </div>
            </button>
          );
        })}
      </div>

      {/* Example Queries */}
      <div className="bg-blue-50 border border-blue-200 rounded-lg p-4">
        <h3 className="font-semibold text-sm text-blue-900 mb-2">Example Queries:</h3>
        <ul className="space-y-1">
          {currentUseCase.examples.map((example, i) => (
            <li key={i}>
              <button
                onClick={() => setQuery(example)}
                className="text-sm text-blue-700 hover:text-blue-900 hover:underline text-left"
              >
                • {example}
              </button>
            </li>
          ))}
        </ul>
      </div>

      {/* Query Input */}
      <div className="flex gap-3">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleQuery()}
          placeholder="Ask a question..."
          className="flex-1 px-4 py-3 border-2 border-gray-300 rounded-lg focus:border-blue-500 focus:outline-none"
        />
        <button
          onClick={handleQuery}
          disabled={loading || !query.trim()}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed flex items-center gap-2"
        >
          {loading ? (
            <>
              <Loader2 className="w-5 h-5 animate-spin" />
              Searching...
            </>
          ) : (
            <>
              <Search className="w-5 h-5" />
              Ask
            </>
          )}
        </button>
      </div>

      {/* Answer Display */}
      {answer && (
        <div className="bg-white border-2 border-gray-200 rounded-lg p-6 space-y-4">
          <div className="prose max-w-none">
            <h3 className="text-lg font-bold mb-3">Answer:</h3>
            <div className="text-gray-800 whitespace-pre-wrap">{answer}</div>
          </div>

          {/* Sources */}
          {sources.length > 0 && (
            <div className="pt-4 border-t border-gray-200">
              <h4 className="font-semibold text-sm mb-3 flex items-center gap-2">
                <FileText className="w-4 h-4" />
                Sources ({sources.length})
              </h4>
              <div className="space-y-2">
                {sources.map((source, i) => (
                  <div key={i} className="flex items-start gap-3 text-sm bg-gray-50 p-3 rounded">
                    <div className="flex-1">
                      <div className="font-medium">{source.title}</div>
                      <div className="text-gray-600 text-xs">
                        {source.source_org} • {source.source_type}
                      </div>
                    </div>
                    <div className="text-xs text-gray-500">
                      {Math.round(source.relevance_score * 100)}% match
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

#### Component 2: Document Upload UI

**File**: `src/components/rag/DocumentUploader.tsx`

```typescript
'use client';

import { useState } from 'react';
import { Upload, FileText, CheckCircle, AlertCircle } from 'lucide-react';

export default function DocumentUploader() {
  const [file, setFile] = useState<File | null>(null);
  const [title, setTitle] = useState('');
  const [sourceType, setSourceType] = useState('statistic');
  const [sourceOrg, setSourceOrg] = useState('');
  const [uploading, setUploading] = useState(false);
  const [success, setSuccess] = useState(false);

  const handleUpload = async () => {
    if (!file || !title || !sourceOrg) return;

    setUploading(true);
    setSuccess(false);

    const formData = new FormData();
    formData.append('file', file);
    formData.append('title', title);
    formData.append('sourceType', sourceType);
    formData.append('sourceOrg', sourceOrg);
    formData.append('metadata', JSON.stringify({ upload_date: new Date().toISOString() }));

    try {
      const response = await fetch('/api/rag/upload', {
        method: 'POST',
        body: formData
      });

      if (response.ok) {
        setSuccess(true);
        setFile(null);
        setTitle('');
        setSourceOrg('');
      }
    } catch (error) {
      console.error('Upload error:', error);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-6 bg-white rounded-lg shadow-lg">
      <h2 className="text-2xl font-bold mb-6">Upload Document to Knowledge Base</h2>

      <div className="space-y-4">
        {/* File Upload */}
        <div>
          <label className="block text-sm font-medium mb-2">Document File</label>
          <div className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center">
            <input
              type="file"
              accept=".pdf,.txt,.docx"
              onChange={(e) => setFile(e.target.files?.[0] || null)}
              className="hidden"
              id="file-upload"
            />
            <label htmlFor="file-upload" className="cursor-pointer">
              <Upload className="w-12 h-12 mx-auto text-gray-400 mb-2" />
              <p className="text-sm text-gray-600">
                {file ? file.name : 'Click to upload PDF, TXT, or DOCX'}
              </p>
            </label>
          </div>
        </div>

        {/* Title */}
        <div>
          <label className="block text-sm font-medium mb-2">Document Title</label>
          <input
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            placeholder="e.g., NTB Annual Tourism Statistics 2025"
            className="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500"
          />
        </div>

        {/* Source Type */}
        <div>
          <label className="block text-sm font-medium mb-2">Document Type</label>
          <select
            value={sourceType}
            onChange={(e) => setSourceType(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500"
          >
            <option value="statistic">Tourism Statistics</option>
            <option value="policy">Policy Document</option>
            <option value="workshop">Workshop Output</option>
            <option value="report">Progress Report</option>
          </select>
        </div>

        {/* Source Organization */}
        <div>
          <label className="block text-sm font-medium mb-2">Source Organization</label>
          <select
            value={sourceOrg}
            onChange={(e) => setSourceOrg(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500"
          >
            <option value="">Select organization...</option>
            <option value="NTB">Nepal Tourism Board (NTB)</option>
            <option value="MoCTCA">Ministry of Culture, Tourism & Civil Aviation</option>
            <option value="SDC">Swiss Agency for Development and Cooperation</option>
            <option value="DIGO">DIGO Project</option>
            <option value="ACAP">Annapurna Conservation Area Project</option>
            <option value="Municipality">Municipal Government</option>
          </select>
        </div>

        {/* Upload Button */}
        <button
          onClick={handleUpload}
          disabled={!file || !title || !sourceOrg || uploading}
          className="w-full py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed flex items-center justify-center gap-2"
        >
          {uploading ? (
            <>
              <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white" />
              Uploading and indexing...
            </>
          ) : (
            <>
              <FileText className="w-5 h-5" />
              Upload Document
            </>
          )}
        </button>

        {/* Success Message */}
        {success && (
          <div className="flex items-center gap-2 p-4 bg-green-50 border border-green-200 rounded-lg">
            <CheckCircle className="w-5 h-5 text-green-600" />
            <span className="text-green-800">Document uploaded and indexed successfully!</span>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## Usage Examples

### Example 1: Workshop Preparation

```typescript
// Facilitator preparing for Pokhara workshop
const query = "What does official data show about Lakeside visitor volumes in Q3 2025?";

// RAG System:
// 1. Searches through NTB statistics, hotel occupancy reports
// 2. Finds relevant chunks
// 3. Generates answer with GPT-4

// Response:
{
  answer: "According to NTB statistics for Q3 2025 (July-September), Lakeside area in Pokhara recorded approximately 45,000 tourist arrivals, representing a 12% increase from Q3 2024. Hotel occupancy data from Pokhara Hotel Association shows average occupancy of 68% during this period, with peak occupancy of 85% in September during festival season. Domestic tourists comprised 62% of visitors, with international tourists (primarily from India, China, and South Korea) making up 38%.",
  
  sources: [
    {
      title: "NTB Quarterly Tourism Statistics Q3 2025",
      source_org: "Nepal Tourism Board",
      source_type: "statistic",
      relevance_score: 0.89
    },
    {
      title: "Pokhara Hotel Association Occupancy Report July-Sept 2025",
      source_org: "Pokhara Hotel Association",
      source_type: "statistic",
      relevance_score: 0.82
    }
  ]
}
```

### Example 2: Policy Alignment Check

```typescript
const actionPlan = `
Action Plan: Establish women-led homestay network in Madi RM with capacity building training, 
marketing support, and direct booking platform to reduce dependency on external tour operators.
`;

const query = `Does this action plan align with Nepal's National Tourism Strategy and DIGO sustainable development criteria?

Action Plan: ${actionPlan}`;

// Response:
{
  answer: "Yes, this action plan strongly aligns with both frameworks:

**Nepal National Tourism Strategy 2022-2032 Alignment:**
- Directly supports Strategic Pillar 3: 'Community-Based Tourism Development' (Section 4.3)
- Addresses objective to 'increase local economic benefit retention' (page 34)
- Aligns with gender equity goals under inclusive tourism (Section 5.2)

**DIGO Sustainable Development Criteria Alignment:**
- Meets Criterion 2: 'Economic Leakage Reduction' by building local capacity
- Supports Criterion 5: 'Women's Economic Empowerment' through leadership roles
- Consistent with Criterion 7: 'Direct Community Benefit Mechanisms'

**Recommendation:** This action plan is well-aligned. Consider adding monitoring indicators for women's income increases to strengthen SDC reporting.",

  sources: [...]
}
```

### Example 3: SDC Report Generation

```typescript
const query = "Draft Q2 2026 progress report for SDC highlighting Madi RM outcomes";

// System pulls from:
// - Workshop records (participation, stakeholder engagement)
// - Flow data (visitor counts, pressure analysis)
// - Action plan tracking (implementation status)
// - Evidence quality metadata

// Response:
{
  answer: `
# DIGO Project Progress Report - Q2 2026
**Location:** Madi Rural Municipality
**Reporting Period:** April - June 2026
**Submitted to:** Swiss Agency for Development and Cooperation (SDC)

## Executive Summary
During Q2 2026, DIGO conducted 3 participatory workshops in Madi RM with 47 stakeholders, mapping 8 sustainable visitor flows and identifying critical pressure points at community forest and river tourism areas.

## Key Outcomes

### 1. Stakeholder Engagement (SDC Indicator 1.2)
- **Workshops Conducted:** 3 (April 12, May 8, June 15, 2026)
- **Participants:** 47 total (28 women, 19 men)
- **Stakeholder Categories:** Women's homestay cooperative (15), local guides (12), municipal staff (8), conservation committee (7), transport operators (5)

### 2. Visitor Flow Analysis (SDC Indicator 2.1)
- **Flows Mapped:** 8 complete visitor journeys
- **Evidence Quality:** 
  - Tier 1 (High Confidence): 62.5% of data points
  - Tier 2 (Medium Confidence): 25%
  - Tier 3 (Low Confidence): 12.5%
- **Data Quality Score:** 81% (exceeds 75% target)

### 3. Economic Leakage Reduction (SDC Indicator 3.3)
- **Baseline Leakage:** 82% for external package tours
- **Target Leakage (Women's Homestay Initiative):** 12%
- **Potential Local Benefit Increase:** NPR 2.4M annually if 30% of visitors shift to homestay network

...
`,
  sources: [...]
}
```

---

## Cost Estimation

### OpenAI API Costs (Monthly for active DIGO usage)

**Assumptions:**
- 50 documents uploaded per month (policy docs, statistics)
- Average 20 pages per document = 1000 documents total
- 100 queries per month from facilitators
- Using GPT-4-turbo + text-embedding-ada-002

**Breakdown:**
```
Document Embedding (one-time per document):
- 50 docs × 20 pages × 500 words = 500,000 words
- ~650,000 tokens at $0.0001/1K tokens = $0.065

Query Processing (monthly):
- 100 queries × 5 chunks × 500 tokens = 250,000 tokens
- Embedding: $0.025
- GPT-4 generation: 100 queries × 2000 tokens × $0.01/1K = $2.00

Total Monthly Cost: ~$2.50 (after initial document indexing)
```

**Alternative: Open Source (Free)**
- Use Ollama with llama3 or mistral (local LLM)
- Use local embeddings (Xenova transformers)
- Cost: $0, but requires GPU and slower performance

---

## Success Metrics

### Workshop Preparation Intelligence
- ✅ Average query response time < 10 seconds
- ✅ 80%+ of queries answered with sources
- ✅ Facilitators save 2+ hours per workshop on data gathering

### Policy Alignment Checking
- ✅ 95%+ accuracy in identifying alignment issues
- ✅ All action plans checked before municipal approval
- ✅ Zero policy conflicts in deployed initiatives

### SDC Reporting Assistance
- ✅ Report drafting time reduced from 8 hours to 2 hours
- ✅ 100% of required SDC metrics auto-populated
- ✅ Zero errors in data citation and evidence quality

---

## Next Steps for Intern

### Week 3-4: Foundation
1. Set up Supabase with pgvector extension
2. Create database schema for documents and chunks
3. Implement document upload API with PDF parsing
4. Test embedding generation with sample documents

### Week 5-6: Core RAG
1. Build vector similarity search function
2. Implement RAG query API
3. Test retrieval quality with sample queries
4. Add query logging and analytics

### Week 7: User Interface
1. Build RAG query interface component
2. Create document uploader UI
3. Add source citation display
4. Implement feedback collection

### Week 8: Production Ready
1. Upload initial document corpus (NTB stats, policy docs)
2. User testing with DIGO facilitators
3. Performance optimization
4. Documentation and training materials

---

## Security Considerations

1. **Access Control**: Only authorized users (DIGO staff, municipality admins) can upload documents
2. **Document Visibility**: Implement row-level security to control who can query which documents
3. **API Key Management**: Store OpenAI API keys securely in environment variables
4. **Rate Limiting**: Prevent API abuse with rate limits on query endpoint
5. **Data Privacy**: Don't upload personally identifiable information or sensitive municipal data

---

## Questions?

**Technical Questions**: 
- "How does vector similarity work?" → Read Supabase pgvector docs
- "Why chunk documents?" → Chunking improves retrieval precision
- "Can we use free alternatives?" → Yes, Ollama for local LLMs

**Implementation Questions**:
- Start simple: PDF upload + embedding + search
- Iterate: Add more document types, improve prompts
- Test early: Try with real DIGO facilitator questions

Good luck building the DIGO Intelligence Layer! 🧠
