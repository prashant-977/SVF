# Planning Intelligence Assistant - Implementation Guide

## Overview

The **Planning Intelligence Assistant** is a specialized RAG system designed for DIGO facilitators and municipality planners. Unlike basic RAG, it uses a **multi-criteria ranking layer** to score retrieved information by policy alignment, evidence quality, and SDC reporting relevance.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PLANNING INTELLIGENCE ASSISTANT          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  CORPUS: Document Base                                      │
│  - NTB statistical bulletins                                │
│  - ACAP annual visitor reports                              │
│  - MoCTCA policy docs                                       │
│  - DIGO workshop outputs                                    │
│  - SDC programme frameworks                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  USER: Municipality planner, DIGO facilitator               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  QUERY: "What does NTB data show about off-peak visitor     │
│          potential in Madi?"                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  RETRIEVAL: Vector similarity search (pgvector)             │
│  - Get top 20 candidate chunks                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  RANKING LAYER (Multi-Criteria Scoring)                     │
│                                                             │
│  For each chunk, calculate:                                 │
│  1. Policy Alignment Score      (0-1)                       │
│  2. Data Confidence Tier        (tier1=1, tier2=0.6, tier3=0.2) │
│  3. SDC Reporting Relevance     (0-1)                       │
│                                                             │
│  Final Score = weighted sum of above                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  RE-RANKING: Sort by final score, take top 5                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  OUTPUT: Planning evidence with policy citations            │
│  - Prioritizes high-quality, policy-aligned, SDC-relevant   │
│  - Each answer includes confidence indicators               │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Innovation: Multi-Criteria Ranking Layer

### Standard RAG Problem
Basic RAG systems only use **semantic similarity** - they find text that's *similar to the query*, but don't consider:
- Whether it aligns with official policies
- Whether the data is trustworthy (evidence tier)
- Whether it's relevant for donor reporting

### Planning Intelligence Solution
Add a **ranking layer** that scores each retrieved chunk on multiple dimensions:

#### 1. Policy Alignment Score (0-1)
**Question**: Does this chunk support or conflict with Nepal's tourism policies?

**Calculation**:
```typescript
function calculatePolicyAlignmentScore(chunk: Chunk, query: string): number {
  // Check if chunk comes from policy document
  if (chunk.source_type === 'policy') {
    return 1.0;  // Policy docs have high alignment by definition
  }
  
  // Check if chunk references policy frameworks
  const policyKeywords = [
    'National Tourism Strategy',
    'sustainable development',
    'community-based tourism',
    'visitor dispersal',
    'economic leakage reduction'
  ];
  
  let score = 0;
  for (const keyword of policyKeywords) {
    if (chunk.text.toLowerCase().includes(keyword.toLowerCase())) {
      score += 0.2;
    }
  }
  
  // Check if metadata includes policy tags
  if (chunk.metadata?.policy_alignment) {
    score += 0.3;
  }
  
  return Math.min(score, 1.0);
}
```

**Advanced Version**: Use GPT-4 to assess alignment
```typescript
async function assessPolicyAlignment(chunk: Chunk): Promise<number> {
  const prompt = `
Rate how well this text aligns with Nepal's National Tourism Strategy 2022-2032
on a scale of 0.0 to 1.0:

Text: "${chunk.text}"

Consider:
- Does it support visitor dispersal objectives?
- Does it promote sustainable tourism?
- Does it reduce economic leakage?
- Does it empower local communities?

Return only a number between 0.0 and 1.0.
`;

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.1
  });
  
  return parseFloat(response.choices[0].message.content);
}
```

---

#### 2. Data Confidence Tier (0.2 - 1.0)
**Question**: How trustworthy is this data?

**Mapping**:
```typescript
function getConfidenceScore(evidenceTier: 'tier1' | 'tier2' | 'tier3'): number {
  const scores = {
    tier1: 1.0,   // High confidence - multiple validated sources
    tier2: 0.6,   // Medium confidence - stakeholder consensus
    tier3: 0.2    // Low confidence - single source, needs verification
  };
  return scores[evidenceTier];
}
```

**Why This Matters**:
- Municipality planners need to prioritize **reliable data** for decision-making
- Tier 3 data can be flagged for validation workshops
- SDC reports require evidence quality tracking

**Example**:
```
Chunk A: "12,000 tourists visit Fewa Lake daily" (Tier 3 - boat operator guess)
  → Confidence Score: 0.2

Chunk B: "11,200 tourists visit Fewa Lake daily" (Tier 1 - ticket sales + surveys)
  → Confidence Score: 1.0

Chunk B gets prioritized even if Chunk A has slightly higher semantic similarity!
```

---

#### 3. SDC Reporting Relevance (0-1)
**Question**: Is this useful for SDC donor reporting?

**Calculation**:
```typescript
function calculateSDCRelevance(chunk: Chunk, query: string): number {
  let score = 0;
  
  // SDC cares about quantitative outcomes
  if (hasQuantitativeData(chunk.text)) {
    score += 0.3;
  }
  
  // SDC indicators: gender, economic impact, sustainability
  const sdcIndicators = [
    'women-led',
    'gender equity',
    'economic leakage',
    'local benefit',
    'homestay',
    'community-based',
    'sustainable',
    'capacity building'
  ];
  
  for (const indicator of sdcIndicators) {
    if (chunk.text.toLowerCase().includes(indicator)) {
      score += 0.15;
    }
  }
  
  // Bonus if chunk includes outcome metrics
  if (chunk.metadata?.has_outcome_metrics) {
    score += 0.2;
  }
  
  // Bonus if from validated workshop (SDC wants participatory evidence)
  if (chunk.source_type === 'workshop' && chunk.metadata?.participants >= 10) {
    score += 0.2;
  }
  
  return Math.min(score, 1.0);
}

function hasQuantitativeData(text: string): boolean {
  // Check for numbers, percentages, statistics
  const numberPattern = /\d+(?:\.\d+)?(?:\s*%|\s*tourists|\s*NPR|\s*visitors)/i;
  return numberPattern.test(text);
}
```

---

### Combined Ranking Score

```typescript
interface RankingWeights {
  policyAlignment: number;
  dataConfidence: number;
  sdcRelevance: number;
}

// Default weights (can be adjusted per query type)
const DEFAULT_WEIGHTS: RankingWeights = {
  policyAlignment: 0.35,   // 35% weight
  dataConfidence: 0.40,    // 40% weight (most important for planners)
  sdcRelevance: 0.25       // 25% weight
};

function calculateFinalScore(
  chunk: Chunk,
  semanticSimilarity: number,
  query: string,
  weights: RankingWeights = DEFAULT_WEIGHTS
): number {
  const policyScore = calculatePolicyAlignmentScore(chunk, query);
  const confidenceScore = getConfidenceScore(chunk.evidence_tier);
  const sdcScore = calculateSDCRelevance(chunk, query);
  
  // Combine semantic similarity with ranking criteria
  const rankingScore = 
    (policyScore * weights.policyAlignment) +
    (confidenceScore * weights.dataConfidence) +
    (sdcScore * weights.sdcRelevance);
  
  // Final score is weighted average of semantic similarity and ranking score
  const finalScore = (semanticSimilarity * 0.5) + (rankingScore * 0.5);
  
  return finalScore;
}
```

---

## Database Schema Updates

### Enhanced Chunks Table

```sql
-- Extend rag_chunks with ranking metadata
ALTER TABLE rag_chunks ADD COLUMN policy_alignment_score DECIMAL(3,2);
ALTER TABLE rag_chunks ADD COLUMN evidence_tier TEXT CHECK (evidence_tier IN ('tier1', 'tier2', 'tier3'));
ALTER TABLE rag_chunks ADD COLUMN sdc_relevance_score DECIMAL(3,2);
ALTER TABLE rag_chunks ADD COLUMN has_quantitative_data BOOLEAN DEFAULT FALSE;

-- Create index for faster filtering
CREATE INDEX idx_chunks_evidence_tier ON rag_chunks(evidence_tier);
CREATE INDEX idx_chunks_policy_score ON rag_chunks(policy_alignment_score DESC);

-- Update metadata JSONB structure
-- Example metadata:
{
  "source_date": "2025-10-15",
  "participants": 15,  // For workshop chunks
  "has_outcome_metrics": true,
  "policy_references": ["National Tourism Strategy Section 4.3"],
  "sdc_indicators": ["women-led", "economic leakage"],
  "validated_by": "Municipality & DIGO cross-verification",
  "validation_date": "2026-05-05"
}
```

---

## Implementation: Enhanced RAG Query API

**File**: `src/app/api/planning-intelligence/query/route.ts`

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

interface RankingWeights {
  policyAlignment: number;
  dataConfidence: number;
  sdcRelevance: number;
}

export async function POST(request: NextRequest) {
  try {
    const { 
      query, 
      queryType = 'planning',  // 'planning', 'policy_check', 'sdc_report'
      prioritizeHighQuality = true 
    } = await request.json();
    
    // Step 1: Generate query embedding
    const embeddingResponse = await openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: query
    });
    const queryEmbedding = embeddingResponse.data[0].embedding;
    
    // Step 2: Vector similarity search (get more candidates than we need)
    const { data: candidateChunks, error } = await supabase.rpc('match_documents', {
      query_embedding: queryEmbedding,
      match_threshold: 0.5,  // Lower threshold to get more candidates
      match_count: 20,  // Get 20 candidates, will re-rank to top 5
    });
    
    if (error) throw error;
    
    // Step 3: Apply multi-criteria ranking
    const weights = getWeightsForQueryType(queryType);
    
    const rankedChunks = candidateChunks
      .map(chunk => ({
        ...chunk,
        policyScore: calculatePolicyAlignmentScore(chunk, query),
        confidenceScore: getConfidenceScore(chunk.evidence_tier || 'tier3'),
        sdcScore: calculateSDCRelevance(chunk, query),
        finalScore: calculateFinalScore(chunk, chunk.similarity, query, weights)
      }))
      .sort((a, b) => b.finalScore - a.finalScore)
      .slice(0, 5);  // Take top 5 after ranking
    
    // Step 4: Build context from top-ranked chunks
    const context = rankedChunks
      .map((chunk, i) => `
[Source ${i+1}: ${chunk.source_org} - ${chunk.title}]
[Evidence Quality: ${chunk.evidence_tier?.toUpperCase() || 'UNKNOWN'}]
[Policy Alignment: ${(chunk.policyScore * 100).toFixed(0)}%]
[SDC Relevance: ${(chunk.sdcScore * 100).toFixed(0)}%]

${chunk.chunk_text}
      `)
      .join('\n\n---\n\n');
    
    // Step 5: Generate answer with GPT-4
    const systemPrompt = getPlanningSystemPrompt(queryType);
    
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo',
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: `Context:\n${context}\n\nQuestion: ${query}` }
      ],
      temperature: 0.3
    });
    
    const answer = completion.choices[0].message.content;
    
    // Step 6: Format sources with ranking scores
    const sources = rankedChunks.map(chunk => ({
      title: chunk.title,
      source_org: chunk.source_org,
      source_type: chunk.source_type,
      evidence_tier: chunk.evidence_tier || 'tier3',
      semantic_similarity: chunk.similarity,
      policy_alignment: chunk.policyScore,
      data_confidence: chunk.confidenceScore,
      sdc_relevance: chunk.sdcScore,
      final_score: chunk.finalScore,
      // Human-readable quality indicator
      quality_label: getQualityLabel(chunk.confidenceScore),
      policy_label: getPolicyLabel(chunk.policyScore)
    }));
    
    // Step 7: Generate confidence summary
    const confidenceSummary = generateConfidenceSummary(rankedChunks);
    
    return NextResponse.json({
      answer,
      sources,
      confidence_summary: confidenceSummary,
      ranking_method: 'multi_criteria',
      weights_used: weights
    });
    
  } catch (error: any) {
    console.error('Planning Intelligence query error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}

// Helper Functions

function getWeightsForQueryType(queryType: string): RankingWeights {
  const weightProfiles = {
    planning: {
      policyAlignment: 0.35,
      dataConfidence: 0.40,  // Planners need reliable data
      sdcRelevance: 0.25
    },
    policy_check: {
      policyAlignment: 0.60,  // Policy checks prioritize alignment
      dataConfidence: 0.20,
      sdcRelevance: 0.20
    },
    sdc_report: {
      policyAlignment: 0.25,
      dataConfidence: 0.35,
      sdcRelevance: 0.40  // SDC reports prioritize donor-relevant metrics
    }
  };
  
  return weightProfiles[queryType as keyof typeof weightProfiles] || weightProfiles.planning;
}

function calculatePolicyAlignmentScore(chunk: any, query: string): number {
  let score = 0;
  
  if (chunk.source_type === 'policy') {
    score += 0.5;
  }
  
  const policyKeywords = [
    'National Tourism Strategy',
    'sustainable development',
    'community-based tourism',
    'visitor dispersal',
    'economic leakage',
    'DIGO criteria'
  ];
  
  for (const keyword of policyKeywords) {
    if (chunk.chunk_text.toLowerCase().includes(keyword.toLowerCase())) {
      score += 0.1;
    }
  }
  
  if (chunk.metadata?.policy_alignment) {
    score += 0.3;
  }
  
  return Math.min(score, 1.0);
}

function getConfidenceScore(evidenceTier: string): number {
  const scores: Record<string, number> = {
    tier1: 1.0,
    tier2: 0.6,
    tier3: 0.2
  };
  return scores[evidenceTier] || 0.2;
}

function calculateSDCRelevance(chunk: any, query: string): number {
  let score = 0;
  
  // Check for quantitative data
  const hasNumbers = /\d+(?:\.\d+)?(?:\s*%|\s*tourists|\s*NPR|\s*visitors)/i.test(chunk.chunk_text);
  if (hasNumbers) {
    score += 0.3;
  }
  
  // SDC indicator keywords
  const sdcIndicators = [
    'women-led',
    'gender',
    'economic leakage',
    'local benefit',
    'homestay',
    'community-based',
    'sustainable',
    'capacity building',
    'stakeholder participation'
  ];
  
  for (const indicator of sdcIndicators) {
    if (chunk.chunk_text.toLowerCase().includes(indicator)) {
      score += 0.1;
    }
  }
  
  // Workshop outputs are valuable for SDC (participatory evidence)
  if (chunk.source_type === 'workshop') {
    score += 0.2;
  }
  
  return Math.min(score, 1.0);
}

function calculateFinalScore(
  chunk: any,
  semanticSimilarity: number,
  query: string,
  weights: RankingWeights
): number {
  const policyScore = calculatePolicyAlignmentScore(chunk, query);
  const confidenceScore = getConfidenceScore(chunk.evidence_tier);
  const sdcScore = calculateSDCRelevance(chunk, query);
  
  const rankingScore = 
    (policyScore * weights.policyAlignment) +
    (confidenceScore * weights.dataConfidence) +
    (sdcScore * weights.sdcRelevance);
  
  // Weighted combination of semantic similarity and ranking criteria
  return (semanticSimilarity * 0.5) + (rankingScore * 0.5);
}

function getQualityLabel(confidenceScore: number): string {
  if (confidenceScore >= 0.9) return 'High Quality - Multiple Sources';
  if (confidenceScore >= 0.5) return 'Medium Quality - Stakeholder Consensus';
  return 'Low Quality - Needs Verification';
}

function getPolicyLabel(policyScore: number): string {
  if (policyScore >= 0.7) return 'Strongly Aligned';
  if (policyScore >= 0.4) return 'Partially Aligned';
  return 'No Policy Reference';
}

function generateConfidenceSummary(chunks: any[]): any {
  const tier1Count = chunks.filter(c => c.evidence_tier === 'tier1').length;
  const tier2Count = chunks.filter(c => c.evidence_tier === 'tier2').length;
  const tier3Count = chunks.filter(c => c.evidence_tier === 'tier3').length;
  
  const avgPolicyAlignment = chunks.reduce((sum, c) => sum + c.policyScore, 0) / chunks.length;
  const avgSDCRelevance = chunks.reduce((sum, c) => sum + c.sdcScore, 0) / chunks.length;
  
  return {
    evidence_quality: {
      tier1_sources: tier1Count,
      tier2_sources: tier2Count,
      tier3_sources: tier3Count,
      overall_confidence: tier1Count >= 3 ? 'high' : tier3Count >= 2 ? 'low' : 'medium'
    },
    policy_alignment: {
      average_score: avgPolicyAlignment,
      level: avgPolicyAlignment >= 0.7 ? 'strong' : avgPolicyAlignment >= 0.4 ? 'moderate' : 'weak'
    },
    sdc_reporting: {
      average_relevance: avgSDCRelevance,
      suitable_for_reporting: avgSDCRelevance >= 0.6
    }
  };
}

function getPlanningSystemPrompt(queryType: string): string {
  return `You are the DIGO Planning Intelligence Assistant helping municipality planners and DIGO facilitators make evidence-based tourism decisions.

Your role:
- Provide planning evidence with policy citations
- Prioritize high-quality data (Tier 1 > Tier 2 > Tier 3)
- Reference Nepal's tourism policy frameworks
- Include SDC-relevant metrics when available

Important:
- Always cite evidence tiers: "According to Tier 1 data from NTB..."
- Flag low-quality data: "Note: This estimate is Tier 3 and needs validation"
- Connect findings to policy: "This aligns with National Tourism Strategy Section X..."
- Include quantitative data when available
- Be transparent about data limitations

Format your answer as:

**Key Findings:**
[Bullet points with evidence tier indicators]

**Policy Alignment:**
[How this relates to Nepal Tourism Strategy, DIGO criteria, etc.]

**Data Quality Notes:**
[Any Tier 3 data that needs validation]

**Recommendation:**
[Actionable next steps for planners]`;
}
```

---

## UI Component: Planning Intelligence Interface

**File**: `src/components/planning/PlanningIntelligenceInterface.tsx`

```typescript
'use client';

import { useState } from 'react';
import { Search, FileText, AlertTriangle, CheckCircle, TrendingUp } from 'lucide-react';

interface Source {
  title: string;
  source_org: string;
  evidence_tier: string;
  policy_alignment: number;
  data_confidence: number;
  sdc_relevance: number;
  final_score: number;
  quality_label: string;
  policy_label: string;
}

interface ConfidenceSummary {
  evidence_quality: {
    tier1_sources: number;
    tier2_sources: number;
    tier3_sources: number;
    overall_confidence: 'high' | 'medium' | 'low';
  };
  policy_alignment: {
    average_score: number;
    level: 'strong' | 'moderate' | 'weak';
  };
  sdc_reporting: {
    average_relevance: number;
    suitable_for_reporting: boolean;
  };
}

export default function PlanningIntelligenceInterface() {
  const [query, setQuery] = useState('');
  const [loading, setLoading] = useState(false);
  const [answer, setAnswer] = useState('');
  const [sources, setSources] = useState<Source[]>([]);
  const [confidenceSummary, setConfidenceSummary] = useState<ConfidenceSummary | null>(null);

  const exampleQueries = [
    'What does NTB data show about off-peak visitor potential in Madi?',
    'What evidence exists for economic leakage in Pokhara tourism?',
    'How can we reduce pressure on Fewa Lake waterfront?',
    'What outcomes have women-led homestay initiatives achieved?'
  ];

  const handleQuery = async () => {
    if (!query.trim()) return;
    
    setLoading(true);
    setAnswer('');
    setSources([]);
    setConfidenceSummary(null);
    
    try {
      const response = await fetch('/api/planning-intelligence/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
          query,
          queryType: 'planning',
          prioritizeHighQuality: true
        })
      });
      
      const data = await response.json();
      setAnswer(data.answer);
      setSources(data.sources);
      setConfidenceSummary(data.confidence_summary);
    } catch (error) {
      console.error('Query error:', error);
      setAnswer('Error processing query. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-6xl mx-auto p-6 space-y-6">
      <div className="bg-gradient-to-r from-blue-600 to-blue-800 text-white p-6 rounded-lg">
        <h1 className="text-2xl font-bold mb-2">Planning Intelligence Assistant</h1>
        <p className="text-blue-100">
          Evidence-based answers for municipality planners and DIGO facilitators
        </p>
      </div>

      {/* Example Queries */}
      <div className="bg-blue-50 border border-blue-200 rounded-lg p-4">
        <h3 className="font-semibold text-sm text-blue-900 mb-2">Example Questions:</h3>
        <div className="space-y-2">
          {exampleQueries.map((example, i) => (
            <button
              key={i}
              onClick={() => setQuery(example)}
              className="block w-full text-left text-sm text-blue-700 hover:text-blue-900 hover:bg-blue-100 p-2 rounded transition"
            >
              {example}
            </button>
          ))}
        </div>
      </div>

      {/* Query Input */}
      <div className="flex gap-3">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleQuery()}
          placeholder="Ask about tourism data, policies, or planning evidence..."
          className="flex-1 px-4 py-3 border-2 border-gray-300 rounded-lg focus:border-blue-500 focus:outline-none"
        />
        <button
          onClick={handleQuery}
          disabled={loading || !query.trim()}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed flex items-center gap-2"
        >
          {loading ? (
            <>
              <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white" />
              Analyzing...
            </>
          ) : (
            <>
              <Search className="w-5 h-5" />
              Ask
            </>
          )}
        </button>
      </div>

      {/* Confidence Summary */}
      {confidenceSummary && (
        <div className="grid grid-cols-3 gap-4">
          {/* Evidence Quality */}
          <div className="bg-white border-2 border-gray-200 rounded-lg p-4">
            <div className="flex items-center gap-2 mb-2">
              <FileText className="w-5 h-5 text-gray-600" />
              <h3 className="font-semibold text-sm">Evidence Quality</h3>
            </div>
            <div className="space-y-1 text-sm">
              <div className="flex justify-between">
                <span className="text-green-600">Tier 1:</span>
                <span className="font-semibold">{confidenceSummary.evidence_quality.tier1_sources}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-amber-600">Tier 2:</span>
                <span className="font-semibold">{confidenceSummary.evidence_quality.tier2_sources}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-red-600">Tier 3:</span>
                <span className="font-semibold">{confidenceSummary.evidence_quality.tier3_sources}</span>
              </div>
            </div>
            <div className={`mt-3 px-3 py-1 rounded text-center text-xs font-semibold ${
              confidenceSummary.evidence_quality.overall_confidence === 'high' ? 'bg-green-100 text-green-800' :
              confidenceSummary.evidence_quality.overall_confidence === 'medium' ? 'bg-amber-100 text-amber-800' :
              'bg-red-100 text-red-800'
            }`}>
              {confidenceSummary.evidence_quality.overall_confidence.toUpperCase()} CONFIDENCE
            </div>
          </div>

          {/* Policy Alignment */}
          <div className="bg-white border-2 border-gray-200 rounded-lg p-4">
            <div className="flex items-center gap-2 mb-2">
              <CheckCircle className="w-5 h-5 text-gray-600" />
              <h3 className="font-semibold text-sm">Policy Alignment</h3>
            </div>
            <div className="text-center">
              <div className="text-3xl font-bold text-blue-600">
                {Math.round(confidenceSummary.policy_alignment.average_score * 100)}%
              </div>
              <div className="text-xs text-gray-600 mt-1">
                {confidenceSummary.policy_alignment.level.toUpperCase()} alignment
              </div>
            </div>
          </div>

          {/* SDC Reporting */}
          <div className="bg-white border-2 border-gray-200 rounded-lg p-4">
            <div className="flex items-center gap-2 mb-2">
              <TrendingUp className="w-5 h-5 text-gray-600" />
              <h3 className="font-semibold text-sm">SDC Reporting</h3>
            </div>
            <div className="text-center">
              <div className="text-3xl font-bold text-purple-600">
                {Math.round(confidenceSummary.sdc_reporting.average_relevance * 100)}%
              </div>
              <div className={`text-xs mt-1 font-semibold ${
                confidenceSummary.sdc_reporting.suitable_for_reporting ? 'text-green-600' : 'text-red-600'
              }`}>
                {confidenceSummary.sdc_reporting.suitable_for_reporting ? 
                  '✓ Suitable for reporting' : 
                  '✗ Needs more SDC metrics'}
              </div>
            </div>
          </div>
        </div>
      )}

      {/* Answer Display */}
      {answer && (
        <div className="bg-white border-2 border-gray-200 rounded-lg p-6 space-y-4">
          <div className="prose max-w-none">
            <h3 className="text-lg font-bold mb-3">Planning Evidence:</h3>
            <div className="text-gray-800 whitespace-pre-wrap">{answer}</div>
          </div>

          {/* Sources with Ranking Scores */}
          {sources.length > 0 && (
            <div className="pt-4 border-t border-gray-200">
              <h4 className="font-semibold text-sm mb-3">
                Sources (Ranked by Quality, Policy Alignment & SDC Relevance)
              </h4>
              <div className="space-y-3">
                {sources.map((source, i) => (
                  <div key={i} className="border-2 border-gray-200 rounded-lg p-4">
                    <div className="flex items-start justify-between mb-2">
                      <div className="flex-1">
                        <div className="font-medium text-sm">{source.title}</div>
                        <div className="text-xs text-gray-600">
                          {source.source_org}
                        </div>
                      </div>
                      <div className={`px-2 py-1 rounded text-xs font-semibold ${
                        source.evidence_tier === 'tier1' ? 'bg-green-100 text-green-800' :
                        source.evidence_tier === 'tier2' ? 'bg-amber-100 text-amber-800' :
                        'bg-red-100 text-red-800'
                      }`}>
                        {source.evidence_tier.toUpperCase()}
                      </div>
                    </div>

                    {/* Ranking Scores */}
                    <div className="grid grid-cols-3 gap-2 mt-3">
                      <div className="bg-blue-50 p-2 rounded">
                        <div className="text-xs text-gray-600">Policy Alignment</div>
                        <div className="text-sm font-semibold text-blue-700">
                          {Math.round(source.policy_alignment * 100)}%
                        </div>
                        <div className="text-xs text-gray-500">{source.policy_label}</div>
                      </div>
                      <div className="bg-green-50 p-2 rounded">
                        <div className="text-xs text-gray-600">Data Quality</div>
                        <div className="text-sm font-semibold text-green-700">
                          {Math.round(source.data_confidence * 100)}%
                        </div>
                        <div className="text-xs text-gray-500">{source.quality_label}</div>
                      </div>
                      <div className="bg-purple-50 p-2 rounded">
                        <div className="text-xs text-gray-600">SDC Relevance</div>
                        <div className="text-sm font-semibold text-purple-700">
                          {Math.round(source.sdc_relevance * 100)}%
                        </div>
                      </div>
                    </div>

                    {/* Overall Score */}
                    <div className="mt-2 text-right">
                      <span className="text-xs text-gray-500">Final Score: </span>
                      <span className="text-sm font-bold text-gray-800">
                        {(source.final_score * 100).toFixed(0)}/100
                      </span>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* Warning for Tier 3 Data */}
          {sources.some(s => s.evidence_tier === 'tier3') && (
            <div className="flex items-start gap-2 p-4 bg-amber-50 border-l-4 border-amber-500 rounded">
              <AlertTriangle className="w-5 h-5 text-amber-600 mt-0.5" />
              <div className="flex-1">
                <div className="font-semibold text-sm text-amber-900">
                  Data Quality Warning
                </div>
                <div className="text-sm text-amber-800 mt-1">
                  Some sources are Tier 3 (low confidence). Recommend validating through 
                  stakeholder workshops before using for major planning decisions.
                </div>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

---

## Example Query Flow

### Query: "What does NTB data show about off-peak visitor potential in Madi?"

#### Step 1: Vector Retrieval
System retrieves 20 candidate chunks based on semantic similarity

#### Step 2: Multi-Criteria Ranking

**Chunk A**: NTB Statistical Bulletin Q1 2026
- Semantic Similarity: 0.85
- Policy Alignment: 0.7 (mentions visitor dispersal)
- Evidence Tier: tier1 → Confidence: 1.0
- SDC Relevance: 0.8 (has quantitative data, economic metrics)
- **Final Score: 0.84**

**Chunk B**: Single hotel owner estimate
- Semantic Similarity: 0.90 (higher than Chunk A!)
- Policy Alignment: 0.2 (no policy reference)
- Evidence Tier: tier3 → Confidence: 0.2
- SDC Relevance: 0.3 (anecdotal, no metrics)
- **Final Score: 0.40**

**Chunk C**: DIGO Workshop Output (15 participants)
- Semantic Similarity: 0.75
- Policy Alignment: 0.6 (references DIGO criteria)
- Evidence Tier: tier2 → Confidence: 0.6
- SDC Relevance: 0.7 (participatory data, stakeholder validated)
- **Final Score: 0.66**

**Result**: Chunk A ranks #1 despite lower semantic similarity!

#### Step 3: GPT-4 Answer Generation

Uses context from top 5 ranked chunks:

```
**Key Findings:**
• Tier 1 Data (NTB Q1 2026): Off-peak months (May-August) show 30% lower visitor 
  volumes in Madi compared to peak season, presenting opportunity for targeted promotion
• Tier 2 Data (DIGO Workshop): Local homestay operators report capacity to host 
  additional 200 guests/month during monsoon season
• Tier 3 Data (Single estimate): One hotel suggests 40% vacancy rate - NEEDS VALIDATION

**Policy Alignment:**
This aligns with National Tourism Strategy Section 4.7 "Seasonal Diversification" and 
DIGO Criterion 3 "Sustainable Capacity Utilization"

**Data Quality Notes:**
The hotel vacancy estimate is Tier 3 - recommend cross-validating with Hotel Association 
before using for planning

**Recommendation:**
1. Develop monsoon tourism packages (nature walks, cultural experiences)
2. Validate capacity estimates through multi-stakeholder workshop
3. Pilot program with women-led homestays (SDC-aligned)
```

---

## Document Upload with Auto-Tagging

When uploading documents, automatically calculate and store ranking metadata:

```typescript
async function processAndUploadDocument(file: File, metadata: any) {
  // Parse document
  const documentText = await parseDocument(file);
  
  // Chunk text
  const chunks = chunkText(documentText);
  
  // For each chunk, calculate ranking metadata
  const enrichedChunks = await Promise.all(
    chunks.map(async (chunkText, index) => {
      // Generate embedding
      const embedding = await generateEmbedding(chunkText);
      
      // Auto-calculate policy alignment score
      const policyScore = await assessPolicyAlignment(chunkText, metadata);
      
      // Auto-detect SDC relevance
      const sdcScore = calculateSDCRelevance({ chunk_text: chunkText }, '');
      
      // Determine evidence tier from metadata or default to tier3
      const evidenceTier = metadata.evidence_tier || 'tier3';
      
      // Check for quantitative data
      const hasQuantData = /\d+(?:\.\d+)?(?:\s*%|\s*tourists|\s*NPR)/.test(chunkText);
      
      return {
        chunk_text: chunkText,
        chunk_index: index,
        embedding,
        evidence_tier: evidenceTier,
        policy_alignment_score: policyScore,
        sdc_relevance_score: sdcScore,
        has_quantitative_data: hasQuantData,
        metadata: {
          ...metadata,
          auto_tagged: true,
          processed_at: new Date().toISOString()
        }
      };
    })
  );
  
  // Insert into database
  await supabase.from('rag_chunks').insert(enrichedChunks);
}
```

---

## Success Metrics

### Accuracy Metrics
- ✅ 90%+ of planners find answers useful
- ✅ 95%+ policy alignment assessments are correct
- ✅ Tier 1 sources prioritized in 85%+ of queries

### Efficiency Metrics
- ✅ Planning research time reduced from 3 hours → 30 minutes
- ✅ Query response time < 15 seconds
- ✅ SDC report preparation time cut by 75%

### Quality Metrics
- ✅ Average evidence tier in top 5 results: Tier 1.5 or better
- ✅ Policy alignment score > 0.6 for planning queries
- ✅ SDC relevance score > 0.5 for reporting queries

---

## Next Steps for Implementation

1. **Week 1**: Extend database schema with ranking metadata columns
2. **Week 2**: Implement multi-criteria scoring functions
3. **Week 3**: Build enhanced query API with re-ranking
4. **Week 4**: Create Planning Intelligence UI component
5. **Week 5**: Test with real DIGO facilitators and iterate

---

This Planning Intelligence Assistant gives DIGO a competitive advantage: not just finding information, but **ranking it by what matters for sustainable tourism planning** - policy alignment, data quality, and donor reporting value. 🎯
