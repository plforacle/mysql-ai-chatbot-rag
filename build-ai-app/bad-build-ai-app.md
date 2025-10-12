# Build MySQL AI RAG Application
## Introduction

This implementation showcases a pure RAG (Retrieval-Augmented Generation) application built on the LAMP stack, powered by MySQL Enterprise Edition 9.4.1's native AI capabilities. The application delivers intelligent movie recommendations from the Sakila sample database through semantic vector search and context-aware LLM responses, all processed within the database layer.

**Note:** This code is for educational purposes only and is not designed for production use.

_Estimated Lab Time:_ 30 minutes

### Objectives

- Understand the application architecture
- Deploy and configure the application
- Check for Errors
- Run and test the RAG application

### Prerequisites

- Oracle Trial or Paid Cloud Account
- Apache Web server with PHP 8.2+
- MySQL Enterprise Edition 9.4.1 or higher with MySQL AI enabled
- MySQL AI models configured:
  - `all_minilm_l12_v2` for embeddings
  - `llama3.2-3b-instruct-v1` for text generation
- Basic familiarity with MySQL and PHP
- Completed Lab 6


## Overview

A complete RAG chatbot demonstrating MySQL AI's in-database semantic search and LLM inference. All AI operations (embeddings, search, LLM) run inside MySQL - zero external API calls.

**Notable Design Decisions**

- JSON contexts for LLM (more efficient than plain text)
- Similarity threshold filtering (0.30) to prevent low-quality matches
- Conversation history windowing (max 4 turns)
- HEX/UNHEX encoding for PDO compatibility with VECTOR type
- Semantic enrichment in embedding text (e.g., "family-friendly" markers)

**File Structure**
```
chatbot-mysql-ai-rag/
├── api_key.php              # Database connection
├── config.php               # RAG parameters & model IDs
├── chat_handler.php         # Main RAG logic & API endpoint
├── generate_embeddings.php  # One-time vector generation
├── index.html               # Frontend UI
└── styles.css               # Styling
```

**RAG Workflow**

```
User Query
  ↓
Generate embedding (ML_EMBED_ROW)
  ↓
Similarity search (DISTANCE)
  ↓
Build JSON contexts
  ↓
Create super prompt
  ↓
LLM generation (ML_GENERATE)
  ↓
Response
```



---

## File Summaries

### `api_key.php` - Database Connection
Establishes PDO connection to MySQL with Sakila database.
**Production**: Use environment variables, not hardcoded credentials.
---

### `config.php` - Settings

**RAG Parameters**:
```php
define('RAG_K', 4);                           // Movies to retrieve
define('RAG_SIMILARITY_THRESHOLD', 0.30);     // Min similarity score
define('RAG_MAX_CONVERSATION_HISTORY', 4);    // Max turns to keep
```

**MySQL AI Models**:
```php
define('MODEL_ID_EMBED', 'all_minilm_l12_v2');
define('MODEL_ID_LLM', 'llama3.2-3b-instruct-v1');
define('MODEL_LLM_TEMPERATURE', 0.25);
```

---

### `chat_handler.php` - RAG Engine

Main application with complete RAG workflow.

**API Endpoints**:
- `POST {"message": "query"}` - Get AI response
- `POST {"action": "clear_history"}` - Clear session

**Core Functions**:

#### `getRAGResponse($user_query)`
Orchestrates RAG: retrieve movies → build contexts → generate response → update history.

#### `semanticMovieRetrieval($pdo, $query, $rag_k)`
Two-step semantic search:
1. Generate query embedding with `ML_EMBED_ROW()`
2. Find similar movies with `DISTANCE()` function

Returns top-K movies sorted by similarity.

#### `buildMovieContext($films)`
Formats movies as JSON for LLM:
```json
[{
  "orderOfSignificance": 1,
  "title": "ACADEMY DINOSAUR",
  "similarityScore": 0.85,
  ...
}]
```

#### `buildConversationContext($history)`
Formats chat history as JSON for LLM continuity.

#### `buildSuperPrompt($userQuery, $movieContext, $conversationContext)`
Constructs 7-section prompt:
1. AI Identity (personality)
2. Key Rules (knowledge boundaries)
3. Style Guide (response format)
4. Movie Context (retrieved films)
5. Conversation History
6. Response Instructions
7. User Request

#### `generateAIResponse($pdo, $superPrompt)`
Calls `ML_GENERATE()` for in-database LLM inference.

---

### `generate_embeddings.php` - Vector Setup

**⚠️ Must run once before using app!**

Generates vector embeddings for all films using `ML_EMBED_ROW()`.

**Process**:
1. Create enhanced text for each film
2. Store text in `embedding_text` column
3. Generate embedding (in-database)
4. Store in `vector_embedding` VECTOR column

**Run**:
```bash
php generate_embeddings.php
```

Expected time: ~2-8 minutes for 1000 films.

#### `create_film_text($film)`
Creates comprehensive text representation with semantic enrichment:
- Core metadata (title, description, actors)
- Family-friendly markers for G/PG ratings
- Genre descriptions for better matching

---

### `index.html` - Frontend

Terminal-style chat interface with:
- Menu bar (New Chat, Sample Prompts, About)
- Scrollable chat container
- Multi-line input with Send button

**Key JavaScript Functions**:
- `sendMessage()` - POST to chat_handler.php
- `addAIMessage()` - Render with markdown/syntax highlighting
- `addCopyButtonsToCodeBlocks()` - Copy functionality
- `clearConversationHistory()` - Clear session

**Libraries**:
- Showdown.js (markdown parsing)
- Prism.js (syntax highlighting)

---

### `styles.css` - Styling

Terminal-inspired dark theme with:
- Green AI text (#00ff00)
- Yellow user text (#ffff00)
- Orange "MySQL AI RAG" branding (#ff6b35)
- Monospace fonts
- Responsive design

---

## MySQL AI Functions

### `sys.ML_EMBED_ROW(text, options)`
Generate vector embeddings in-database.
```sql
SELECT HEX(sys.ML_EMBED_ROW(
  'text', 
  JSON_OBJECT('model_id', 'all_minilm_l12_v2')
));
```

### `sys.ML_GENERATE(prompt, options)`
Generate text with in-database LLM.
```sql
SELECT sys.ML_GENERATE(
  'prompt',
  JSON_OBJECT('task', 'generation', 'model_id', 'llama3.2-3b-instruct-v1')
);
```

### `DISTANCE(vector1, vector2, metric)`
Calculate vector similarity.
```sql
SELECT 1 - DISTANCE(v1, v2, 'COSINE') AS similarity;
```

---

## Quick Setup

1. **Extract files** to `/var/www/html/chatbot-mysql-ai-rag/`
2. **Configure** `api_key.php` with your credentials
3. **Set permissions**: `sudo chmod 644 *.php *.html *.css`
4. **Generate embeddings**: `php generate_embeddings.php` ⚠️ **Required!**
5. **Open browser**: `http://your-server-ip/chatbot-mysql-ai-rag/`

---

## Database Schema

```sql
CREATE TABLE film_list_rag (
  film_id INT PRIMARY KEY,
  title VARCHAR(255),
  description TEXT,
  category VARCHAR(100),
  embedding_text TEXT,           -- Text used for embedding
  vector_embedding VECTOR(384)   -- 384-dim vector
);
```

---

## Troubleshooting

| Issue | Check | Fix |
|-------|-------|-----|
| Blank page | PHP errors | `sudo tail -f /var/log/php-fpm/www-error.log` |
| No DB connection | MySQL running | `sudo systemctl status mysql` |
| No results | Embeddings exist | Run `generate_embeddings.php` |
| ML_GENERATE fails | Models loaded | `SELECT model_id FROM mysql_model_catalog;` |

---

## RAG Workflow

```
User Query
  ↓
Generate embedding (ML_EMBED_ROW)
  ↓
Similarity search (DISTANCE)
  ↓
Build JSON contexts
  ↓
Create super prompt
  ↓
LLM generation (ML_GENERATE)
  ↓
Response
```

---

## Configuration Tuning

**Faster queries**: Lower `RAG_K` to 3
**Better results**: Raise `RAG_K` to 8
**Stricter matching**: Raise `RAG_SIMILARITY_THRESHOLD` to 0.40
**More creative**: Raise `MODEL_LLM_TEMPERATURE` to 0.5

---

## Security Notes

**Current**: Development/educational use only
**Production needs**:
- Environment variables for credentials
- Input validation and sanitization
- Rate limiting
- HTTPS/SSL
- Security headers
- Read-only database user

---

## Key Advantages

✅ **In-Database AI** - No external API calls  
✅ **Data Privacy** - Everything stays in MySQL  
✅ **Simple SQL** - Standard SQL with ML functions  
✅ **Fast** - No network latency to external services  
✅ **Cost-Effective** - No per-query API fees  
✅ **Complete RAG** - Retrieval + generation in one system

---

## Learn More

- [MySQL AI Documentation](https://dev.mysql.com/doc/mysql-ai/9.4/en/)
- [Oracle LiveLabs Tutorial](https://oracle-livelabs.github.io/)

**Authors**: Craig Shallahamer, Perside Foster  
**Last Updated**: October 2025