Below is a complete upload-ready starter project. It includes the FastAPI API, PostgreSQL, and a React dashboard. It records AI responses submitted to the API; it **does not automatically query AI providers**.  
  
```text  
ops-visibility/  
**├──** .env.example  
**├──** .gitignore  
**├──** docker-compose.yml  
**├──** README.md  
**├──** backend/  
│   **├──** Dockerfile  
│   **├──** requirements.txt  
│   **└──** app/  
│       **├──** db.py  
│       **├──** main.py  
│       **├──** models.py  
│       **├──** schemas.py  
│       **└──** services/  
│           **└──** reporting.py  
**└──** frontend/  
    **├──** Dockerfile  
    **├──** index.html  
    **├──** package.json  
    **├──** tsconfig.json  
    **├──** vite.config.ts  
    **└──** src/  
        **├──** App.tsx  
        **├──** main.tsx  
        **└──** styles.css  
```  
  
**`.env.example`**  
  
```dotenv name=.env.example  
POSTGRES_USER=postgres  
POSTGRES_PASSWORD=change-me-for-local-use  
POSTGRES_DB=ai_visibility  
DATABASE_URL=postgresql+psycopg://postgres:change-me-for-local-use@db:5432/ai_visibility  
```  
  
**`.gitignore`**  
  
```gitignore name=.gitignore  
.env  
.venv/  
venv/  
__pycache__/  
*.py[cod]  
.pytest_cache/  
.coverage  
htmlcov/  
node_modules/  
dist/  
.DS_Store  
```  
  
**`docker-compose.yml`**  
  
```yaml name=docker-compose.yml  
services:  
  db:  
    image: postgres:16  
    restart: unless-stopped  
    environment:  
      POSTGRES_USER: ${POSTGRES_USER:-postgres}  
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-change-me-for-local-use}  
      POSTGRES_DB: ${POSTGRES_DB:-ai_visibility}  
    ports:  
      - "5432:5432"  
    volumes:  
      - postgres_data:/var/lib/postgresql/data  
    healthcheck:  
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-ai_visibility}"]  
      interval: 5s  
      timeout: 5s  
      retries: 10  
  
  api:  
    build: ./backend  
    restart: unless-stopped  
    environment:  
      DATABASE_URL: ${DATABASE_URL:-postgresql+psycopg://postgres:change-me-for-local-use@db:5432/ai_visibility}  
    depends_on:  
      db:  
        condition: service_healthy  
    ports:  
      - "8000:8000"  
  
  frontend:  
    build: ./frontend  
    restart: unless-stopped  
    environment:  
      VITE_API_PROXY_TARGET: http://api:8000  
    depends_on:  
      - api  
    ports:  
      - "5173:5173"  
  
volumes:  
  postgres_data:  
```  
  
**`README.md`**  
  
````markdown name=README.md  
# OPS Visibility  
  
A self-hostable starter dashboard and API for tracking brand mentions in AI-generated responses.  
  
## Features  
  
- Store brands, prompts, and submitted AI responses  
- Detect mentions of tracked brands in response text  
- Record a simple, rule-based sentiment label  
- Review mention counts and sentiment confidence in a dashboard  
- Run locally with Docker Compose  
  
## Requirements  
  
- Docker and Docker Compose  
  
## Run locally  
  
1. Copy `.env.example` to `.env`.  
2. Change `POSTGRES_PASSWORD` and the password in `DATABASE_URL` to the same value.  
3. Start the services:  
  
   ```bash  
   docker compose up --build  
   ```  
  
4. Open:  
   - Dashboard: <http://localhost:5173>  
   - API documentation: <http://localhost:8000/docs>  
   - Health check: <http://localhost:8000/health>  
  
The API creates the database tables on startup. For a real deployment, use a unique secret password and put the services behind appropriate network and access controls.  
  
## Example: create a brand and prompt  
  
```bash  
curl -X POST http://localhost:8000/brands \  
  -H "Content-Type: application/json" \  
  -d '{"name":"Acme AI","website":"https://acme.example"}'  
```  
  
```bash  
curl -X POST http://localhost:8000/prompts \  
  -H "Content-Type: application/json" \  
  -d '{"text":"What are useful AI tools for marketing teams?"}'  
```  
  
## Example: submit an AI response  
  
Replace the IDs with the IDs returned by the API.  
  
```bash  
curl -X POST http://localhost:8000/results \  
  -H "Content-Type: application/json" \  
  -d '{  
    "brand_id": 1,  
    "prompt_id": 1,  
    "model_name": "example-model",  
    "response_text": "Acme AI is a useful option for marketing teams.",  
    "source_urls": ["https://example.com/article"]  
  }'  
```  
  
Submitting a result scans its response for all tracked brand names and records detected mentions. `brand_id` identifies the brand the result was collected for; mentions are associated with whichever tracked brand names occur in the response.  
  
## API endpoints  
  
- `GET /health`  
- `GET /brands` and `POST /brands`  
- `GET /prompts` and `POST /prompts`  
- `POST /results`  
- `GET /reports`  
  
## Notes and limitations  
  
- This project stores responses provided to the API. It does not collect responses from ChatGPT, Claude, Gemini, Perplexity, or other providers.  
- Sentiment analysis is a small word-list heuristic, not a trained sentiment model. Treat it as a rough signal.  
- The report currently shows all-time aggregates. Historical charts, provider integrations, authentication, and database migrations are not included in this starter.  
- Source URLs are stored with each result, but source-domain analytics are not yet implemented.  
  
## Development  
  
The frontend is a Vite React app. Docker Compose starts it in development mode. To run it outside Docker, set up Node.js, then run:  
  
```bash  
cd frontend  
npm install  
npm run dev  
```  
````  
  
**`backend/requirements.txt`**  
  
```text name=backend/requirements.txt  
fastapi>=0.115,<1.0  
uvicorn[standard]>=0.30,<1.0  
sqlalchemy>=2.0,<3.0  
psycopg[binary]>=3.2,<4.0  
pydantic>=2.8,<3.0  
```  
  
**`backend/Dockerfile`**  
  
```dockerfile name=backend/Dockerfile  
FROM python:3.12-slim  
  
WORKDIR /app  
  
COPY requirements.txt .  
RUN pip install --no-cache-dir -r requirements.txt  
  
COPY app ./app  
  
EXPOSE 8000  
  
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]  
```  
  
**`backend/app/db.py`**  
  
```python name=backend/app/db.py  
import os  
from collections.abc import Generator  
  
from sqlalchemy import create_engine  
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker  
  
  
DATABASE_URL = os.getenv(  
    "DATABASE_URL",  
    "postgresql+psycopg://postgres:postgres@localhost:5432/ai_visibility",  
)  
  
engine = create_engine(DATABASE_URL, pool_pre_ping=True)  
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)  
  
  
class Base(DeclarativeBase):  
    pass  
  
  
def get_db() -> Generator[Session, None, None]:  
    db = SessionLocal()  
    try:  
        yield db  
    finally:  
        db.close()  
```  
  
**`backend/app/models.py`**  
  
```python name=backend/app/models.py  
from datetime import datetime  
from typing import Optional  
  
from sqlalchemy import DateTime, Float, ForeignKey, Integer, JSON, String, Text, func  
from sqlalchemy.orm import Mapped, mapped_column, relationship  
  
from .db import Base  
  
  
class Brand(Base):  
    __tablename__ = "brands"  
  
    id: Mapped[int] = mapped_column(Integer, primary_key=True)  
    name: Mapped[str] = mapped_column(String(255), unique=True, index=True)  
    website: Mapped[Optional[str]] = mapped_column(String(500), nullable=True)  
    created_at: Mapped[datetime] = mapped_column(  
        DateTime(timezone=True), server_default=func.now()  
    )  
  
    results: Mapped[list["AIResult"]] = relationship(back_populates="brand")  
    mentions: Mapped[list["Mention"]] = relationship(back_populates="brand")  
  
  
class Prompt(Base):  
    __tablename__ = "prompts"  
  
    id: Mapped[int] = mapped_column(Integer, primary_key=True)  
    text: Mapped[str] = mapped_column(Text)  
    created_at: Mapped[datetime] = mapped_column(  
        DateTime(timezone=True), server_default=func.now()  
    )  
  
    results: Mapped[list["AIResult"]] = relationship(back_populates="prompt")  
  
  
class AIResult(Base):  
    __tablename__ = "ai_results"  
  
    id: Mapped[int] = mapped_column(Integer, primary_key=True)  
    prompt_id: Mapped[int] = mapped_column(ForeignKey("prompts.id"), index=True)  
    brand_id: Mapped[int] = mapped_column(ForeignKey("brands.id"), index=True)  
    model_name: Mapped[str] = mapped_column(String(255))  
    response_text: Mapped[str] = mapped_column(Text)  
    source_urls: Mapped[Optional[list[str]]] = mapped_column(JSON, nullable=True)  
    created_at: Mapped[datetime] = mapped_column(  
        DateTime(timezone=True), server_default=func.now(), index=True  
    )  
  
    prompt: Mapped["Prompt"] = relationship(back_populates="results")  
    brand: Mapped["Brand"] = relationship(back_populates="results")  
    mentions: Mapped[list["Mention"]] = relationship(  
        back_populates="result", cascade="all, delete-orphan"  
    )  
  
  
class Mention(Base):  
    __tablename__ = "mentions"  
  
    id: Mapped[int] = mapped_column(Integer, primary_key=True)  
    result_id: Mapped[int] = mapped_column(ForeignKey("ai_results.id"), index=True)  
    brand_id: Mapped[int] = mapped_column(ForeignKey("brands.id"), index=True)  
    mention_text: Mapped[str] = mapped_column(String(255))  
    sentiment: Mapped[str] = mapped_column(String(50), default="neutral")  
    score: Mapped[float] = mapped_column(Float, default=0.5)  
    created_at: Mapped[datetime] = mapped_column(  
        DateTime(timezone=True), server_default=func.now()  
    )  
  
    result: Mapped["AIResult"] = relationship(back_populates="mentions")  
    brand: Mapped["Brand"] = relationship(back_populates="mentions")  
```  
  
**`backend/app/schemas.py`**  
  
```python name=backend/app/schemas.py  
from pydantic import BaseModel, ConfigDict, Field, HttpUrl  
  
  
class BrandCreate(BaseModel):  
    name: str = Field(min_length=1, max_length=255)  
    website: HttpUrl | None = None  
  
  
class BrandRead(BaseModel):  
    model_config = ConfigDict(from_attributes=True)  
  
    id: int  
    name: str  
    website: str | None = None  
  
  
class PromptCreate(BaseModel):  
    text: str = Field(min_length=1)  
  
  
class PromptRead(BaseModel):  
    model_config = ConfigDict(from_attributes=True)  
  
    id: int  
    text: str  
  
  
class AIResultCreate(BaseModel):  
    prompt_id: int  
    brand_id: int  
    model_name: str = Field(min_length=1, max_length=255)  
    response_text: str = Field(min_length=1)  
    source_urls: list[HttpUrl] | None = None  
  
  
class AIResultRead(BaseModel):  
    model_config = ConfigDict(from_attributes=True)  
  
    id: int  
    prompt_id: int  
    brand_id: int  
    model_name: str  
    response_text: str  
    source_urls: list[str] | None = None  
  
  
class MentionRead(BaseModel):  
    model_config = ConfigDict(from_attributes=True)  
  
    id: int  
    result_id: int  
    brand_id: int  
    mention_text: str  
    sentiment: str  
    score: float  
```  
  
**`backend/app/services/reporting.py`**  
  
```python name=backend/app/services/reporting.py  
import re  
  
  
POSITIVE_WORDS = {  
    "best",  
    "effective",  
    "excellent",  
    "good",  
    "great",  
    "helpful",  
    "innovative",  
    "popular",  
    "reliable",  
    "recommended",  
    "strong",  
    "useful",  
}  
  
NEGATIVE_WORDS = {  
    "bad",  
    "confusing",  
    "difficult",  
    "expensive",  
    "limited",  
    "poor",  
    "risky",  
    "slow",  
    "unreliable",  
    "weak",  
}  
  
  
def analyze_sentiment(text: str) -> dict[str, float | str]:  
    """Return a rough, lexicon-based sentiment label and confidence.  
  
    This is only a starter heuristic. The score is confidence in the selected  
    label, not a measure of how positive or negative the text is.  
    """  
    words = set(re.findall(r"\b[\w'-]+\b", text.lower()))  
    positive_hits = len(words & POSITIVE_WORDS)  
    negative_hits = len(words & NEGATIVE_WORDS)  
  
    if positive_hits > negative_hits:  
        sentiment = "positive"  
        confidence = min(0.95, 0.55 + 0.1 * (positive_hits - negative_hits))  
    elif negative_hits > positive_hits:  
        sentiment = "negative"  
        confidence = min(0.95, 0.55 + 0.1 * (negative_hits - positive_hits))  
    else:  
        sentiment = "neutral"  
        confidence = 0.5  
  
    return {"sentiment": sentiment, "score": round(confidence, 2)}  
  
  
def extract_mentions(  
    response_text: str, brands: list[tuple[int, str]]  
) -> list[dict[str, int | str | float]]:  
    """Find each tracked brand in a response and classify the response sentiment."""  
    sentiment_data = analyze_sentiment(response_text)  
    mentions: list[dict[str, int | str | float]] = []  
  
    for brand_id, brand_name in brands:  
        pattern = re.compile(rf"(?<!\w){re.escape(brand_name)}(?!\w)", re.IGNORECASE)  
        match = pattern.search(response_text)  
        if match:  
            mentions.append(  
                {  
                    "brand_id": brand_id,  
                    "mention_text": match.group(0),  
                    "sentiment": sentiment_data["sentiment"],  
                    "score": sentiment_data["score"],  
                }  
            )  
  
    return mentions  
```  
  
**`backend/app/main.py`**  
  
```python name=backend/app/main.py  
from contextlib import asynccontextmanager  
  
from fastapi import Depends, FastAPI, HTTPException, status  
from sqlalchemy import func  
from sqlalchemy.orm import Session  
  
from .db import Base, engine, get_db  
from .models import AIResult, Brand, Mention, Prompt  
from .schemas import (  
    AIResultCreate,  
    AIResultRead,  
    BrandCreate,  
    BrandRead,  
    PromptCreate,  
    PromptRead,  
)  
from .services.reporting import extract_mentions  
  
  
@asynccontextmanager  
async def lifespan(_: FastAPI):  
    Base.metadata.create_all(bind=engine)  
    yield  
  
  
app = FastAPI(title="OPS Visibility API", lifespan=lifespan)  
  
  
@app.get("/health")  
def health() -> dict[str, str]:  
    return {"status": "ok"}  
  
  
@app.post("/brands", response_model=BrandRead, status_code=status.HTTP_201_CREATED)  
def create_brand(payload: BrandCreate, db: Session = Depends(get_db)):  
    name = payload.name.strip()  
    if not name:  
        raise HTTPException(status_code=422, detail="Brand name cannot be blank")  
  
    existing = db.query(Brand).filter(func.lower(Brand.name) == name.lower()).first()  
    if existing:  
        raise HTTPException(status_code=409, detail="Brand already exists")  
  
    brand = Brand(name=name, website=str(payload.website) if payload.website else None)  
    db.add(brand)  
    db.commit()  
    db.refresh(brand)  
    return brand  
  
  
@app.get("/brands", response_model=list[BrandRead])  
def list_brands(db: Session = Depends(get_db)):  
    return db.query(Brand).order_by(Brand.name).all()  
  
  
@app.post("/prompts", response_model=PromptRead, status_code=status.HTTP_201_CREATED)  
def create_prompt(payload: PromptCreate, db: Session = Depends(get_db)):  
    prompt = Prompt(text=payload.text.strip())  
    db.add(prompt)  
    db.commit()  
    db.refresh(prompt)  
    return prompt  
  
  
@app.get("/prompts", response_model=list[PromptRead])  
def list_prompts(db: Session = Depends(get_db)):  
    return db.query(Prompt).order_by(Prompt.id.desc()).all()  
  
  
@app.post("/results", response_model=AIResultRead, status_code=status.HTTP_201_CREATED)  
def create_result(payload: AIResultCreate, db: Session = Depends(get_db)):  
    prompt = db.get(Prompt, payload.prompt_id)  
    if prompt is None:  
        raise HTTPException(status_code=404, detail="Prompt not found")  
  
    brand = db.get(Brand, payload.brand_id)  
    if brand is None:  
        raise HTTPException(status_code=404, detail="Brand not found")  
  
    result = AIResult(  
        prompt_id=payload.prompt_id,  
        brand_id=payload.brand_id,  
        model_name=payload.model_name.strip(),  
        response_text=payload.response_text,  
        source_urls=[str(url) for url in payload.source_urls]  
        if payload.source_urls  
        else [],  
    )  
    db.add(result)  
    db.flush()  
  
    tracked_brands = [(row.id, row.name) for row in db.query(Brand).all()]  
    for detected in extract_mentions(payload.response_text, tracked_brands):  
        db.add(  
            Mention(  
                result_id=result.id,  
                brand_id=int(detected["brand_id"]),  
                mention_text=str(detected["mention_text"]),  
                sentiment=str(detected["sentiment"]),  
                score=float(detected["score"]),  
            )  
        )  
  
    db.commit()  
    db.refresh(result)  
    return result  
  
  
@app.get("/reports")  
def build_report(db: Session = Depends(get_db)) -> dict:  
    rows = (  
        db.query(  
            Brand.id,  
            Brand.name,  
            func.count(Mention.id).label("mention_count"),  
            func.avg(Mention.score).label("avg_score"),  
        )  
        .outerjoin(Mention, Mention.brand_id == Brand.id)  
        .group_by(Brand.id, Brand.name)  
        .order_by(func.count(Mention.id).desc(), Brand.name)  
        .all()  
    )  
  
    brands = [  
        {  
            "brand_id": row.id,  
            "brand_name": row.name,  
            "mention_count": int(row.mention_count or 0),  
            "avg_score": round(float(row.avg_score or 0), 2),  
        }  
        for row in rows  
    ]  
  
    return {  
        "brands": brands,  
        "total_brands": len(brands),  
        "total_mentions": sum(item["mention_count"] for item in brands),  
    }  
```  
  
**`frontend/package.json`**  
  
```json name=frontend/package.json  
{  
  "name": "ops-visibility-dashboard",  
  "private": true,  
  "version": "0.1.0",  
  "type": "module",  
  "scripts": {  
    "dev": "vite --host 0.0.0.0",  
    "build": "tsc --noEmit && vite build",  
    "preview": "vite preview --host 0.0.0.0"  
  },  
  "dependencies": {  
    "@vitejs/plugin-react": "^4.3.1",  
    "vite": "^5.4.8",  
    "typescript": "^5.6.2",  
    "react": "^18.3.1",  
    "react-dom": "^18.3.1"  
  },  
  "devDependencies": {  
    "@types/react": "^18.3.5",  
    "@types/react-dom": "^18.3.0"  
  }  
}  
```  
  
**`frontend/Dockerfile`**  
  
```dockerfile name=frontend/Dockerfile  
FROM node:20-alpine  
  
WORKDIR /app  
  
COPY package.json ./  
RUN npm install  
  
COPY . .  
  
EXPOSE 5173  
  
CMD ["npm", "run", "dev"]  
```  
  
**`frontend/index.html`**  
  
```html name=frontend/index.html  
<!doctype html>  
<html lang="en">  
  <head>  
    <meta charset="UTF-8" />  
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />  
    <meta name="description" content="Monitor brand mentions in AI responses." />  
    <title>OPS Visibility</title>  
  </head>  
  <body>  
    <div id="root"></div>  
    <script type="module" src="/src/main.tsx"></script>  
  </body>  
</html>  
```  
  
**`frontend/vite.config.ts`**  
  
```typescript name=frontend/vite.config.ts  
import { defineConfig } from "vite";  
import react from "@vitejs/plugin-react";  
  
export default defineConfig({  
  plugins: [react()],  
  server: {  
    host: "0.0.0.0",  
    proxy: {  
      "/api": {  
        target: process.env.VITE_API_PROXY_TARGET ?? "http://localhost:8000",  
        changeOrigin: true,  
        rewrite: (path) => path.replace(/^\/api/, ""),  
      },  
    },  
  },  
});  
```  
  
**`frontend/tsconfig.json`**  
  
```json name=frontend/tsconfig.json  
{  
  "compilerOptions": {  
    "target": "ES2020",  
    "useDefineForClassFields": true,  
    "lib": ["ES2020", "DOM", "DOM.Iterable"],  
    "module": "ESNext",  
    "skipLibCheck": true,  
    "moduleResolution": "Bundler",  
    "resolveJsonModule": true,  
    "isolatedModules": true,  
    "noEmit": true,  
    "jsx": "react-jsx",  
    "strict": true  
  },  
  "include": ["src"]  
}  
```  
  
**`frontend/src/main.tsx`**  
  
```tsx name=frontend/src/main.tsx  
import React from "react";  
import ReactDOM from "react-dom/client";  
import App from "./App";  
import "./styles.css";  
  
ReactDOM.createRoot(document.getElementById("root")!).render(  
  <React.StrictMode>  
    <App />  
  </React.StrictMode>,  
);  
```  
  
**`frontend/src/App.tsx`**  
  
```tsx name=frontend/src/App.tsx  
import { useCallback, useEffect, useMemo, useState } from "react";  
  
type BrandReport = {  
  brand_id: number;  
  brand_name: string;  
  mention_count: number;  
  avg_score: number;  
};  
  
type Report = {  
  brands: BrandReport[];  
  total_brands: number;  
  total_mentions: number;  
};  
  
const API_BASE = "/api";  
  
function App() {  
  const [report, setReport] = useState<Report | null>(null);  
  const [loading, setLoading] = useState(true);  
  const [error, setError] = useState<string | null>(null);  
  const [lastUpdated, setLastUpdated] = useState<Date | null>(null);  
  
  const loadReport = useCallback(async () => {  
    setLoading(true);  
    setError(null);  
  
    try {  
      const response = await fetch(`${API_BASE}/reports`);  
      if (!response.ok) {  
        throw new Error(`Report API returned HTTP ${response.status}.`);  
      }  
  
      const data: Report = await response.json();  
      if (!Array.isArray(data.brands)) {  
        throw new Error("Report API returned an unexpected response.");  
      }  
  
      setReport(data);  
      setLastUpdated(new Date());  
    } catch (err) {  
      setError(  
        err instanceof Error ? err.message : "Could not load the report.",  
      );  
    } finally {  
      setLoading(false);  
    }  
  }, []);  
  
  useEffect(() => {  
    void loadReport();  
  }, [loadReport]);  
  
  const brands = useMemo(  
    () =>  
      [...(report?.brands ?? [])].sort(  
        (a, b) => b.mention_count - a.mention_count,  
      ),  
    [report],  
  );  
  
  const mentionWeightedScore = useMemo(() => {  
    if (!brands.length) return 0;  
    const total = brands.reduce((sum, brand) => sum + brand.mention_count, 0);  
    if (!total) return 0;  
  
    return (  
      brands.reduce(  
        (sum, brand) => sum + brand.avg_score * brand.mention_count,  
        0,  
      ) / total  
    );  
  }, [brands]);  
  
  const maxMentions = Math.max(1, ...brands.map((brand) => brand.mention_count));  
  
  return (  
    <main className="layout">  
      <aside className="sidebar">  
        <a className="logo" href="#" aria-label="OPS Visibility dashboard">  
          <span className="logo-mark">O</span>  
          <span>  
            <strong>OPS Visibility</strong>  
            <small>AI mention reporting</small>  
          </span>  
        </a>  
        <div className="nav-label">WORKSPACE</div>  
        <a className="nav-link active" href="#overview">  
          <span aria-hidden="true">**▦**</span> Overview  
        </a>  
        <a className="nav-link" href="#brands">  
          <span aria-hidden="true">**◉**</span> Brand rankings  
        </a>  
        <div className="sidebar-bottom">  
          <span className="status-dot" /> Report API dashboard  
        </div>  
      </aside>  
  
      <section className="content" id="overview">  
        <header className="topbar">  
          <span>Workspace / Overview</span>  
          <button onClick={() => void loadReport()} disabled={loading}>  
            {loading ? "Loading…" : "↻ Refresh"}  
          </button>  
        </header>  
  
        <div className="heading">  
          <div>  
            <p className="eyebrow">AI SEARCH ANALYTICS</p>  
            <h1>Visibility overview</h1>  
            <p className="muted">  
              Brand mentions found in responses submitted to the API.  
            </p>  
          </div>  
          <span className="updated">  
            {lastUpdated  
              ? `Updated ${lastUpdated.toLocaleTimeString([], {  
                  hour: "2-digit",  
                  minute: "2-digit",  
                })}`  
              : "Waiting for data"}  
          </span>  
        </div>  
  
        {error && (  
          <div className="error" role="alert">  
            <div>  
              <strong>Couldn’t load the report</strong>  
              <p>{error} Check that the API service is running.</p>  
            </div>  
            <button className="retry" onClick={() => void loadReport()}>  
              Try again  
            </button>  
          </div>  
        )}  
  
        <section className="metrics" aria-label="Report metrics">  
          <article className="card">  
            <span className="metric-label">Total mentions</span>  
            <strong className="metric-value">  
              {loading && !report ? "—" : (report?.total_mentions ?? 0)}  
            </strong>  
            <span className="metric-note">Across tracked brands</span>  
          </article>  
          <article className="card">  
            <span className="metric-label">Tracked brands</span>  
            <strong className="metric-value">  
              {loading && !report ? "—" : (report?.total_brands ?? 0)}  
            </strong>  
            <span className="metric-note">Currently in the report</span>  
          </article>  
          <article className="card">  
            <span className="metric-label">Avg. sentiment confidence</span>  
            <strong className="metric-value">  
              {loading && !report  
                ? "—"  
                : `${Math.round(mentionWeightedScore * 100)}%`}  
            </strong>  
            <span className="metric-note">Heuristic confidence, weighted by mentions</span>  
          </article>  
        </section>  
  
        <section className="panel" id="brands">  
          <div className="panel-heading">  
            <div>  
              <h2>Brand visibility</h2>  
              <p>All-time mentions detected in submitted responses.</p>  
            </div>  
            <span className="tag">All time</span>  
          </div>  
  
          {loading && !report ? (  
            <div className="empty" aria-live="polite">Loading report…</div>  
          ) : brands.length === 0 ? (  
            <div className="empty">  
              <strong>No brand data yet</strong>  
              <p>Add a brand, then submit AI response results through the API.</p>  
            </div>  
          ) : (  
            <div className="table-scroll">  
              <table>  
                <thead>  
                  <tr>  
                    <th>Brand</th>  
                    <th>Mentions</th>  
                    <th>Share of mentions</th>  
                    <th>Sentiment confidence</th>  
                  </tr>  
                </thead>  
                <tbody>  
                  {brands.map((brand, index) => {  
                    const share =  
                      report && report.total_mentions  
                        ? (brand.mention_count / report.total_mentions) * 100  
                        : 0;  
  
                    return (  
                      <tr key={brand.brand_id}>  
                        <td>  
                          <div className="brand-name">  
                            <span className="rank">  
                              {String(index + 1).padStart(2, "0")}  
                            </span>  
                            <strong>{brand.brand_name}</strong>  
                          </div>  
                        </td>  
                        <td className="count">{brand.mention_count}</td>  
                        <td>  
                          <div className="share">  
                            <div  
                              className="bar"  
                              role="img"  
                              aria-label={`${brand.brand_name}: ${share.toFixed(1)}% of mentions`}  
                            >  
                              <span  
                                style={{  
                                  width: `${(brand.mention_count / maxMentions) * 100}%`,  
                                }}  
                              />  
                            </div>  
                            <small>{share.toFixed(1)}%</small>  
                          </div>  
                        </td>  
                        <td>  
                          <span className="score">  
                            {Math.round(brand.avg_score * 100)}%  
                          </span>  
                        </td>  
                      </tr>  
                    );  
                  })}  
                </tbody>  
              </table>  
            </div>  
          )}  
        </section>  
  
        <footer>  
          <span>OPS Visibility</span>  
          <span>Sentiment is estimated using a basic word-list heuristic.</span>  
        </footer>  
      </section>  
    </main>  
  );  
}  
  
export default App;  
```  
  
**`frontend/src/styles.css`**  
  
```css name=frontend/src/styles.css  
@import url("https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=Manrope:wght@500;600;700;800&display=swap");  
  
:root {  
  font-family: "DM Sans", sans-serif;  
  color: #24243a;  
  background: #f7f8fc;  
  font-synthesis: none;  
  text-rendering: optimizeLegibility;  
  -webkit-font-smoothing: antialiased;  
  font-size: 14px;  
}  
  
* {  
  box-sizing: border-box;  
}  
  
body {  
  min-width: 320px;  
  min-height: 100vh;  
  margin: 0;  
}  
  
button,  
a {  
  font: inherit;  
}  
  
button:focus-visible,  
a:focus-visible {  
  outline: 3px solid #aa9bff;  
  outline-offset: 3px;  
}  
  
.layout {  
  min-height: 100vh;  
}  
  
.sidebar {  
  position: fixed;  
  inset: 0 auto 0 0;  
  display: flex;  
  width: 235px;  
  flex-direction: column;  
  padding: 26px 17px 18px;  
  border-right: 1px solid #eaeaf1;  
  background: #fff;  
}  
  
.logo {  
  display: flex;  
  align-items: center;  
  gap: 11px;  
  padding: 0 7px;  
  color: inherit;  
  text-decoration: none;  
}  
  
.logo-mark {  
  display: grid;  
  width: 38px;  
  height: 38px;  
  place-items: center;  
  border-radius: 12px;  
  background: #6b55e8;  
  color: white;  
  font-family: "Manrope", sans-serif;  
  font-size: 20px;  
  font-weight: 800;  
}  
  
.logo strong,  
.logo small {  
  display: block;  
}  
  
.logo strong {  
  font-family: "Manrope", sans-serif;  
  font-size: 14px;  
}  
  
.logo small {  
  margin-top: 3px;  
  color: #9293a3;  
  font-size: 10px;  
}  
  
.nav-label {  
  margin: 46px 10px 12px;  
  color: #9c9eac;  
  font-size: 10px;  
  font-weight: 700;  
  letter-spacing: 0.1em;  
}  
  
.nav-link {  
  display: flex;  
  align-items: center;  
  gap: 11px;  
  min-height: 40px;  
  margin-bottom: 4px;  
  padding: 0 12px;  
  border-radius: 8px;  
  color: #77798b;  
  text-decoration: none;  
}  
  
.nav-link span {  
  width: 18px;  
  text-align: center;  
  font-size: 16px;  
}  
  
.nav-link.active {  
  background: #f0edff;  
  color: #5e49d4;  
  font-weight: 600;  
}  
  
.sidebar-bottom {  
  display: flex;  
  align-items: center;  
  gap: 8px;  
  margin-top: auto;  
  padding: 15px 8px 3px;  
  border-top: 1px solid #efeff4;  
  color: #858797;  
  font-size: 11px;  
}  
  
.status-dot {  
  width: 7px;  
  height: 7px;  
  border-radius: 50%;  
  background: #50bd89;  
}  
  
.content {  
  max-width: 1500px;  
  min-height: 100vh;  
  margin-left: 235px;  
  padding: 0 42px;  
}  
  
.topbar {  
  display: flex;  
  height: 70px;  
  align-items: center;  
  justify-content: space-between;  
  border-bottom: 1px solid #e9eaf1;  
  color: #898b9c;  
  font-size: 12px;  
}  
  
.topbar button {  
  min-height: 35px;  
  padding: 0 13px;  
  border: 1px solid #e4e4ed;  
  border-radius: 7px;  
  background: #fff;  
  color: #4b4c60;  
  cursor: pointer;  
  font-weight: 600;  
}  
  
.topbar button:disabled {  
  cursor: wait;  
  opacity: 0.65;  
}  
  
.heading {  
  display: flex;  
  align-items: flex-end;  
  justify-content: space-between;  
  gap: 20px;  
  padding: 34px 0 24px;  
}  
  
.eyebrow {  
  margin: 0 0 8px;  
  color: #7864e8;  
  font-size: 10px;  
  font-weight: 700;  
  letter-spacing: 0.11em;  
}  
  
h1,  
h2 {  
  font-family: "Manrope", sans-serif;  
}  
  
h1 {  
  margin: 0 0 6px;  
  font-size: clamp(24px, 3vw, 30px);  
  letter-spacing: -0.045em;  
}  
  
.muted,  
.panel-heading p {  
  margin: 0;  
  color: #888b9c;  
  font-size: 12px;  
}  
  
.updated {  
  padding-bottom: 3px;  
  color: #898b9c;  
  font-size: 11px;  
  white-space: nowrap;  
}  
  
.metrics {  
  display: grid;  
  grid-template-columns: repeat(3, minmax(0, 1fr));  
  gap: 16px;  
  margin-bottom: 20px;  
}  
  
.card,  
.panel {  
  border: 1px solid #ececf3;  
  border-radius: 10px;  
  background: #fff;  
  box-shadow: 0 2px 6px rgb(35 37 65 / 2%);  
}  
  
.card {  
  display: flex;  
  min-height: 137px;  
  flex-direction: column;  
  padding: 18px 20px;  
}  
  
.metric-label {  
  color: #777a8d;  
  font-size: 12px;  
}  
  
.metric-value {  
  margin: 12px 0 3px;  
  font-family: "Manrope", sans-serif;  
  font-size: 29px;  
  font-weight: 800;  
  letter-spacing: -0.05em;  
}  
  
.metric-note {  
  color: #a0a2b0;  
  font-size: 10px;  
}  
  
.panel {  
  overflow: hidden;  
}  
  
.panel-heading {  
  display: flex;  
  align-items: center;  
  justify-content: space-between;  
  gap: 15px;  
  padding: 19px 21px;  
  border-bottom: 1px solid #f0f0f5;  
}  
  
.panel-heading h2 {  
  margin: 0 0 4px;  
  font-size: 15px;  
}  
  
.panel-heading p {  
  font-size: 11px;  
}  
  
.tag {  
  padding: 6px 9px;  
  border: 1px solid #ededf3;  
  border-radius: 6px;  
  color: #777a8d;  
  font-size: 10px;  
}  
  
.table-scroll {  
  overflow-x: auto;  
}  
  
table {  
  width: 100%;  
  border-collapse: collapse;  
  text-align: left;  
}  
  
th {  
  padding: 12px 20px;  
  background: #fcfcfe;  
  color: #9698a7;  
  font-size: 9px;  
  font-weight: 700;  
  letter-spacing: 0.07em;  
  text-transform: uppercase;  
  white-space: nowrap;  
}  
  
td {  
  height: 61px;  
  padding: 10px 20px;  
  border-top: 1px solid #f1f1f6;  
  color: #56596d;  
  font-size: 12px;  
}  
  
.brand-name {  
  display: flex;  
  min-width: 150px;  
  align-items: center;  
  gap: 11px;  
}  
  
.brand-name strong {  
  color: #35374c;  
  font-weight: 600;  
}  
  
.rank {  
  display: grid;  
  width: 27px;  
  height: 27px;  
  place-items: center;  
  border-radius: 7px;  
  background: #f1edff;  
  color: #735de5;  
  font-size: 9px;  
  font-weight: 700;  
}  
  
.count {  
  color: #33364a;  
  font-weight: 600;  
}  
  
.share {  
  display: flex;  
  min-width: 150px;  
  align-items: center;  
  gap: 10px;  
}  
  
.bar {  
  width: min(150px, 14vw);  
  height: 6px;  
  overflow: hidden;  
  border-radius: 10px;  
  background: #f0eff7;  
}  
  
.bar span {  
  display: block;  
  height: 100%;  
  border-radius: inherit;  
  background: linear-gradient(90deg, #8b75f0, #624ce0);  
}  
  
.share small {  
  min-width: 37px;  
  color: #777a8d;  
  font-size: 10px;  
}  
  
.score {  
  display: inline-block;  
  padding: 5px 8px;  
  border-radius: 6px;  
  background: #f2f0ff;  
  color: #6552c9;  
  font-size: 10px;  
  font-weight: 600;  
}  
  
.empty {  
  display: flex;  
  min-height: 205px;  
  flex-direction: column;  
  align-items: center;  
  justify-content: center;  
  color: #85889a;  
  text-align: center;  
}  
  
.empty strong {  
  color: #414358;  
  font-family: "Manrope", sans-serif;  
  font-size: 14px;  
}  
  
.empty p {  
  max-width: 350px;  
  margin: 7px 12px 0;  
  color: #9698a7;  
  font-size: 11px;  
  line-height: 1.6;  
}  
  
.error {  
  display: flex;  
  align-items: center;  
  justify-content: space-between;  
  gap: 20px;  
  margin-bottom: 18px;  
  padding: 13px 16px;  
  border: 1px solid #f0d7d4;  
  border-radius: 8px;  
  background: #fff8f7;  
  color: #9c4740;  
}  
  
.error strong {  
  font-size: 12px;  
}  
  
.error p {  
  margin: 4px 0 0;  
  color: #a46b66;  
  font-size: 11px;  
}  
  
.retry {  
  border: 0;  
  background: transparent;  
  color: #8e3f39;  
  cursor: pointer;  
  font-size: 11px;  
  font-weight: 700;  
}  
  
footer {  
  display: flex;  
  justify-content: space-between;  
  gap: 12px;  
  padding: 18px 2px;  
  color: #a0a2b0;  
  font-size: 10px;  
}  
  
@media (max-width: 900px) {  
  .sidebar {  
    width: 190px;  
  }  
  
  .content {  
    margin-left: 190px;  
    padding: 0 24px;  
  }  
  
  .metrics {  
    gap: 10px;  
  }  
  
  .card {  
    padding-right: 14px;  
    padding-left: 14px;  
  }  
}  
  
@media (max-width: 680px) {  
  .sidebar {  
    position: static;  
    width: auto;  
    height: auto;  
    padding: 12px 16px;  
    border-right: 0;  
    border-bottom: 1px solid #ececf3;  
  }  
  
  .nav-label,  
  .nav-link,  
  .sidebar-bottom {  
    display: none;  
  }  
  
  .content {  
    margin-left: 0;  
    padding: 0 16px;  
  }  
  
  .topbar {  
    height: 55px;  
  }  
  
  .heading {  
    align-items: flex-start;  
    flex-direction: column;  
    padding: 25px 0 20px;  
  }  
  
  .metrics {  
    grid-template-columns: 1fr;  
  }  
  
  .card {  
    min-height: 115px;  
  }  
  
  footer {  
    flex-direction: column;  
  }  
  
  th,  
  td {  
    padding-right: 12px;  
    padding-left: 12px;  
  }  
}  
```  
  
To run it locally:  
  
```bash  
cp .env.example .env  
docker compose up --build  
```  
  
Then visit `http://localhost:5173`. Before publishing, replace the example database password in `.env`; keep `.env` out of Git (it’s in `.gitignore`).  
