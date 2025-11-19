# ScamShield Backend - Architecture & Code Walkthrough

**Document Purpose**: Deep dive into how the ScamShield backend works
**Audience**: Developers learning the codebase
**Last Updated**: November 2025

---

## Table of Contents

1. [High-Level Architecture Overview](#1-high-level-architecture-overview)
2. [Technology Stack Explained](#2-technology-stack-explained)
3. [Project Structure Deep Dive](#3-project-structure-deep-dive)
4. [Database Models & Relationships](#4-database-models--relationships)
5. [API Endpoints Explained](#5-api-endpoints-explained)
6. [Audio Processing Workflow](#6-audio-processing-workflow)
7. [Encryption System](#7-encryption-system)
8. [Configuration Management](#8-configuration-management)
9. [Complete Call Flow Walkthrough](#9-complete-call-flow-walkthrough)
10. [Code Examples & Explanations](#10-code-examples--explanations)

---

## 1. High-Level Architecture Overview

### 1.1 What is ScamShield?

ScamShield is a **real-time scam call detection system** that:
1. Receives phone call audio recordings
2. Processes audio in 10-second chunks
3. Sends chunks to AI server for analysis
4. Returns scam probability in real-time
5. Alerts user if call is likely a scam

### 1.2 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Vue.js)                        │
│                    Mimics Phone Call Screen                      │
└───────────────┬─────────────────────────────────────────────────┘
                │
                │ HTTP/REST API
                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND (FastAPI - Python)                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    API Layer (Routers)                     │ │
│  │  • call_routes.py    - Call management                     │ │
│  │  • encryption_routes.py - Audio encryption                 │ │
│  │  • root_routes.py    - Health check                        │ │
│  └─────────────┬──────────────────────────────────────────────┘ │
│                │                                                  │
│  ┌─────────────▼──────────────────────────────────────────────┐ │
│  │                    Business Logic                          │ │
│  │  • audio_processor.py - Audio chunking & sending          │ │
│  │  • encryption.py     - AES encryption/decryption          │ │
│  └─────────────┬──────────────────────────────────────────────┘ │
│                │                                                  │
│  ┌─────────────▼──────────────────────────────────────────────┐ │
│  │                    Data Layer                              │ │
│  │  • models.py    - Database schemas                        │ │
│  │  • database.py  - DB connection & session management      │ │
│  └────────────────────────────────────────────────────────────┘ │
└───────────────┬─────────────────────────────────────────────────┘
                │
    ┌───────────┴──────────────┐
    │                          │
    ↓                          ↓
┌──────────────┐     ┌─────────────────────┐
│   SQLite     │     │   AI Server         │
│   Database   │     │   (Not Implemented) │
│              │     │   • Whisper STT     │
│  • Scammer   │     │   • FAISS Search    │
│  • CallSess. │     │   • Keyword Extract │
│  • CallCont. │     └─────────────────────┘
└──────────────┘
```

### 1.3 Key Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **API Layer** | Handles HTTP requests/responses | FastAPI |
| **Audio Processor** | Cuts audio into chunks | pydub |
| **Encryption** | Secures audio transmission | AES-GCM |
| **Database** | Stores call sessions & results | SQLite + SQLModel |
| **Background Tasks** | Async audio processing | FastAPI BackgroundTasks |
| **Cache** | Real-time probability storage | In-memory dict |

---

## 2. Technology Stack Explained

### 2.1 Core Technologies

#### **FastAPI** (Backend Framework)
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
def hello():
    return {"message": "Hello"}
```

**Why FastAPI?**
- ✅ Very fast (built on Starlette & Pydantic)
- ✅ Automatic API documentation (Swagger UI)
- ✅ Type hints for validation
- ✅ Async support out of the box
- ✅ Easy to learn

#### **SQLModel** (Database ORM)
```python
from sqlmodel import SQLModel, Field

class User(SQLModel, table=True):
    id: int = Field(primary_key=True)
    name: str
```

**Why SQLModel?**
- ✅ Combines SQLAlchemy + Pydantic
- ✅ Type-safe database operations
- ✅ Easy migrations
- ✅ Works with FastAPI seamlessly

#### **Pydantic** (Data Validation)
```python
from pydantic import BaseModel

class StartCallRequest(BaseModel):
    scammer_ph_number: str  # Automatically validated!
```

**Why Pydantic?**
- ✅ Automatic request validation
- ✅ Clear error messages
- ✅ Type safety
- ✅ JSON serialization

### 2.2 Supporting Libraries

| Library | Purpose | Example Use |
|---------|---------|-------------|
| **pydub** | Audio processing | Split audio into chunks |
| **cryptography** | Encryption | Secure audio transmission |
| **httpx** | HTTP client | Send data to AI server |
| **uvicorn** | ASGI server | Run FastAPI app |

---

## 3. Project Structure Deep Dive

### 3.1 Complete File Tree

```
backend/
├── app/                          # Main application package
│   ├── __init__.py              # Package marker
│   │
│   ├── api/                      # API route handlers
│   │   ├── __init__.py
│   │   ├── call_routes.py       # Call management endpoints
│   │   ├── encryption_routes.py # Encryption endpoints
│   │   └── root_routes.py       # Health check
│   │
│   ├── utils/                    # Utility functions
│   │   ├── __init__.py
│   │   ├── audio_processor.py   # Audio chunking logic
│   │   ├── encryption.py        # AES encryption
│   │   └── for_ai_server.py     # AI server helper
│   │
│   ├── models.py                 # Database models
│   ├── database.py               # DB connection setup
│   ├── config.py                 # Configuration settings
│   └── fastapi.py                # FastAPI app instance
│
├── tests/                        # Unit tests
│   ├── __init__.py
│   ├── conftest.py              # Pytest fixtures
│   └── test_basic_flow.py       # API tests
│
├── audio_recordings/             # Stored call recordings
│   ├── 9876543210.wav
│   └── 1234567890.m4a
│
├── audio_chunks/                 # Temporary chunks (created at runtime)
│   └── {session_uuid}/
│       ├── chunk_0.wav
│       ├── chunk_1.wav
│       └── ...
│
├── main.py                       # Application entry point
├── requirements.txt              # Python dependencies
├── pytest.ini                    # Pytest configuration
└── .env                          # Environment variables
```

### 3.2 What Each File Does

#### **main.py** - The Entry Point
```python
# This is where everything starts!

from fastapi.middleware.cors import CORSMiddleware
from app.api.root_routes import router as root
from app.api.call_routes import router as call_routes
from app.database import create_db_and_tables
from app.fastapi import app

# Include all routers (connect API endpoints)
app.include_router(root)
app.include_router(call_routes)

# Allow frontend to make requests (CORS)
app.add_middleware(CORSMiddleware, allow_origins=["*"])

# Create database tables on startup
@app.on_event("startup")
def on_startup():
    create_db_and_tables()
```

**What happens when you run `uvicorn main:app`?**
1. Python imports `main.py`
2. FastAPI app is created
3. All routers are registered
4. Database tables are created
5. Server starts listening on http://localhost:8000

---

## 4. Database Models & Relationships

### 4.1 Database Schema

```
┌─────────────────┐
│    Scammer      │
├─────────────────┤
│ id (PK)         │
│ ph_number       │◄─────────┐
│ total_scam_count│          │
│ last_checked    │          │ Foreign Key
└─────────────────┘          │
                              │
                    ┌─────────┴──────────┐
                    │                    │
          ┌─────────▼────────┐  ┌───────▼────────┐
          │   CallSession    │  │  CallContent   │
          ├──────────────────┤  ├────────────────┤
          │ id (PK)          │  │ id (PK)        │
          │ session_uuid     │◄─┤ call_uuid (FK) │
          │ scammer_ph_number│  │ ph_number (FK) │
          │ start_time       │  │ keywords       │
          │ end_time         │  │ transcription  │
          │ call_detected... │  │ output_simil.. │
          │ is_scam_weight   │  └────────────────┘
          └──────────────────┘
```

### 4.2 Model Explanations

#### **Scammer Model**
```python
class Scammer(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    ph_number: str = Field(index=True, unique=True)  # Phone number (unique)
    total_scam_count: int = Field(default=0)         # How many scam calls
    last_checked: Optional[datetime] = None          # Last time checked

    # Relationships (connects to other tables)
    call_sessions: List["CallSession"] = Relationship(back_populates="scammer")
    call_contents: List["CallContent"] = Relationship(back_populates="scammer")
```

**Purpose**: Store information about scammer phone numbers
**Example Data**:
```
id  | ph_number   | total_scam_count | last_checked
----|-------------|------------------|------------------
1   | 9876543210  | 5                | 2025-11-19 12:00
2   | 1234567890  | 2                | 2025-11-18 10:30
```

#### **CallSession Model**
```python
class CallSession(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    session_uuid: str = Field(default_factory=lambda: str(uuid4()), unique=True)
    scammer_ph_number: str = Field(foreign_key="scammer.ph_number")
    start_time: datetime = Field(default_factory=datetime.now)
    end_time: Optional[datetime] = None

    # AI prediction results
    call_detected_as_scam: bool = Field(default=False)  # Is it a scam?
    is_scam_weight: Optional[float] = None              # Probability (0.0-1.0)
```

**Purpose**: Store individual call session data
**Example Data**:
```
session_uuid          | scammer_ph_number | start_time | detected_as_scam | is_scam_weight
----------------------|-------------------|------------|------------------|----------------
abc-123-def-456       | 9876543210        | 12:00:00   | true             | 0.95
xyz-789-ghi-012       | 1234567890        | 12:05:00   | false            | 0.12
```

#### **CallContent Model**
```python
class CallContent(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    ph_number: str = Field(foreign_key="scammer.ph_number")
    call_uuid: str = Field(foreign_key="callsession.session_uuid")

    # AI analysis results
    keywords: List[str] = Field(default_factory=list, sa_column=Column(JSON))
    transcription: Optional[str] = None
    window_transcriptions: List[str] = Field(default_factory=list, sa_column=Column(JSON))
    output_similarity: Optional[float] = None
```

**Purpose**: Store AI analysis results (transcription, keywords)
**Example Data**:
```
call_uuid       | keywords                    | transcription
----------------|-----------------------------|---------------------------------
abc-123-def-456 | ["OTP", "bank", "verify"]  | "Please provide your OTP..."
```

### 4.3 Relationships Explained

**One Scammer → Many CallSessions**
```python
# One scammer phone number can have multiple call sessions
scammer = session.get(Scammer, 1)
print(scammer.call_sessions)  # List of all calls from this number
```

**One CallSession → One CallContent**
```python
# Each call session has one content analysis
call = session.get(CallSession, 1)
print(call.call_contents)  # Analysis for this call
```

---

## 5. API Endpoints Explained

### 5.1 Call Management Endpoints

#### **1. POST /call/start-call** - Start a New Call

**Location**: `app/api/call_routes.py:58`

**What it does**:
1. Receives phone number
2. Checks if audio file exists
3. Creates CallSession in database
4. Renames audio file (adds UUID)
5. Starts background audio processing
6. Returns session_uuid to frontend

**Code Flow**:
```python
@router.post("/call/start-call")
async def start_call(request: StartCallRequest, background_tasks, session):
    # Step 1: Check if audio file exists
    audio_file = audio_processor.get_audio_file(request.scammer_ph_number)
    if not audio_file:
        raise HTTPException(404, detail="No audio file found")

    # Step 2: Create database entry
    new_session = CallSession(
        scammer_ph_number=request.scammer_ph_number,
        start_time=datetime.now()
    )
    session.add(new_session)
    session.commit()

    # Step 3: Initialize cache (for real-time updates)
    probability_cache[new_session.session_uuid] = {
        "probability": 0.0,
        "is_scam": False,
        "chunk_index": -1
    }

    # Step 4: Start background processing
    background_tasks.add_task(
        process_audio_background,
        new_session.session_uuid,
        request.scammer_ph_number,
        session
    )

    return {"session_uuid": new_session.session_uuid}
```

**Request Example**:
```json
POST /call/start-call
{
    "scammer_ph_number": "9876543210"
}
```

**Response Example**:
```json
{
    "session_uuid": "abc-123-def-456",
    "start_time": "2025-11-19T12:00:00",
    "message": "Call started - audio processing running in parallel"
}
```

---

#### **2. GET /call/probability/{session_uuid}** - Get Real-Time Probability

**Location**: `app/api/call_routes.py:124`

**What it does**:
1. Receives session UUID
2. Looks up probability in in-memory cache
3. Returns current scam probability
4. Frontend polls this every 2 seconds

**Why use cache instead of database?**
- ✅ **Much faster** (RAM vs disk)
- ✅ **No database locks** (concurrent access)
- ✅ **Real-time updates** (as AI processes chunks)

**Code Flow**:
```python
# In-memory cache (Python dictionary)
probability_cache = {}

@router.get("/call/probability/{session_uuid}")
def get_probability(session_uuid: str):
    # Check if session exists in cache
    if session_uuid not in probability_cache:
        raise HTTPException(404, detail="Session not found")

    # Return cached probability (instant!)
    return probability_cache[session_uuid]
```

**How Cache Gets Updated**:
```python
# When AI returns result for a chunk:
async def handle_ai_response(...):
    result = await send_chunk_to_ai(...)  # Get AI result

    # Update cache immediately
    cache[session_uuid] = {
        "probability": result["probability"],  # New probability!
        "is_scam": result["probability"] >= 0.8,
        "chunk_index": chunk_number,
        "timestamp": datetime.now().isoformat()
    }
```

**Request Example**:
```
GET /call/probability/abc-123-def-456
```

**Response Example**:
```json
{
    "probability": 0.75,
    "is_scam": false,
    "chunk_index": 3,
    "timestamp": "2025-11-19T12:00:30"
}
```

**Frontend Polling Pattern**:
```javascript
// Frontend calls this every 2 seconds
setInterval(async () => {
    const response = await fetch(`/call/probability/${sessionUUID}`);
    const data = await response.json();

    if (data.is_scam) {
        showScamAlert();  // Show warning to user!
    }
}, 2000);  // Every 2 seconds
```

---

#### **3. PUT /call/stop/{session_uuid}** - Stop a Call

**Location**: `app/api/call_routes.py:195`

**What it does**:
1. Receives session UUID
2. Sets end_time in database
3. Calculates call duration
4. Returns duration

**Code Flow**:
```python
@router.put("/call/stop/{session_uuid}")
def stop_call(session_uuid: str, db: Session):
    # Find call session in database
    call_session = db.exec(
        select(CallSession).where(CallSession.session_uuid == session_uuid)
    ).first()

    if not call_session:
        raise HTTPException(404, detail="Session not found")

    # Check if already stopped
    if call_session.end_time is not None:
        raise HTTPException(400, detail="Call already stopped")

    # Mark call as ended
    call_session.end_time = datetime.now()
    db.commit()

    # Calculate duration
    duration = (call_session.end_time - call_session.start_time).total_seconds()

    return {"duration": duration, "end_time": call_session.end_time}
```

---

#### **4. GET /call/results/{session_uuid}** - Get Complete Analysis

**Location**: `app/api/call_routes.py:292`

**What it does**:
1. Returns complete call analysis
2. Includes transcription, keywords, probability
3. Called after call ends

**Response Structure**:
```json
{
    "session_uuid": "abc-123-def-456",
    "scammer_ph_number": "9876543210",
    "start_time": "2025-11-19T12:00:00",
    "end_time": "2025-11-19T12:05:00",
    "duration": 300.0,

    "detected_as_scam": true,
    "probability": 0.95,
    "risk_level": "high",

    "keywords": ["OTP", "bank", "verify", "CVV"],
    "transcription": "Full call transcription here...",
    "window_transcriptions": [
        "Hello sir, your loan has been approved",
        "Please provide your bank details",
        "What is your OTP number?"
    ]
}
```

---

## 6. Audio Processing Workflow

### 6.1 Complete Audio Processing Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    AUDIO PROCESSING FLOW                     │
└─────────────────────────────────────────────────────────────┘

Step 1: Call Started
├─ Frontend sends phone number
├─ Backend finds audio file: "9876543210.wav"
└─ File exists: ✓

Step 2: Rename File
├─ Generate UUID: "abc-123-def-456"
├─ Rename: "9876543210.wav" → "abc-123-def-456_9876543210.wav"
└─ Prevents conflicts

Step 3: Load Audio File
├─ Read entire file into memory
├─ File: "abc-123-def-456_9876543210.wav"
├─ Duration: 60 seconds
└─ Format: WAV, 44.1kHz

Step 4: Cut into Chunks (Every 10 seconds)
├─ Chunk 0: 0-10s    → "chunk_0.wav"
├─ Chunk 1: 10-20s   → "chunk_1.wav"
├─ Chunk 2: 20-30s   → "chunk_2.wav"
├─ Chunk 3: 30-40s   → "chunk_3.wav"
├─ Chunk 4: 40-50s   → "chunk_4.wav"
└─ Chunk 5: 50-60s   → "chunk_5.wav"

Step 5: Process Each Chunk (Parallel!)
For each chunk:
  ├─ Encrypt chunk (AES-256)
  ├─ Send to AI server (async)
  ├─ AI returns: {"probability": 0.75, "keywords": ["bank"]}
  ├─ Update database
  └─ Update cache (for real-time polling)

Step 6: Wait 10 Seconds Between Chunks
├─ Chunk 0 sent at T+0s
├─ Wait 10 seconds...
├─ Chunk 1 sent at T+10s
├─ Wait 10 seconds...
└─ (Simulates real-time processing)

Step 7: Aggregate Results
├─ Collect all transcriptions
├─ Merge all keywords
├─ Take MAX probability
└─ Store final results in CallContent table
```

### 6.2 Audio Processor Code Walkthrough

#### **Location**: `app/utils/audio_processor.py`

```python
class AudioProcessor:
    def __init__(self):
        self.audio_folder = 'audio_recordings'
        self.chunks_folder = 'audio_chunks'
        self.chunk_seconds = 10  # 10-second chunks
        self.ai_server_url = 'http://localhost:5000/analyze'
```

#### **Method 1: Find Audio File**
```python
def get_audio_file(self, phone_number):
    """
    Find audio file for given phone number.
    Supports: .mp3, .wav, .ogg, .m4a
    """
    folder = Path(self.audio_folder)

    # Try each extension
    for ext in ['.mp3', '.wav', '.ogg', '.m4a']:
        file_path = folder / f"{phone_number}{ext}"
        if file_path.exists():
            return str(file_path)  # Found it!

    return None  # Not found
```

**Example**:
```python
audio_file = audio_processor.get_audio_file("9876543210")
# Returns: "audio_recordings/9876543210.wav"
```

---

#### **Method 2: Cut Audio into Chunks**
```python
def cut_audio_into_chunks(self, audio_file, session_uuid):
    """
    Cut audio file into 10-second chunks.
    """
    # Load audio file
    audio = AudioSegment.from_file(audio_file)
    chunk_ms = self.chunk_seconds * 1000  # 10s = 10000ms

    # Create folder for this session's chunks
    session_folder = Path(self.chunks_folder) / session_uuid
    session_folder.mkdir(parents=True, exist_ok=True)

    chunk_paths = []
    position = 0
    chunk_number = 0

    # Loop through audio, cutting 10s at a time
    while position < len(audio):
        # Extract 10-second chunk
        chunk = audio[position:position + chunk_ms]

        # Save chunk to disk
        chunk_path = session_folder / f"chunk_{chunk_number}.wav"
        chunk.export(str(chunk_path), format="wav")
        chunk_paths.append(str(chunk_path))

        # Move to next chunk
        position += chunk_ms
        chunk_number += 1

    return chunk_paths  # List of all chunk paths
```

**Example**:
```python
chunks = audio_processor.cut_audio_into_chunks(
    "abc-123_9876543210.wav",
    "abc-123-def-456"
)
# Returns: [
#     "audio_chunks/abc-123-def-456/chunk_0.wav",
#     "audio_chunks/abc-123-def-456/chunk_1.wav",
#     ...
# ]
```

---

#### **Method 3: Send Chunk to AI Server**
```python
async def send_chunk_to_ai(self, chunk_path, session_uuid, chunk_number, phone_number):
    """
    Encrypt chunk and send to AI server for analysis.
    """
    # Step 1: Encrypt the chunk
    ciphertext, nonce, salt = self.encryptor.encrypt_file(chunk_path)

    # Step 2: Encode for transmission (Base64)
    encrypted_payload = SecureAudioEncryption.encode_for_transmission(
        ciphertext, nonce, salt
    )

    # Step 3: Prepare request
    async with httpx.AsyncClient(timeout=30.0) as client:
        data = {
            "session_uuid": session_uuid,
            "chunk_index": chunk_number,
            "phone_number": phone_number,
            "encrypted_audio": encrypted_payload  # Encrypted!
        }

        # Step 4: Send to AI server
        response = await client.post(self.ai_server_url, json=data)
        result = response.json()

        # Step 5: Return AI analysis
        return {
            "probability": result.get("probability", 0.0),
            "keywords": result.get("keywords", []),
            "transcription": result.get("transcription", "")
        }
```

**What AI Server Returns**:
```json
{
    "probability": 0.85,
    "keywords": ["OTP", "bank", "verify"],
    "transcription": "Please provide your OTP for verification"
}
```

---

#### **Method 4: Process Audio (Main Function)**
```python
async def process_audio(self, audio_file, session_uuid, phone_number, db_session, cache):
    """
    Main function: Cut chunks every 10 seconds and send to AI immediately.
    Runs in background - doesn't block the API response!
    """
    # Load audio file
    audio = AudioSegment.from_file(audio_file)
    chunk_ms = self.chunk_seconds * 1000  # 10000ms

    # Create folder for chunks
    session_folder = Path(self.chunks_folder) / session_uuid
    session_folder.mkdir(parents=True, exist_ok=True)

    # Store async tasks (for parallel processing)
    ai_tasks = []

    position = 0
    chunk_number = 0

    # Process chunks in real-time
    while position < len(audio):
        # Cut ONE chunk
        chunk = audio[position:position + chunk_ms]

        # Save chunk to disk
        chunk_path = session_folder / f"chunk_{chunk_number}.wav"
        chunk.export(str(chunk_path), format="wav")

        # Send to AI immediately (don't wait for response!)
        task = asyncio.create_task(
            self.handle_ai_response(
                str(chunk_path), session_uuid, chunk_number,
                phone_number, db_session, cache
            )
        )
        ai_tasks.append(task)

        # Move to next chunk
        position += chunk_ms
        chunk_number += 1

        # Wait 10 seconds before cutting next chunk (simulate real-time)
        if position < len(audio):
            await asyncio.sleep(self.chunk_seconds)

    # Wait for all AI responses to complete
    results = await asyncio.gather(*ai_tasks)

    # Aggregate final results
    all_transcriptions = []
    all_keywords = []
    max_probability = 0.0

    for result in results:
        if isinstance(result, dict):
            all_transcriptions.append(result.get("transcription", ""))
            all_keywords.extend(result.get("keywords", []))
            max_probability = max(max_probability, result.get("probability", 0.0))

    # Save final results to database
    call_content = CallContent(
        ph_number=phone_number,
        call_uuid=session_uuid,
        keywords=list(set(all_keywords)),  # Remove duplicates
        transcription=" ".join(all_transcriptions),
        window_transcriptions=all_transcriptions,
        output_similarity=max_probability
    )
    db_session.add(call_content)
    db_session.commit()
```

---

## 7. Encryption System

### 7.1 Why Encrypt Audio?

**Security Reasons**:
- Protect user privacy
- Secure transmission over network
- Prevent audio tampering
- Comply with data protection laws

### 7.2 Encryption Algorithm: AES-256-GCM

**AES-256-GCM** = Advanced Encryption Standard with Galois/Counter Mode
- **256-bit key**: Very secure (2^256 possible keys)
- **Authenticated**: Prevents tampering
- **Industry standard**: Used by banks, military

### 7.3 Encryption Process

```
┌────────────────────────────────────────────────────────┐
│                 ENCRYPTION WORKFLOW                     │
└────────────────────────────────────────────────────────┘

Step 1: Generate Key from Password
├─ Input: "scamshield_password"
├─ Salt: Random 16 bytes
├─ Algorithm: PBKDF2 (100,000 iterations)
└─ Output: 256-bit encryption key

Step 2: Encrypt Audio File
├─ Input: "chunk_0.wav" (binary data)
├─ Generate: Random nonce (12 bytes)
├─ Encrypt: Using AES-256-GCM
└─ Output: Ciphertext (encrypted audio)

Step 3: Package for Transmission
├─ Combine: salt_length + salt + nonce_length + nonce + ciphertext
├─ Encode: Base64 (safe for JSON)
└─ Output: "dGVzdC1lbmNyeXB0ZWQtZGF0YQ..."

Step 4: Send to AI Server
├─ Payload: {"encrypted_audio": "dGVzdC..."}
└─ Transmitted securely!

Step 5: Decrypt on AI Server
├─ Decode: Base64 → binary
├─ Extract: salt, nonce, ciphertext
├─ Derive key: From password + salt
├─ Decrypt: Using AES-256-GCM
└─ Output: Original "chunk_0.wav"
```

### 7.4 Encryption Code Walkthrough

```python
class SecureAudioEncryption:
    def __init__(self, password: str, salt: Optional[bytes] = None):
        """Initialize encryptor with password."""
        # Generate or use provided salt
        if salt is None:
            self.salt = os.urandom(16)  # Random 16 bytes
        else:
            self.salt = salt

        # Derive encryption key from password
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,                    # 256 bits = 32 bytes
            salt=self.salt,
            iterations=100000,            # Slow down brute-force attacks
            backend=default_backend()
        )
        self.key = kdf.derive(password.encode())

        # Create AES-GCM cipher
        self.aesgcm = AESGCM(self.key)

    def encrypt_file(self, filepath: str) -> Tuple[bytes, bytes, bytes]:
        """Encrypt entire file and return (ciphertext, nonce, salt)."""
        # Read file
        with open(filepath, "rb") as f:
            data = f.read()

        # Generate random nonce (number used once)
        nonce = os.urandom(12)

        # Encrypt data
        ciphertext = self.aesgcm.encrypt(nonce, data, None)

        return ciphertext, nonce, self.salt
```

**Example Usage**:
```python
# Encrypt a file
enc = SecureAudioEncryption("my_secret_password")
ciphertext, nonce, salt = enc.encrypt_file("chunk_0.wav")

# Package for transmission
encoded = SecureAudioEncryption.encode_for_transmission(ciphertext, nonce, salt)
# Output: "dGVzdC1lbmNyeXB0ZWQtZGF0YQ..."

# Send to AI server
response = requests.post("http://ai-server/analyze", json={
    "encrypted_audio": encoded
})
```

---

## 8. Configuration Management

### 8.1 Configuration File: `app/config.py`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """Application configuration settings"""

    # Audio processing folders
    AUDIO_RECORDINGS_FOLDER: str = 'audio_recordings'
    AUDIO_CHUNKS_FOLDER: str = 'audio_chunks'

    # Processing settings
    CHUNK_DURATION_SECONDS: int = 10

    # AI server configuration
    AI_SERVER_URL: str = 'http://localhost:5000/analyze'

    # Security
    ENCRYPTION_PASSWORD: str = 'scamshield_password'

    class Config:
        env_file = ".env"  # Load from .env file
        env_file_encoding = "utf-8"

# Create singleton instance
settings = Settings()
```

### 8.2 Environment Variables (.env file)

```bash
# .env file
DATABASE_URL=sqlite:///./app.db
DEBUG=true
ENVIRONMENT=development
SECRET_KEY=change-this-in-production

# Can override config.py values:
# CHUNK_DURATION_SECONDS=15
# AI_SERVER_URL=http://production-ai-server.com/analyze
```

### 8.3 Using Settings in Code

```python
from app.config import settings

# Access settings anywhere
audio_folder = settings.AUDIO_RECORDINGS_FOLDER
chunk_duration = settings.CHUNK_DURATION_SECONDS
ai_url = settings.AI_SERVER_URL
```

---

## 9. Complete Call Flow Walkthrough

### 9.1 Step-by-Step: User Receives Scam Call

```
┌─────────────────────────────────────────────────────────────┐
│           COMPLETE CALL FLOW - FROM START TO END            │
└─────────────────────────────────────────────────────────────┘

T+0s: User Receives Call
├─ Phone rings: Unknown number "9876543210"
├─ User clicks "Answer" in ScamShield app
└─ Frontend sends: POST /call/start-call {"scammer_ph_number": "9876543210"}

T+0.1s: Backend Processes Start Request
├─ Find audio file: audio_recordings/9876543210.wav ✓
├─ Create database entry:
│   └─ CallSession(session_uuid="abc-123", start_time=now())
├─ Initialize cache:
│   └─ probability_cache["abc-123"] = {"probability": 0.0, "is_scam": false}
├─ Start background task (audio processing)
└─ Return to frontend: {"session_uuid": "abc-123"}

T+0.2s: Frontend Starts Polling
├─ Display: "Analyzing call..."
├─ Start timer to poll every 2 seconds
└─ GET /call/probability/abc-123

T+0.2s: Background Processing Begins
├─ Rename file: 9876543210.wav → abc-123_9876543210.wav
├─ Load audio: Duration 60 seconds
└─ Start chunking loop...

T+0-10s: Process Chunk 0
├─ Cut chunk: audio[0:10s] → chunk_0.wav
├─ Encrypt chunk: AES-256-GCM
├─ Send to AI server: POST http://ai-server/analyze
│   └─ {"session_uuid": "abc-123", "chunk_index": 0, "encrypted_audio": "..."}
└─ Wait for AI response...

T+2s: Frontend Polls (1st time)
├─ GET /call/probability/abc-123
├─ Response: {"probability": 0.0, "chunk_index": -1}  # Still processing
└─ Display: "Probability: 0%"

T+5s: AI Returns Result for Chunk 0
├─ AI Response: {"probability": 0.3, "keywords": ["hello"], "transcription": "Hello sir"}
├─ Update cache: probability_cache["abc-123"]["probability"] = 0.3
└─ Update database: CallSession.is_scam_weight = 0.3

T+4s: Frontend Polls (2nd time)
├─ GET /call/probability/abc-123
├─ Response: {"probability": 0.3, "chunk_index": 0}  # Updated!
└─ Display: "Probability: 30%" (green - safe)

T+10s: Wait Period (Simulate Real-Time)
└─ Sleep 10 seconds before next chunk...

T+10-20s: Process Chunk 1
├─ Cut chunk: audio[10:20s] → chunk_1.wav
├─ Encrypt and send to AI
└─ Wait for response...

T+6s: Frontend Polls (3rd time)
├─ Still 0.3 (chunk 1 not yet processed)
└─ Display: "Probability: 30%"

T+15s: AI Returns Result for Chunk 1
├─ AI Response: {"probability": 0.6, "keywords": ["bank", "account"]}
├─ Update cache: probability = 0.6
└─ Update database

T+8s: Frontend Polls (4th time)
├─ Response: {"probability": 0.6, "chunk_index": 1}
└─ Display: "Probability: 60%" (yellow - medium risk)

T+20-30s: Process Chunk 2
├─ Cut chunk 2
├─ Send to AI
└─ AI Response: {"probability": 0.85, "keywords": ["OTP", "verify"]}

T+10s: Frontend Polls (5th time)
├─ Response: {"probability": 0.85, "is_scam": true}  # SCAM DETECTED!
└─ Frontend Action:
    ├─ Show RED alert: "⚠️ SCAM CALL DETECTED"
    ├─ Vibrate phone
    ├─ Play warning sound
    └─ Suggest: "Hang up immediately"

T+30s: User Hangs Up
├─ Frontend sends: PUT /call/stop/abc-123
└─ Backend marks call as ended

T+30.1s: Backend Stops Call
├─ Update database: end_time = now()
├─ Calculate duration: 30 seconds
└─ Return: {"duration": 30.0, "end_time": "..."}

T+31s: Frontend Requests Full Results
├─ GET /call/results/abc-123
└─ Response:
    {
      "detected_as_scam": true,
      "probability": 0.85,
      "risk_level": "high",
      "keywords": ["hello", "bank", "account", "OTP", "verify"],
      "transcription": "Hello sir, your bank account needs verification...",
      "duration": 30.0
    }

T+32s: Frontend Displays Results
├─ Show summary screen:
│   ├─ "This call was flagged as SCAM"
│   ├─ "Risk Level: HIGH (85%)"
│   ├─ "Keywords detected: OTP, bank, verify"
│   └─ "Full transcription: ..."
└─ User is safe! ✓
```

---

## 10. Code Examples & Explanations

### 10.1 How Background Tasks Work

**Problem**: Audio processing takes time (10+ seconds). We can't make the user wait!

**Solution**: FastAPI Background Tasks

```python
from fastapi import BackgroundTasks

@app.post("/call/start-call")
async def start_call(background_tasks: BackgroundTasks):
    # This returns IMMEDIATELY
    return {"message": "Processing started"}

    # Meanwhile, this runs in background:
    background_tasks.add_task(slow_function)

def slow_function():
    # This runs AFTER the response is sent
    time.sleep(60)  # Takes 1 minute
    print("Done!")
```

**Timeline**:
```
T+0s: User sends request
T+0.1s: Backend returns response (fast!)
T+0.1s-60s: slow_function() runs in background
T+60s: Background task completes
```

### 10.2 How Database Sessions Work

```python
from sqlmodel import Session

# Get database session
def get_session():
    with Session(engine) as session:
        yield session  # Provides session to endpoint
    # Automatically closes session when done

# Use in endpoint
@app.get("/users")
def get_users(session: Session = Depends(get_session)):
    # session is automatically provided by FastAPI
    users = session.exec(select(User)).all()
    return users
    # Session automatically closed here
```

### 10.3 How Dependency Injection Works

```python
from fastapi import Depends

# Define a dependency
def get_current_user():
    return {"username": "john"}

# Use in endpoint
@app.get("/profile")
def get_profile(user = Depends(get_current_user)):
    # FastAPI automatically calls get_current_user()
    # and passes result to this function
    return {"profile": user["username"]}
```

### 10.4 How Async/Await Works

```python
import asyncio

# Synchronous (blocking)
def slow_task():
    time.sleep(5)  # Blocks everything for 5 seconds
    return "done"

# Asynchronous (non-blocking)
async def fast_task():
    await asyncio.sleep(5)  # Allows other tasks to run
    return "done"

# Run multiple tasks in parallel
async def main():
    task1 = asyncio.create_task(fast_task())
    task2 = asyncio.create_task(fast_task())
    task3 = asyncio.create_task(fast_task())

    # All 3 run in parallel!
    results = await asyncio.gather(task1, task2, task3)
    # Total time: 5 seconds (not 15!)
```

---

## Summary: Key Takeaways

### What You Should Understand Now

1. **Architecture**:
   - FastAPI handles HTTP requests
   - SQLModel manages database
   - Background tasks process audio
   - Cache provides real-time updates

2. **Data Flow**:
   - Frontend → API → Database
   - Audio → Chunks → AI Server
   - AI Results → Cache → Frontend

3. **Key Components**:
   - `call_routes.py` - API endpoints
   - `audio_processor.py` - Audio chunking
   - `models.py` - Database structure
   - `encryption.py` - Security

4. **Critical Concepts**:
   - Background tasks for async work
   - In-memory cache for speed
   - Encryption for security
   - Database relationships

### Next Steps for Learning

1. **Read the Code**: Start with `main.py`, follow the imports
2. **Run the Server**: `uvicorn main:app --reload`
3. **Test Endpoints**: Use Swagger UI at http://localhost:8000/docs
4. **Debug**: Add `print()` statements to see flow
5. **Modify**: Try changing chunk duration or thresholds

---

**Questions to Think About:**

1. What happens if AI server is down?
2. How would you add user authentication?
3. Could we process multiple calls simultaneously?
4. How would you optimize for production?

---

**END OF ARCHITECTURE DOCUMENT**

*Version*: 1.0
*Last Updated*: November 19, 2025
*For*: Learning & Understanding ScamShield Backend