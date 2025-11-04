# Medical RAG API Documentation

This document describes how to use the Medical RAG Assistant via the Gradio API.

## Installation

```bash
pip install gradio_client
```

## Setup

```python
from gradio_client import Client, handle_file
client = Client("BinKhoaLe1812/MedSwin-Distilled-7B") 
```

Or another client:
```python
client = Client("MedAI-COS30018/MedSwin-Distilled-7B")
```

## API Endpoints

### 1. Upload and Index Documents

**Endpoint:** `/create_or_update_index`

Upload PDF or text files to build a searchable index for retrieval-augmented generation.

```python
result = client.predict(
    files=[handle_file('path/to/document.pdf')],
    api_name="/create_or_update_index"
)
print(result)
```

**Parameters:**
- `files` (list[filepath], Required): List of file paths to upload (PDF or TXT)

**Returns:**
- `tuple[str, str]`: Status message and file information HTML

**Example:**
```python
result = client.predict(
    files=[
        handle_file('clinical_guidelines.pdf'),
        handle_file('patient_notes.txt')
    ],
    api_name="/create_or_update_index"
)
status, file_info = result
print(f"Status: {status}")
```

### 2. Chat with Medical Model

**Endpoint:** `/stream_chat`

Query the medical model with optional document retrieval. Returns a complete conversation history.

```python
result = client.predict(
    message="What are the treatment options for melanoma?",
    history=[],
    system_prompt="Answer clinically and concisely using the provided context.",
    disable_retrieval=False,
    temperature=0.1,
    max_new_tokens=512,
    top_p=0.5,
    top_k=20,
    penalty=1.3,
    retriever_k=15,
    merge_threshold=0.5,
    api_name="/stream_chat"
)
print(result)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | str | Required | User's question |
| `history` | list[dict] | `[]` | Previous conversation messages |
| `system_prompt` | str | `"Answer clinically..."` | System instruction prompt |
| `disable_retrieval` | bool | `False` | Disable document retrieval (use model knowledge only) |
| `temperature` | float | `0.1` | Sampling temperature (0.0-1.0) |
| `max_new_tokens` | float | `512` | Maximum tokens to generate |
| `top_p` | float | `0.5` | Nucleus sampling threshold |
| `top_k` | float | `20` | Top-k sampling |
| `penalty` | float | `1.3` | Repetition penalty |
| `retriever_k` | float | `15` | Number of documents to retrieve |
| `merge_threshold` | float | `0.5` | Node merging threshold (lower = more merging) |

**Returns:**
- `list[dict]`: Conversation history with `role` and `content` keys

**Example:**
```python
# First query
result1 = client.predict(
    message="What are the symptoms of diabetes?",
    history=[],
    api_name="/stream_chat"
)

# Continue conversation
history = result1
result2 = client.predict(
    message="What are the treatment options?",
    history=history,
    api_name="/stream_chat"
)
```

## Complete Workflow Example

```python
from gradio_client import Client, handle_file

# Initialize client
client = Client("BinKhoaLe1812/MedSwin-Distilled-7B")

# Step 1: Upload documents
print("Uploading documents...")
status, info = client.predict(
    files=[handle_file('medical_guidelines.pdf')],
    api_name="/create_or_update_index"
)
print(status)

# Step 2: Query with retrieval
print("\nQuerying model...")
response = client.predict(
    message="What are the recommended treatments?",
    history=[],
    disable_retrieval=False,
    temperature=0.1,
    max_new_tokens=512,
    api_name="/stream_chat"
)

# Extract assistant's last message
for msg in response:
    if msg.get("role") == "assistant":
        print(f"Assistant: {msg.get('content')}")
```

## Notes

- **File Upload:** Supported formats are PDF and TXT
- **Retrieval:** Set `disable_retrieval=False` to use document context (requires prior upload)
- **Temperature:** Lower values (0.0-0.2) produce more deterministic, clinical responses
- **History:** Maintain conversation context by passing previous responses in `history`