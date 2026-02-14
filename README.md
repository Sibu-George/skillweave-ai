# SkillWeave 🎓

> Transform YouTube tutorials into interactive, mastery-based courses powered by AI

SkillWeave automatically converts YouTube educational videos into structured learning experiences with AI-generated quizzes, coding exercises, and intelligent progress tracking. Built for hackathons, optimized for learning.

## ✨ Features

### 🎬 Smart Video Processing
- Extract and analyze YouTube video transcripts
- Automatically segment content into logical topics
- Generate hierarchical course outlines
- Preserve timestamps for easy navigation

### 🧠 AI-Powered Content Generation
- Generate contextual quizzes for each topic
- Create coding exercises with test cases
- Adaptive difficulty based on performance
- Explanations for every answer

### 📊 Intelligent Learning Analytics
- Semantic concept clustering using embeddings
- Automatic weakness detection
- Personalized learning recommendations
- Visual concept maps showing knowledge relationships

### 🎯 Mastery-Based Progression
- Track progress across topics and courses
- Calculate mastery scores (quiz + exercise performance)
- Adaptive quizzes targeting weak areas
- Retry system for continuous improvement

## 🚀 Quick Start

### Prerequisites
- Docker & Docker Compose
- OpenAI API key
- YouTube Data API key (optional, for metadata)

### Installation

1. **Clone the repository**
```bash
git clone https://github.com/yourusername/skillweave.git
cd skillweave
```

2. **Set up environment variables**
```bash
cp .env.example .env
# Edit .env and add your API keys:
# OPENAI_API_KEY=sk-...
# YOUTUBE_API_KEY=your-key (optional)
```

3. **Start the application**
```bash
docker-compose up -d
```

4. **Access the application**
- Frontend: http://localhost:3000
- API: http://localhost:8000
- API Docs: http://localhost:8000/docs

### Manual Setup (Development)

**Backend:**
```bash
cd backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

**Frontend:**
```bash
cd frontend
npm install
npm start
```

**Database:**
```bash
# Install PostgreSQL with pgvector extension
# Create database
createdb skillweave

# Run migrations
alembic upgrade head
```

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│         Frontend (React)                 │
│  Course Browser | Learning Interface     │
│  Quiz System | Progress Dashboard        │
└─────────────────────────────────────────┘
                  │ REST API
┌─────────────────────────────────────────┐
│         Backend (FastAPI)                │
│  ┌──────────┐  ┌──────────┐            │
│  │  Course  │  │Intelligence│            │
│  │ Service  │  │  Service   │            │
│  └──────────┘  └──────────┘            │
│  ┌──────────┐  ┌──────────┐            │
│  │ Content  │  │ Progress │            │
│  │   Gen    │  │ Service  │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
                  │
┌─────────────────────────────────────────┐
│  PostgreSQL + pgvector | Redis | OpenAI │
└─────────────────────────────────────────┘
```

## 📖 Usage

### Creating a Course

1. **Submit a YouTube URL**
```bash
curl -X POST http://localhost:8000/api/courses/process \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"youtube_url": "https://youtube.com/watch?v=VIDEO_ID"}'
```

2. **Check processing status**
```bash
curl http://localhost:8000/api/courses/status/JOB_ID \
  -H "Authorization: Bearer YOUR_TOKEN"
```

3. **Start learning**
- Browse to the course page
- Work through topics sequentially
- Take quizzes and complete exercises
- Track your mastery progress

### Using the Intelligence Features

**Get personalized weaknesses:**
```bash
curl http://localhost:8000/api/intelligence/weaknesses \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Generate adaptive quiz:**
```bash
curl http://localhost:8000/api/intelligence/adaptive-quiz \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**View concept map:**
```bash
curl http://localhost:8000/api/intelligence/concept-map \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## 🛠️ Tech Stack

### Backend
- **FastAPI** - Modern Python web framework
- **PostgreSQL + pgvector** - Database with vector similarity search
- **Redis** - Caching layer
- **SQLAlchemy** - ORM
- **Pydantic** - Data validation
- **OpenAI API** - LLM for content generation and embeddings

### Frontend
- **React 18** - UI framework
- **TypeScript** - Type safety
- **Redux Toolkit** - State management
- **Material-UI** - Component library
- **D3.js** - Data visualization
- **Axios** - HTTP client

### ML/AI
- **OpenAI GPT-4** - Content generation
- **text-embedding-3-large** - Semantic embeddings
- **scikit-learn** - Clustering algorithms
- **NumPy** - Numerical computing

## 📁 Project Structure

```
skillweave/
├── backend/
│   ├── app/
│   │   ├── api/              # API routes
│   │   ├── models/           # Database models
│   │   ├── schemas/          # Pydantic schemas
│   │   ├── services/         # Business logic
│   │   └── utils/            # Utilities
│   ├── tests/
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/       # React components
│   │   ├── pages/            # Page components
│   │   ├── services/         # API clients
│   │   ├── store/            # Redux store
│   │   └── types/            # TypeScript types
│   └── package.json
├── docker-compose.yml
└── README.md
```

## 🧪 Testing

**Backend tests:**
```bash
cd backend
pytest
pytest --cov=app tests/  # With coverage
```

**Frontend tests:**
```bash
cd frontend
npm test
npm run test:coverage
```

## 🔑 Environment Variables

```bash
# Backend
DATABASE_URL=postgresql://user:pass@localhost:5432/skillweave
REDIS_URL=redis://localhost:6379
OPENAI_API_KEY=sk-...
SECRET_KEY=your-secret-key-here
YOUTUBE_API_KEY=your-youtube-api-key

# Frontend
REACT_APP_API_URL=http://localhost:8000
```

## 📊 API Documentation

Once the backend is running, visit:
- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

### Key Endpoints

**Authentication:**
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login and get JWT token
- `POST /api/auth/refresh` - Refresh access token

**Courses:**
- `POST /api/courses/process` - Submit YouTube URL for processing
- `GET /api/courses` - List all courses
- `GET /api/courses/{id}` - Get course details
- `GET /api/courses/{id}/outline` - Get course outline

**Learning:**
- `GET /api/courses/{id}/topics/{topic_id}/quiz` - Get quiz
- `POST /api/progress/quiz` - Submit quiz answers
- `GET /api/progress/mastery/{course_id}` - Get mastery scores

**Intelligence:**
- `GET /api/intelligence/weaknesses` - Get detected weaknesses
- `GET /api/intelligence/adaptive-quiz` - Generate adaptive quiz
- `GET /api/intelligence/concept-map` - Get concept relationships

## 🎯 Roadmap

### Phase 1: MVP ✅
- [x] User authentication
- [x] YouTube video processing
- [x] Quiz generation
- [x] Progress tracking
- [x] Basic UI

### Phase 2: Intelligence 🚧
- [x] Concept embeddings
- [x] Weakness detection
- [x] Adaptive quizzes
- [ ] Concept map visualization
- [ ] Learning recommendations

### Phase 3: Enhancement 📋
- [ ] Coding exercise execution
- [ ] Spaced repetition system
- [ ] Mobile app
- [ ] Collaborative features
- [ ] Advanced analytics

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- OpenAI for GPT-4 and embedding models
- pgvector for efficient vector similarity search
- The open-source community for amazing tools and libraries

## 📧 Contact

- **Project Link**: https://github.com/yourusername/skillweave
- **Issues**: https://github.com/yourusername/skillweave/issues
- **Discussions**: https://github.com/yourusername/skillweave/discussions

---

Built with ❤️ for learners everywhere
