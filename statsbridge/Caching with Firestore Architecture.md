# Layered Architecture Pattern - Complete Guide

## Overview

This document describes a **Layered Architecture Pattern** (also known as N-Tier or Clean Architecture) for building scalable web applications. This pattern organizes code into distinct layers, each with specific responsibilities, promoting separation of concerns and maintainability.

---

## Architecture Layers

### 1. **Presentation Layer (API/Routes)**

**Location:** `/api/`

**Responsibility:**
- Handle HTTP requests and responses
- Route incoming requests to appropriate handlers
- Parse request parameters and validate input
- Return formatted responses (JSON, etc.)
- Handle request/response serialization

**Key Components:**
- Router files that define API endpoints
- Request/response validation
- HTTP status codes and error handling
- No business logic should live here

**Example Endpoints:**
```
POST /api/search
POST /api/stored-searches
GET /api/articles
POST /api/tracking
```

**Why Separate?**
- Decouples API contracts from business logic
- Allows changing HTTP frameworks without touching business logic
- Easy to add new endpoints or modify existing ones

---

### 2. **Models Layer (Data Models)**

**Location:** `/models/`

**Responsibility:**
- Define data structures using validation libraries (Pydantic for Python)
- Represent request/response schemas
- Validate incoming data against schemas
- Type hints for better IDE support and code clarity

**Key Features:**
- **Request Models:** Define what data API endpoints accept
- **Response Models:** Define what data endpoints return
- **Data Validation:** Automatic validation and type conversion
- **Documentation:** Models serve as API documentation

**Example Structure:**
```python
class SearchRequest(BaseModel):
    searchterm: str
    params: Optional[Dict[str, Any]]

class SearchResponse(BaseModel):
    results: List[Article]
    total_count: int
    metadata: Dict[str, Any]
```

**Why Separate?**
- Single source of truth for data structure
- Automatic validation without manual checks
- Self-documenting code
- Easy to change schema across the application

---

### 3. **Services Layer (Business Logic)**

**Location:** `/services/`

**Responsibility:**
- Implement core business logic
- Orchestrate operations across multiple systems
- Handle data transformations
- Manage caching and optimization logic
- Interact with external APIs and databases

**Key Components:**
- **Primary Services:** Core business logic
- **Client Services:** External API integrations
- **Cache Services:** Caching layer
- **Utility Services:** Shared operations

**Example Services:**
```
SearchService        → Search operations with caching
CacheService         → Cache management (read/write)
OpointClient         → External API calls
StoredSearchService  → Manage saved searches
```

**Typical Flow:**
1. Receive request from API layer
2. Check if data exists in cache
3. If not, fetch from external source/database
4. Process and transform data
5. Store in cache
6. Return to API layer

**Why Separate?**
- Reusable business logic across different endpoints
- Easy to test business logic independently
- Changes to business logic don't affect API contracts
- Supports multiple clients (API, CLI, webhooks)

---

### 4. **Data Access Layer (Database/External APIs)**

**Location:** Implicit in services (integrated with cache and client services)

**Responsibility:**
- Direct communication with databases
- Call external APIs
- Handle data persistence
- Manage connections

**Components:**
- **Cache Service:** Firestore, Redis, etc.
- **API Clients:** External service integrations
- **Database Drivers:** Direct database access

**Why Separate?**
- Isolates data source complexity
- Easy to switch data sources
- Enables caching strategies
- Supports multiple data sources simultaneously

---

### 5. **Configuration Layer**

**Location:** `/config/`

**Responsibility:**
- Load environment variables
- Provide application settings
- Manage secrets and credentials
- Database/API connection setup

**Key Components:**
```
settings.py      → Configuration management
firestore.py     → Database initialization
```

**Why Separate?**
- Centralized configuration management
- Easy environment switching (dev/test/prod)
- Secrets management
- No hardcoded values in code

---

### 6. **Utilities Layer**

**Location:** `/utils/`

**Responsibility:**
- Helper functions used across layers
- Hashing, encryption, validation
- String manipulation, formatting
- Query building, transformation

**Examples:**
```
hashing.py                → Hash functions
search_query_builder.py    → Query construction
```

**Why Separate?**
- Reusable utilities across services
- Reduces code duplication
- Centralizes common operations
- Easier maintenance

---

## Data Flow Architecture

### Typical Request Lifecycle

```
1. HTTP Request
        ↓
2. API Layer (Router)
   - Route request to handler
   - Parse input, validate against model
        ↓
3. Service Layer
   - Check cache service
   - If cache hit → return cached data
   - If cache miss → call external API/database
   - Transform data
   - Store in cache
        ↓
4. Data Access Layer
   - Cache operations (Firestore)
   - External API calls
   - Database queries
        ↓
5. Service Layer (continued)
   - Process response
   - Prepare response data
        ↓
6. API Layer (Router)
   - Format response (serialize to JSON)
   - Set HTTP status
        ↓
7. HTTP Response
```

---

## Dependency Flow

```
Presentation Layer (API)
        ↓
Models Layer (validates)
        ↓
Services Layer (orchestrates)
        ↓
Configuration + Utils
        ↓
External Systems (Databases, APIs)
```

**Key Principle:** Lower layers don't depend on upper layers. High-level modules depend on abstractions, not concrete implementations.

---

## File Organization

```
app/
├── main.py                 # Application entry point, FastAPI initialization
│
├── api/                    # Presentation Layer
│   ├── __init__.py
│   ├── router.py          # Main router that includes all sub-routers
│   ├── search.py          # Search endpoint handlers
│   ├── articles.py        # Article endpoint handlers
│   ├── tracking.py        # Tracking endpoint handlers
│   └── storedsearch.py    # Stored search endpoint handlers
│
├── models/                 # Models Layer
│   ├── __init__.py
│   ├── article.py         # Article models
│   ├── search.py          # Search request/response models
│   ├── tracking.py        # Tracking models
│   └── storedsearch.py    # Stored search models
│
├── services/               # Services Layer (Business Logic)
│   ├── __init__.py
│   ├── search_service.py       # Search business logic
│   ├── cache_service.py        # Cache management
│   ├── opoint_client.py        # External API client
│   ├── storedsearch_service.py # Stored search logic
│   └── tracking_service.py     # Tracking logic
│
├── config/                 # Configuration Layer
│   ├── __init__.py
│   ├── settings.py        # Environment configuration
│   └── firestore.py       # Database initialization
│
└── utils/                  # Utilities Layer
    ├── __init__.py
    ├── hashing.py         # Hash functions
    └── search_query_builder.py  # Query helpers
```

---

## Key Architectural Principles

### 1. **Separation of Concerns**
Each layer has a single responsibility:
- API layer handles HTTP
- Service layer handles business logic
- Data layer handles persistence

### 2. **DRY (Don't Repeat Yourself)**
- Services contain reusable logic
- Utilities provide shared functions
- Models define schema once

### 3. **Testability**
- Each layer can be tested independently
- Services can be tested with mock data
- No tight coupling between layers

### 4. **Maintainability**
- Changes in one layer don't affect others
- Easy to locate specific functionality
- Clear code organization

### 5. **Scalability**
- Easy to add new features (new endpoints, services)
- Services can be extracted into separate microservices
- Caching reduces load on external systems

---

## Communication Between Layers

### API → Service
```python
# In API layer
from app.services.search_service import search_service

@router.post("/api/search")
async def search(request: SearchRequest):
    result = await search_service.search(
        searchterm=request.searchterm,
        params=request.params
    )
    return SearchResponse(**result)
```

### Service → Cache Service
```python
# In service layer
from app.services.cache_service import cache_service

cached = await cache_service.get_cache(collection, key)
await cache_service.set_cache(collection, key, data)
```

### Service → External API
```python
# In service layer
from app.services.opoint_client import opoint_client

response = await opoint_client.search(searchterm, params)
```

---

## Configuration Management

Centralized settings allow environment-specific configuration:

```python
# In config/settings.py
class Settings(BaseSettings):
    API_KEY: str          # From .env or environment
    BASE_URL: str
    DEBUG: bool
    DATABASE_URL: str
```

**Benefits:**
- No hardcoded credentials in code
- Easy switching between environments
- Secrets management
- Clear dependency injection

---

## Caching Strategy with Firestore

### Overview
The application implements a **two-tier caching strategy**:
1. **Request-level caching** - Cache entire API responses
2. **Firestore persistence** - Store in database for cross-session reuse

### How Data Flows Through the Cache

#### Step 1: Request Comes In
```
User Request (POST /api/search)
├─ searchterm: "economic"
└─ params: {...}
```

#### Step 2: Generate Cache Key
```python
# Hash the entire request payload into a unique key
request_data = {
    "searchterm": "economic",
    "params": {
        "requestedarticles": 10,
        "main": {"header": 1, "summary": 1, "text": 1}
    }
}

# SHA-256 hash generates unique key
cache_key = "773aece3c0cab8c3868f3c09bebf9e389e42a53b943e2b7a2f57d7434ea8b708"
```

**Why Hashing?**
- Same request parameters → Same hash → Same cache entry
- Different parameters → Different hash → Different cache entry
- No need to store original request (privacy)
- Fast lookup using deterministic key

#### Step 3: Check Firestore Cache
```
Query Firestore Collection: "search_results"
Query Document: {cache_key}
        ↓
Document exists?
    ├─ YES → CACHE HIT (Return stored data)
    └─ NO  → CACHE MISS (Fetch from external API)
```

#### Step 4a: Cache HIT Flow
```
User Request
    ↓
Generate cache key: "24cee259e9b7073914340ba23170c61def1854710ef84f5484363bdaf178e19f"
    ↓
Query Firestore: search_results/24cee259...
    ↓
Document found!
    ├─ Return cached data instantly
    └─ No external API call needed ✓ (Faster response)
```

**Log Example (Cache HIT):**
```
2025-12-23 08:48:37,992 - Checking cache for: SPAN/5(labour, economic) AND lang:en
2025-12-23 08:48:38,009 - CACHE HIT - search_results/24cee259e9b7073914340ba23170c61def1854710ef84f5484363bdaf178e19f
2025-12-23 08:48:38,010 - ✓ CACHE HIT - returning cached result
```

#### Step 4b: Cache MISS Flow
```
User Request
    ↓
Generate cache key: "773aece3c0cab8c3868f3c09bebf9e389e42a53b943e2b7a2f57d7434ea8b708"
    ↓
Query Firestore: search_results/773aece3...
    ↓
Document NOT found!
    ├─ Call external API (e.g., Opoint)
    ├─ Get response data
    ├─ Store in Firestore
    └─ Return to client
```

**Log Example (Cache MISS → Store):**
```
2025-12-23 08:46:13,656 - Checking cache for: economic
2025-12-23 08:46:13,733 - ✗ CACHE MISS → Fetching from Opoint API for: economic
2025-12-23 08:46:15,550 - HTTP Request: POST https://api.opoint.com/search/ "HTTP/1.1 200 OK"
2025-12-23 08:46:15,556 - Opoint Search API successful - Got 10 documents
2025-12-23 08:46:15,720 - ✓ CACHE SET - search_results/773aece3c0cab8c3868f3c09bebf9e389e42a53b943e2b7a2f57d7434ea8b708
```

### Firestore Data Structure

#### Collection Structure
```
Firestore Database
└── search_results/                          # Collection name
    ├── 773aece3c0cab8c3868f3c09bebf9e389...  # Document (cache_key)
    │   ├── data: {                          # Actual API response
    │   │   ├── documents: [...]
    │   │   ├── host: "..."
    │   │   ├── cputime: 2.5
    │   │   └── ...
    │   ├── cached_at: "2025-12-23T08:46:15.720Z"
    │   └── expires_at: "2025-12-30T08:46:15.720Z"  # Optional TTL
    │
    ├── 24cee259e9b7073914340ba23170c61def...  # Another cached result
    │   ├── data: {...}
    │   ├── cached_at: "2025-12-23T08:47:57.733Z"
    │   └── expires_at: "2025-12-30T08:47:57.733Z"
    │
    └── ... more cached results
```

#### What Gets Stored
```python
{
    "data": {
        # Complete response from external API
        "documents": [
            {"id": "doc1", "title": "...", "content": "..."},
            {"id": "doc2", "title": "...", "content": "..."},
            # ... more documents
        ],
        "host": "api.example.com",
        "cputime": 2.5
    },
    "cached_at": "2025-12-23T08:46:15.720Z",      # When was it cached?
    "expires_at": "2025-12-30T08:46:15.720Z"      # When does it expire?
}
```

### Complete Request Lifecycle with Real Example

**Scenario 1: First request for "economic"**

```
TIME: 08:46:13 - User makes request
┌────────────────────────────────────────┐
│ POST /api/search                       │
│ {                                      │
│   "searchterm": "economic",            │
│   "params": {"requestedarticles": 10}  │
│ }                                      │
└────────────────────────────────────────┘
        ↓
API Layer receives request
        ↓
Service: SearchService.search()
        ↓
Generate cache key
cache_key = hash({"searchterm": "economic", "params": {...}})
        ↓
Check: cache_service.get_cache("search_results", cache_key)
        ↓
Firestore Query: 
  collection("search_results").document(cache_key).get()
        ↓
❌ Document doesn't exist (CACHE MISS)
        ↓
Call external API: 
  opoint_client.search("economic", params)
        ↓
TIME: 08:46:15 - External API responds with 10 documents
        ↓
Store in Firestore:
  db.collection("search_results")
    .document(cache_key)
    .set({
      "data": {response from API},
      "cached_at": "2025-12-23T08:46:15.720Z",
      "expires_at": "2025-12-30T08:46:15.720Z"
    })
        ↓
Return response to user (with 10 documents)
        ↓
TIME: 08:46:16 - Response sent (~3 seconds total)
```

**Scenario 2: Same request made again later**

```
TIME: 08:48:37 - User makes same request
┌──────────────────────────────────────────────────┐
│ POST /api/search                                 │
│ {                                                │
│   "searchterm": "SPAN/5(labour, economic) AND   │
│                  lang:en",                       │
│   "params": {"requestedarticles": 5}             │
│ }                                                │
└──────────────────────────────────────────────────┘
        ↓
API Layer receives request
        ↓
Service: SearchService.search()
        ↓
Generate same cache key (same input = same hash)
cache_key = hash({...}) = "24cee259e9b7073914340ba23170c61def..."
        ↓
Check: cache_service.get_cache("search_results", cache_key)
        ↓
Firestore Query:
  collection("search_results").document(cache_key).get()
        ↓
✅ Document exists! (CACHE HIT)
        ↓
Check TTL:
  if cached_at < expires_at: use cache
        ↓
Return cached data to user (with 5 documents)
        ↓
TIME: 08:48:38 - Response sent (~0.1 seconds total - 30x faster!)
        ↓
🎯 No external API call made, significant performance gain!
```

### Key Caching Concepts

#### 1. Cache Key Generation
```python
# Generate unique key from request payload
cache_key = SHA256(
    JSON(searchterm) + 
    JSON(params)
)

# Example:
# Request A: {"searchterm": "economic", "params": {...}}
# → cache_key = "773aece3c0cab8c3868f3c09bebf9e389e42a53b943e2b7a2f57d7434ea8b708"
#
# Request B: {"searchterm": "technology", "params": {...}}
# → cache_key = "different_hash_completely"
#
# Request A (again): {"searchterm": "economic", "params": {...}}
# → cache_key = "773aece3c0cab8c3868f3c09bebf9e389e42a53b943e2b7a2f57d7434ea8b708" (SAME!)
```

#### 2. Cache Hit vs Cache Miss

| Scenario | Flow | Time | API Calls |
|----------|------|------|-----------|
| **Cache MISS** | API → Firestore → Return | ~2-3 sec | ✓ YES (slow) |
| **Cache HIT** | Firestore → Return | ~0.1 sec | ✗ NO (fast) |

#### 3. TTL (Time to Live)
```python
# Cache expires after 7 days
expires_at = cached_at + 7 days

# When checking cache:
if now() < expires_at:
    return cached_data  # Still fresh
else:
    delete_from_cache()  # Expired, fetch fresh data
```

#### 4. Collections Organization
```
Different collections for different features:
┌─────────────────────────────┐
│    Firestore Database       │
├─────────────────────────────┤
│ search_results/             │ ← Cache for search queries
├─────────────────────────────┤
│ stored_searches/            │ ← Cache for saved searches
├─────────────────────────────┤
│ tracking_data/              │ ← Cache for tracking info
├─────────────────────────────┤
│ articles/                   │ ← Cache for article data
└─────────────────────────────┘
```

### Why This Caching Approach?

✅ **Performance**
- Cache hits return data 30x faster
- Reduces external API calls
- Lower latency for repeated requests

✅ **Cost Reduction**
- Fewer external API calls = lower bills
- Firestore reads are cheaper than external API calls
- Cache hits = zero external service costs

✅ **Reliability**
- If external API is down, serve cached data
- Improves application resilience

✅ **Scalability**
- Handle more users without proportional API call increase
- Cache reduces load on external services

✅ **User Experience**
- Instant responses for frequently requested data
- Reduced latency perceived by users

### Cache Invalidation Strategy

```python
# Manual invalidation when data changes
async def invalidate_cache(cache_key: str):
    await cache_service.delete_cache(collection, cache_key)

# Automatic invalidation when TTL expires
# Scheduled job removes expired entries
async def cleanup_expired_cache():
    # Find all documents where expires_at < now()
    # Delete them
    pass
```

**Why This Layer Separation Matters:**
- Cache logic centralized in CacheService
- API layer doesn't know about caching
- Services orchestrate cache + API calls
- Easy to change cache backend (Firestore → Redis → Memcache)
- Each layer maintains single responsibility

---

## Extending the Architecture

### Adding a New Feature

1. **Create Models** (`/models/newfeature.py`)
   - Define request/response schemas

2. **Create API Handlers** (`/api/newfeature.py`)
   - Create router with endpoints
   - Use models for validation

3. **Create Service** (`/services/newfeature_service.py`)
   - Implement business logic
   - Handle cache/API calls

4. **Register Router** (`/api/router.py`)
   - Include new router

5. **Update Config** (if needed)
   - Add new settings

---

## Best Practices

| Practice | Benefit |
|----------|---------|
| Keep API layer thin | Single responsibility |
| Use type hints | IDE support, self-documenting |
| Validate input early | Catch errors at API boundary |
| Cache strategically | Performance optimization |
| Log at each layer | Easier debugging |
| Use dependency injection | Easier testing |
| Document models | API clarity |
| Error handling in service | Consistent error responses |

---

## Common Patterns

### Repository Pattern (Data Access)
```python
class CacheRepository:
    async def get(self, key):
        pass
    
    async def set(self, key, value):
        pass
```

### Service Pattern (Business Logic)
```python
class SearchService:
    def __init__(self, cache_repo, api_client):
        self.cache = cache_repo
        self.api = api_client
    
    async def search(self, term):
        # Orchestrate cache + API logic
        pass
```

### Dependency Injection
```python
# In main.py
app.dependency_overrides[get_cache_service] = lambda: MockCacheService()
```

---

## Summary

This layered architecture provides:
✅ **Clear separation of concerns**
✅ **Ease of testing**
✅ **Code reusability**
✅ **Easy maintenance and modification**
✅ **Scalability for adding new features**
✅ **Environment flexibility**
✅ **Centralized configuration**

It's ideal for applications that need to interact with multiple external systems, implement caching strategies, and maintain clean, testable code.
