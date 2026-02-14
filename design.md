# System Design: SkillWeave Platform

## 1. Overview

SkillWeave transforms YouTube tutorials into interactive, mastery-based courses using LLM-powered content generation and semantic embeddings for intelligent learning analytics.

### Core Goals
- Process YouTube videos into structured courses
- Generate quizzes and coding exercises automatically
- Track user progress and mastery
- Detect weaknesses and adapt learning content
- Visualize concept relationships

## 2. Architecture

### High-Level Structure
```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│  - Course browsing  - Learning interface                │
│  - Quiz taking      - Progress dashboard                │
│  - Concept maps     - Weakness reports                  │
└─────────────────────────────────────────────────────────┘
                          │ REST API
┌─────────────────────────────────────────────────────────┐
│                  Backend (FastAPI)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Auth         │  │ Course       │  │ Intelligence │  │
│  │ Service      │  │ Service      │  │ Service      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ Content Gen  │  │ Progress     │                    │
│  │ Service      │  │ Service      │                    │
│  └──────────────┘  └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│              Data & External Services                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ PostgreSQL   │  │ Redis        │  │ YouTube API  │  │
│  │ + pgvector   │  │ (cache)      │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐                                       │
│  │ OpenAI API   │                                       │
│  │ (LLM + embed)│                                       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### Tech Stack
- **Frontend**: React + TypeScript, Redux Toolkit, Material-UI, D3.js
- **Backend**: FastAPI (Python 3.11+), Pydantic, SQLAlchemy
- **Database**: PostgreSQL 15+ with pgvector extension
- **Cache**: Redis
- **LLM**: OpenAI GPT-4 (content gen) + text-embedding-3-large (embeddings)
- **Deployment**: Docker Compose (hackathon), simple cloud hosting

## 3. Data Models

### Core Entities

```python
# User & Auth
class User:
    id: UUID
    email: str
    hashed_password: str
    full_name: str
    created_at: datetime

# Course Structure
class Course:
    id: UUID
    title: str
    description: str
    youtube_url: str
    youtube_video_id: str
    duration_seconds: int
    thumbnail_url: str
    status: str  # PROCESSING, READY, FAILED
    created_at: datetime

class Topic:
    id: UUID
    course_id: UUID
    title: str
    description: str
    order: int
    start_timestamp: int
    end_timestamp: int
    estimated_duration_minutes: int

# Assessment
class Quiz:
    id: UUID
    topic_id: UUID
    difficulty: str  # BEGINNER, INTERMEDIATE, ADVANCED

class Question:
    id: UUID
    quiz_id: UUID
    question_text: str
    options: List[str]  # 4 options
    correct_answer_index: int
    explanation: str

class CodingExercise:
    id: UUID
    topic_id: UUID
    title: str
    description: str
    starter_code: str
    language: str
    test_cases: List[dict]

# Progress Tracking
class UserProgress:
    id: UUID
    user_id: UUID
    course_id: UUID
    completed_topics: List[UUID]
    overall_progress_percentage: float

class QuizSubmission:
    id: UUID
    user_id: UUID
    quiz_id: UUID
    answers: List[int]
    score: float
    submitted_at: datetime

class TopicMastery:
    id: UUID
    user_id: UUID
    topic_id: UUID
    mastery_percentage: float  # 0-100
    last_updated: datetime

# Intelligence
class Concept:
    id: UUID
    name: str
    description: str
    topic_id: UUID
    embedding_vector: List[float]  # 1536-dim

class UserConceptMastery:
    id: UUID
    user_id: UUID
    concept_id: UUID
    mastery_score: float  # 0.0-1.0
    attempts: int
    last_assessed: datetime

class Weakness:
    id: UUID
    user_id: UUID
    concept_id: UUID
    severity: float  # 0.0-1.0
    detected_at: datetime
```

## 4. API Design

### Authentication
```
POST   /api/auth/register          # Register new user
POST   /api/auth/login             # Login (returns JWT)
POST   /api/auth/refresh           # Refresh token
```

### Courses
```
POST   /api/courses/process        # Submit YouTube URL
GET    /api/courses/status/{job_id} # Check processing status
GET    /api/courses                # List courses
GET    /api/courses/{id}           # Get course details
GET    /api/courses/{id}/outline   # Get course outline
```

### Learning
```
GET    /api/courses/{id}/topics/{topic_id}          # Get topic
GET    /api/courses/{id}/topics/{topic_id}/quiz     # Get quiz
GET    /api/courses/{id}/topics/{topic_id}/exercise # Get exercise
POST   /api/progress/quiz                           # Submit quiz
POST   /api/progress/exercise                       # Submit exercise
GET    /api/progress/mastery/{course_id}            # Get mastery
```

### Intelligence
```
GET    /api/intelligence/weaknesses           # Get user weaknesses
GET    /api/intelligence/concept-map          # Get concept map
GET    /api/intelligence/adaptive-quiz        # Generate adaptive quiz
POST   /api/intelligence/quiz-feedback        # Submit for analysis
GET    /api/intelligence/recommendations      # Get recommendations
```

## 5. Core Algorithms

### 5.1 Video Processing Pipeline

```python
async def process_video(youtube_url: str) -> Course:
    """
    1. Extract metadata from YouTube
    2. Get transcript
    3. Segment into topics using LLM
    4. Generate outline
    5. Create quizzes and exercises
    6. Generate embeddings
    """
    # 1. Extract metadata
    video_info = await youtube_api.get_video_info(youtube_url)
    
    # 2. Get transcript
    transcript = await youtube_api.get_transcript(video_info.video_id)
    
    # 3. Segment into topics
    topics = await llm_service.segment_content(transcript)
    
    # 4. Generate outline
    outline = await llm_service.generate_outline(topics)
    
    # 5. Generate assessments (parallel)
    for topic in topics:
        await asyncio.gather(
            llm_service.generate_quiz(topic),
            llm_service.generate_exercise(topic)
        )
    
    # 6. Generate embeddings
    concepts = extract_concepts(topics)
    embeddings = await openai.embeddings.create(
        model="text-embedding-3-large",
        input=[c.description for c in concepts]
    )
    
    return course
```

### 5.2 Concept Clustering

```python
def cluster_concepts(concepts: List[Concept], n_clusters: int = 10):
    """K-means clustering on embeddings"""
    from sklearn.cluster import KMeans
    
    embeddings = np.array([c.embedding_vector for c in concepts])
    kmeans = KMeans(n_clusters=n_clusters, random_state=42)
    labels = kmeans.fit_predict(embeddings)
    
    clusters = []
    for i in range(n_clusters):
        cluster_concepts = [c for c, l in zip(concepts, labels) if l == i]
        clusters.append({
            'id': i,
            'concepts': cluster_concepts,
            'centroid': kmeans.cluster_centers_[i]
        })
    
    return clusters
```

### 5.3 Weakness Detection

```python
def detect_weaknesses(user_id: UUID, threshold: float = 0.7):
    """Identify concepts where mastery < threshold"""
    mastery_records = get_user_concept_mastery(user_id)
    
    weaknesses = []
    for record in mastery_records:
        if record.mastery_score < threshold:
            severity = 1.0 - record.mastery_score
            weaknesses.append({
                'concept_id': record.concept_id,
                'severity': severity,
                'attempts': record.attempts
            })
    
    # Sort by severity (worst first)
    return sorted(weaknesses, key=lambda w: w['severity'], reverse=True)
```

### 5.4 Adaptive Quiz Generation

```python
def generate_adaptive_quiz(user_id: UUID, num_questions: int = 10):
    """Generate quiz targeting weaknesses"""
    weaknesses = detect_weaknesses(user_id)
    recent_performance = get_recent_quiz_scores(user_id, limit=5)
    
    # Adjust difficulty based on recent performance
    avg_score = np.mean([p.score for p in recent_performance])
    if avg_score > 0.85:
        difficulty_level = 'ADVANCED'
    elif avg_score < 0.60:
        difficulty_level = 'BEGINNER'
    else:
        difficulty_level = 'INTERMEDIATE'
    
    # Question distribution: 60% weak, 30% moderate, 10% strong
    weak_count = int(num_questions * 0.6)
    moderate_count = int(num_questions * 0.3)
    strong_count = num_questions - weak_count - moderate_count
    
    questions = []
    
    # Weak concepts
    weak_concepts = [w['concept_id'] for w in weaknesses[:3]]
    questions.extend(
        get_questions_for_concepts(weak_concepts, weak_count, difficulty_level)
    )
    
    # Moderate and strong concepts
    questions.extend(get_moderate_questions(moderate_count, difficulty_level))
    questions.extend(get_strong_questions(strong_count, difficulty_level))
    
    random.shuffle(questions)
    return questions
```

### 5.5 Mastery Calculation

```python
def calculate_topic_mastery(user_id: UUID, topic_id: UUID) -> float:
    """
    Mastery = (Quiz Score * 0.6) + (Exercise Completion * 0.4)
    """
    quiz_submissions = get_quiz_submissions(user_id, topic_id)
    exercise_submissions = get_exercise_submissions(user_id, topic_id)
    
    # Best quiz score from last 3 attempts
    recent_scores = sorted([s.score for s in quiz_submissions[-3:]], reverse=True)
    quiz_score = recent_scores[0] if recent_scores else 0.0
    
    # Exercise completion rate
    total_exercises = count_exercises_for_topic(topic_id)
    completed = len([s for s in exercise_submissions if s.passed])
    exercise_score = completed / total_exercises if total_exercises > 0 else 0.0
    
    mastery = (quiz_score * 0.6) + (exercise_score * 0.4)
    return mastery * 100  # Return as percentage
```

## 6. LLM Prompts

### Content Segmentation
```python
SEGMENTATION_PROMPT = """
Analyze this video transcript and segment it into distinct learning topics.

Transcript:
{transcript}

For each topic, provide:
1. Title (concise, descriptive)
2. Description (2-3 sentences)
3. Start and end timestamps
4. Estimated learning time
5. Key concepts covered

Return as JSON array.
"""
```

### Quiz Generation
```python
QUIZ_PROMPT = """
Generate a {difficulty} level quiz for this topic.

Topic: {topic_title}
Description: {topic_description}
Content: {topic_content}

Create 5 multiple-choice questions with:
- Clear, unambiguous question text
- 4 options (A, B, C, D)
- One correct answer
- Brief explanation of the correct answer

Return as JSON array.
"""
```

### Exercise Generation
```python
EXERCISE_PROMPT = """
Create a coding exercise for this programming topic.

Topic: {topic_title}
Description: {topic_description}
Language: {language}

Provide:
1. Exercise title
2. Problem description
3. Starter code template
4. 3-5 test cases (input/output pairs)
5. Difficulty level

Return as JSON.
"""
```

## 7. Database Schema

### Key Tables

```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Courses
CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    youtube_url VARCHAR(500) NOT NULL,
    youtube_video_id VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'PROCESSING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_courses_status ON courses(status);

-- Topics
CREATE TABLE topics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID REFERENCES courses(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    order_index INTEGER NOT NULL,
    start_timestamp INTEGER,
    end_timestamp INTEGER
);

CREATE INDEX idx_topics_course ON topics(course_id, order_index);

-- Concept embeddings (pgvector)
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE concepts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    topic_id UUID REFERENCES topics(id) ON DELETE CASCADE,
    embedding vector(1536)
);

CREATE INDEX idx_concepts_embedding ON concepts 
USING hnsw (embedding vector_cosine_ops);

-- User progress
CREATE TABLE user_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    course_id UUID REFERENCES courses(id),
    completed_topics UUID[],
    overall_progress_percentage FLOAT
);

-- Quiz submissions
CREATE TABLE quiz_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    quiz_id UUID REFERENCES quizzes(id),
    answers INTEGER[],
    score FLOAT,
    submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_quiz_submissions_user ON quiz_submissions(user_id, submitted_at);
```

## 8. Backend Structure

```
backend/
├── app/
│   ├── main.py                 # FastAPI app
│   ├── config.py               # Settings
│   ├── database.py             # DB connection
│   │
│   ├── models/                 # SQLAlchemy models
│   │   ├── user.py
│   │   ├── course.py
│   │   ├── quiz.py
│   │   └── progress.py
│   │
│   ├── schemas/                # Pydantic schemas
│   │   ├── auth.py
│   │   ├── course.py
│   │   └── intelligence.py
│   │
│   ├── api/                    # API routes
│   │   ├── auth.py
│   │   ├── courses.py
│   │   ├── progress.py
│   │   └── intelligence.py
│   │
│   ├── services/               # Business logic
│   │   ├── auth_service.py
│   │   ├── course_service.py
│   │   ├── llm_service.py
│   │   ├── embedding_service.py
│   │   ├── intelligence_service.py
│   │   └── progress_service.py
│   │
│   └── utils/                  # Utilities
│       ├── youtube.py
│       ├── security.py
│       └── cache.py
│
├── tests/
├── requirements.txt
└── Dockerfile
```

## 9. Frontend Structure

```
frontend/
├── src/
│   ├── components/
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   └── RegisterForm.tsx
│   │   ├── course/
│   │   │   ├── CourseCard.tsx
│   │   │   ├── CourseList.tsx
│   │   │   └── VideoProcessor.tsx
│   │   ├── learning/
│   │   │   ├── TopicView.tsx
│   │   │   ├── QuizView.tsx
│   │   │   └── ExerciseView.tsx
│   │   └── intelligence/
│   │       ├── ConceptMap.tsx
│   │       ├── WeaknessReport.tsx
│   │       └── ProgressDashboard.tsx
│   │
│   ├── pages/
│   │   ├── Home.tsx
│   │   ├── CourseDetail.tsx
│   │   ├── Learning.tsx
│   │   └── Dashboard.tsx
│   │
│   ├── services/
│   │   ├── api.ts              # Axios client
│   │   ├── authService.ts
│   │   ├── courseService.ts
│   │   └── intelligenceService.ts
│   │
│   ├── store/                  # Redux
│   │   ├── authSlice.ts
│   │   ├── courseSlice.ts
│   │   └── progressSlice.ts
│   │
│   └── types/
│       ├── course.ts
│       └── intelligence.ts
│
├── package.json
└── Dockerfile
```

## 10. Security Essentials

### Authentication
```python
# JWT with short expiration
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7

# Password hashing
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

### Input Validation
```python
from pydantic import BaseModel, EmailStr, validator

class UserRegister(BaseModel):
    email: EmailStr
    password: str
    full_name: str
    
    @validator('password')
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        return v
```

### Rate Limiting
```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/auth/login")
@limiter.limit("5/minute")
async def login(request: Request):
    pass
```

## 11. Caching Strategy

```python
# Cache course data
async def get_course(course_id: UUID):
    cache_key = f"course:{course_id}"
    
    # Try cache
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Fetch from DB
    course = await db.query(Course).filter(Course.id == course_id).first()
    
    # Cache for 1 hour
    await redis.setex(cache_key, 3600, course.json())
    return course

# Cache embeddings
async def get_concept_embedding(concept_id: UUID):
    cache_key = f"embedding:{concept_id}"
    
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    embedding = await db.query(Concept).filter(Concept.id == concept_id).first()
    
    # Cache for 24 hours
    await redis.setex(cache_key, 86400, json.dumps(embedding.embedding_vector))
    return embedding.embedding_vector
```

## 12. Error Handling

```python
from fastapi import HTTPException

class AppException(Exception):
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

@app.exception_handler(AppException)
async def app_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail}
    )

# Usage
if not course:
    raise AppException(404, "Course not found")
```

## 13. Testing Approach

### Unit Tests
```python
def test_calculate_mastery():
    quiz_submissions = [
        QuizSubmission(score=0.8),
        QuizSubmission(score=0.9)
    ]
    exercise_submissions = []
    
    mastery = calculate_topic_mastery(
        user_id, topic_id,
        quiz_submissions, exercise_submissions
    )
    
    assert mastery == 54.0  # 0.9 * 0.6 * 100
```

### Integration Tests
```python
def test_course_creation_flow():
    # Register
    response = client.post("/api/auth/register", json={...})
    assert response.status_code == 201
    
    # Login
    response = client.post("/api/auth/login", json={...})
    token = response.json()["access_token"]
    
    # Process video
    response = client.post(
        "/api/courses/process",
        json={"youtube_url": "..."},
        headers={"Authorization": f"Bearer {token}"}
    )
    assert response.status_code == 202
```

## 14. Deployment (Hackathon)

### Docker Compose
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/skillweave
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - db
      - redis

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8000

  db:
    image: pgvector/pgvector:pg15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=skillweave
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### Environment Variables
```bash
# .env
DATABASE_URL=postgresql://user:pass@localhost:5432/skillweave
REDIS_URL=redis://localhost:6379
OPENAI_API_KEY=sk-...
SECRET_KEY=your-secret-key-here
YOUTUBE_API_KEY=your-youtube-api-key
```

## 15. MVP Feature Priority

### Phase 1 (Core - Day 1-2)
1. User auth (register, login)
2. YouTube video processing
3. Basic course structure (topics, outline)
4. Simple quiz generation
5. Progress tracking

### Phase 2 (Intelligence - Day 3-4)
1. Concept extraction and embeddings
2. Weakness detection
3. Adaptive quiz generation
4. Basic concept map visualization

### Phase 3 (Polish - Day 5)
1. UI improvements
2. Error handling
3. Demo data
4. Presentation prep

---

**Document Version**: 1.0 (Hackathon Edition)  
**Focus**: Build fast, demo well, win hackathon
