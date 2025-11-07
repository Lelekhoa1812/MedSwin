# Chain-of-Thought (CoT) Sequential Answering System

## Overview

This document describes the sequential Chain-of-Thought (CoT) answering system designed for Small Language Models (SLMs) with limited token input/output capabilities. The system enables models to generate complete answers beyond their initial token limits by continuing generation until the answer is truly complete.

## Core Techniques

### 1. Sequential Generation Beyond Token Limits

**Purpose**: Allow models to generate beyond `max_new_tokens` to complete their answers.

**Mechanism**:
- Initial generation up to `max_new_tokens` (e.g., 1024 tokens)
- If response is incomplete, automatically continues with additional chunks (50% of `max_new_tokens` per continuation)
- Can generate up to 5x `max_new_tokens` total (e.g., 5120 tokens for 1024 limit)
- Maximum of 10 continuation attempts

**Benefits**: Ensures complete answers even for complex questions that require extensive responses.

---

### 2. Response Completeness Detection

**Purpose**: Determine if a response is complete or needs continuation.

**Detection Methods**:
- **Incomplete Indicators**: Detects responses ending with commas, semicolons, conjunctions (and, or, but, with, for, to, the)
- **Punctuation Check**: Verifies proper sentence endings (periods, exclamation marks, question marks)
- **Structure Indicators**: Detects incomplete lists, numbered sequences, or continuation words (next, then, finally, also, additionally)
- **Sentence Structure**: Validates that last sentence has proper structure (subject, verb, object)

**Decision Logic**: Response is marked incomplete if any indicator suggests it was cut off mid-thought.

---

### 3. Embedding-Based Semantic Similarity Checking

**Purpose**: Ensure continuation maintains topic relevance and coherence using semantic understanding rather than keyword matching.

**Implementation**:
- Uses the same embedding model as RAG (`MedEmbed-large-v0.1`)
- Computes cosine similarity between:
  - Continuation and original message
  - Continuation and original response tail
  - Continuation and structure elements
  - Continuation and previous continuations

**Thresholds**:
- **Low Relevance**: Similarity < 0.2 (with 150+ chars) → Likely off-topic
- **Medical Context**: Similarity < 0.25 (with 150+ chars) → Likely off-topic
- **High Repetition**: Similarity > 0.9 AND token overlap > 0.9 → Likely duplicate
- **Near-Duplicate**: Similarity > 0.95 AND token overlap > 0.85 → Near-duplicate
- **Sentence Duplication**: Similarity > 0.9 AND token overlap > 0.9 → Duplicate sentence

**Benefits**: More accurate than keyword matching, especially for medical terminology and context variations. Uses both semantic similarity and token overlap for more precise duplicate detection.

---

### 4. Structure Detection and Maintenance

**Purpose**: Maintain answer structure (numbering, headings, lists) throughout continuations.

**Structure Extraction**:
- Detects numbered lists (1., 2., 3., etc.)
- Identifies headings and bullet points
- Tracks numbering sequence to detect incomplete lists
- Maintains structure metadata (items, numbering, expected next number)

**Structure Maintenance**:
- Continuation prompt includes unmentioned structure elements
- Suggests next numbered item if list is incomplete
- Validates that continuation maintains structure coherence
- Uses semantic similarity to check structure relevance

**Benefits**: Ensures answers maintain logical flow and complete all planned sections.

---

### 5. Semantic Deduplication

**Purpose**: Remove duplicate content from continuations using semantic similarity.

**Deduplication Methods**:
- **Sentence-Level**: Compares each continuation sentence with previous sentences using semantic similarity AND token overlap (>0.9 for both = duplicate)
- **Phrase-Level**: Checks 4-word phrases for semantic similarity AND token overlap (>0.9 for both = duplicate)
- **Full-Text**: Overall similarity check (>0.9 semantic AND >0.9 token overlap = likely repetition)
- **Near-Duplicate**: Catches near-duplicates with >0.95 semantic AND >0.85 token overlap

**Process**:
- Extracts only new sentences from continuation
- Filters out semantically similar sentences
- Appends only unique content to response

**Benefits**: Prevents model from repeating information already provided, keeping responses concise and informative.

---

### 6. Hallucination Detection

**Purpose**: Detect and stop off-topic or hallucinated content in continuations.

**Detection Methods**:
- **Semantic Similarity**: Low similarity (<0.2) with both message and response (150+ chars) indicates off-topic
- **Medical Context**: Low similarity (<0.25) with medical context missing indicates off-topic
- **Keyword Analysis**: Checks for medical context keywords
- **Pattern Matching**: Detects obvious hallucination indicators (Wikipedia links, non-medical topics, meta-content)
- **Language Consistency**: Detects unexpected language switches

**Action**: Stops continuation immediately if hallucination detected and reverts to previous valid response.

**Benefits**: Prevents model from generating irrelevant or incorrect information.

---

### 7. Context Relevance Validation

**Purpose**: Ensure continuation stays relevant to original question and context.

**Validation Checks**:
- **Message Similarity**: Continuation should be semantically similar to original question
- **Response Similarity**: Continuation should continue from where response left off
- **Medical Context**: Continuation should maintain medical/clinical context
- **Topic Coherence**: Continuation should share semantic space with original response

**Multi-Layer Validation**:
- Real-time checks during streaming (every 30 chunks, minimum 150 chars)
- Final validation after continuation completes
- Semantic similarity as primary check, keyword matching as fallback

**Benefits**: Ensures continuations are relevant and maintain answer quality.

---

### 8. Internal Reasoning Token Filtering

**Purpose**: Remove model's internal reasoning tokens that shouldn't appear in final output.

**Filtered Tokens**:
- Thought process markers (`<end_of_instructions>`, `<end_of_turn>`, `thought process:`, `thinking:`, `reasoning:`)
- Meta-commentary (`internal:`, `meta:`, `[thinking]`, `[reasoning]`)
- Completion phrases (`okay, i need to finish`, `i need to finish`)
- Prompt-related tokens (`the prompt asks you`, `the user has stopped`, `provide steps involved`, `so provide steps`, `steps involved in developing`)

**Process**:
- Filters tokens from continuation before appending
- Removes lines starting with reasoning patterns
- Cleans output to show only actual answer content

**Benefits**: Keeps responses clean and professional, hiding model's internal processing.

---

### 9. Early EOS Prevention

**Purpose**: Prevent model from stopping prematurely before completing its answer.

**Mechanisms**:
- **Logits Processor**: Sets EOS token probability to `-inf` until 95% of `max_new_tokens` are generated
- **Min New Tokens**: Ensures at least 95% of `max_new_tokens` before EOS can stop
- **Two-Stage Prevention**: 
  - Stage 1: Blocks EOS until 95% threshold
  - Stage 2: Blocks EOS until <5% tokens remain

**Benefits**: Forces model to use almost all available tokens, preventing incomplete responses.

---

### 10. Structure Coherence Checking

**Purpose**: Validate that continuation maintains the answer's structural plan.

**Coherence Checks**:
- **Structure Mentions**: Continuation should mention structure elements from original response
- **Numbering Continuation**: If original had numbered list, continuation should continue numbering
- **Semantic Structure Similarity**: Continuation should be semantically similar to structure elements
- **Sequence Validation**: Checks if continuation follows expected sequence

**Validation Logic**:
- If no structure mentions AND low similarity (<0.2) → Likely off-plan
- If numbering expected but not found AND low similarity (<0.25) → Likely off-plan
- Otherwise → Accept continuation

**Benefits**: Ensures continuations complete the answer plan rather than going off on tangents.

---

## Data Flow

1. **Initial Generation**: Model generates up to `max_new_tokens`
2. **Completeness Check**: System checks if response is complete
3. **If Incomplete**:
   - Show loader indicator (`*[Generating continuation answer...]*`) to user
   - Log current response state (length, ending text, structure)
   - Extract answer structure and plan
   - Track covered topics to guide continuation
   - Build methodical continuation prompt with context, structure hints, and covered topics
   - Generate continuation chunk (50% of `max_new_tokens`)
   - Real-time hallucination detection during streaming (every 30 chunks, 150+ chars)
   - Filter internal reasoning tokens (including `<end_of_turn>`, prompt-related tokens)
   - Deduplicate using semantic similarity AND token overlap
   - Validate context relevance and structure coherence
   - Log continuation summary (count, length, start, end)
   - Append only new, relevant content
4. **Repeat**: Continue until response is complete or limits reached
5. **Final Cleanup**: Remove special tokens, clean formatting

---

## Key Features

- **Adaptive Token Usage**: Can generate 5x initial limit if needed
- **Semantic Understanding**: Uses embeddings for accurate relevance checking
- **Structure Preservation**: Maintains numbering, headings, and logical flow
- **Duplicate Prevention**: Removes redundant content using semantic similarity AND token overlap
- **Hallucination Prevention**: Stops off-topic content before it contaminates response
- **Context Maintenance**: Ensures continuations stay relevant to original question
- **Clean Output**: Filters internal reasoning and special tokens
- **User Feedback**: Shows loader indicator during continuation generation
- **Methodical Tracking**: Logs response state, structure, and continuation summaries for debugging

---

## Benefits for SLMs

Small Language Models often struggle with:
- Limited context windows
- Premature stopping
- Incomplete answers
- Hallucination
- Repetition

This CoT system addresses all these issues by:
- Allowing generation beyond initial limits
- Preventing premature stopping
- Ensuring completeness
- Detecting and stopping hallucinations
- Removing duplicates
- Maintaining context and structure

---

## Configuration

- **Max Total Tokens**: 5x `max_new_tokens` (e.g., 5120 for 1024 limit)
- **Continuation Chunk Size**: 50% of `max_new_tokens` (e.g., 512 for 1024 limit)
- **Max Continuations**: 10 attempts
- **Similarity Thresholds**:
  - Low relevance: <0.2 (150+ chars required)
  - Medical context: <0.25 (150+ chars required)
  - High repetition: >0.9 semantic AND >0.9 token overlap
  - Near-duplicate: >0.95 semantic AND >0.85 token overlap
  - Duplicate sentence: >0.9 semantic AND >0.9 token overlap
- **EOS Prevention**: 95% of `max_new_tokens` before allowing EOS
- **Check Frequency**: Every 30 chunks (minimum 150 chars) for hallucination detection

