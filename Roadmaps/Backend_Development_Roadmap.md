# Backend Development Roadmap for Learning Platform

## 1. Introduction

### 1.1 Project Overview
This document outlines the implementation roadmap for developing the backend of the Personalised Learning Platform. The platform is a PWA focused initially on Mathematics for secondary/A-Level students, featuring a multi-agent AI system for delivering personalized guidance, user authentication, progress tracking, multiple learning pathways, and payment integration.

### 1.2 Document Purpose
This roadmap serves as a comprehensive implementation guide, focusing on selected technologies and approaches. It provides specific implementation guidance and highlights important dev vs. prod environment differences throughout the development lifecycle.

## 2. Technology Stack Implementation

### 2.1 Core Backend Framework: Python with FastAPI
**Implementation Details:**
- Use Python 3.10+ with FastAPI and Pydantic models
- Configure type hints throughout the codebase for better IDE support and error detection
- Set up linting with flake8, black, and isort for code consistency
- Implement middleware for error handling, authentication, and request validation

**Code Example: Project Structure**
```python
# filepath: app/main.py
from fastapi import FastAPI, Request, Depends
from fastapi.middleware.cors import CORSMiddleware
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.middleware.error_handler import error_handler
from app.routes import router

app = FastAPI(title="Learning Platform API")

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configured based on environment
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(router, prefix="/api")

# Error handling
@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request: Request, exc: StarletteHTTPException):
    return error_handler(request, exc)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

**Dev vs. Prod Considerations:**
- Development uses uvicorn with hot reload enabled
- Production uses Gunicorn with uvicorn workers for better performance and reliability
- Different environment variable files for dev (.env.development) and prod (.env.production)
- Additional logging in development, structured JSON logging in production

### 2.2 Backend Architecture: Microservices with API Gateway
**Implementation Details:**
- Implement an API Gateway as the entry point for all client requests
- Develop the following core microservices:
  1. Authentication Service
  2. Learning Module Service
  3. Progress Tracking Service
  4. Payment Service
  5. Multi-Agent AI Service
  6. User Management Service

**Initial Structure:**
```
/app
  /gateway                 # API Gateway service
  /services
    /auth                  # Authentication service
    /learning              # Learning module service
    /progress              # Progress tracking service
    /payment               # Payment processing service
    /ai                    # Multi-agent AI service
    /user                  # User management service
  /shared                  # Shared libraries and types
  /utils                   # Common utilities
```

**Inter-Service Communication:**
- Implement a message broker using RabbitMQ or Kafka for event-driven communication
- Use direct HTTP/REST calls for synchronous service-to-service communication
- Define standard message formats and API contracts

**Dev vs. Prod Considerations:**
- Development uses Docker Compose to run services locally
- Production uses Kubernetes for orchestration and scaling
- Service discovery through environment variables in dev, through Kubernetes Service in prod
- Circuit breakers and fallbacks for resilient inter-service communication

### 2.3 Database Implementation

#### Firebase Firestore/Realtime Database (Initial Phase)
**Implementation Details:**
- Set up Firebase project with appropriate security rules
- Implement data models for user profiles and authentication state
- Design initial progress tracking schemas
- Set up indices for common queries

**Code Example: Firebase Setup**
```python
# filepath: app/database/firebase.py
import firebase_admin
from firebase_admin import credentials, firestore
import os

cred = credentials.Certificate({
    "type": "service_account",
    "project_id": os.environ.get("FIREBASE_PROJECT_ID"),
    "private_key_id": os.environ.get("FIREBASE_PRIVATE_KEY_ID"),
    "private_key": os.environ.get("FIREBASE_PRIVATE_KEY").replace("\\n", "\n"),
    "client_email": os.environ.get("FIREBASE_CLIENT_EMAIL"),
    "client_id": os.environ.get("FIREBASE_CLIENT_ID"),
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": os.environ.get("FIREBASE_CLIENT_CERT_URL")
})

firebase_app = firebase_admin.initialize_app(cred)
db = firestore.client()
```

#### MongoDB Implementation
**Implementation Details:**
- Set up MongoDB Atlas cluster or self-hosted MongoDB
- Design document schemas for learning content and session data
- Implement MongoDB indexes for performance optimization
- Configure connection pooling and error handling

**Code Example: MongoDB Connection**
```python
# filepath: app/database/mongodb.py
from motor.motor_asyncio import AsyncIOMotorClient
import logging
import os

logger = logging.getLogger(__name__)

async def connect_to_mongo():
    try:
        client = AsyncIOMotorClient(os.environ.get("MONGODB_URI"))
        await client.admin.command('ping')
        logger.info(f"MongoDB Connected: {client.address}")
        return client
    except Exception as e:
        logger.error(f"Error connecting to MongoDB: {str(e)}")
        raise

async def get_database():
    client = await connect_to_mongo()
    return client[os.environ.get("MONGODB_DATABASE")]
```

#### PostgreSQL Implementation
**Implementation Details:**
- Set up PostgreSQL database for structured data
- Design schema for payment records and subscription data
- Configure connection pooling with asyncpg
- Implement database migrations using Alembic

**Code Example: PostgreSQL Connection with SQLAlchemy**
```python
# filepath: app/database/postgres.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base, sessionmaker
import os

DATABASE_URL = os.environ.get("DATABASE_URL")

engine = create_async_engine(
    DATABASE_URL,
    echo=os.environ.get("DEBUG", "False").lower() == "true"
)

AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)
Base = declarative_base()

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

#### Vector Database Implementation (Pinecone)
**Implementation Details:**
- Set up Pinecone account and indexes
- Configure vector dimensions and metrics based on embedding model
- Implement indexing pipeline for curriculum content
- Create retrieval utilities for similarity search

**Code Example: Pinecone Setup**
```python
# filepath: app/database/vector_store.py
import pinecone
import os

api_key = os.environ.get("PINECONE_API_KEY")
environment = os.environ.get("PINECONE_ENVIRONMENT")
index_name = os.environ.get("PINECONE_INDEX")

pinecone.init(api_key=api_key, environment=environment)
index = pinecone.Index(index_name)

def get_vector_store():
    return index
```

**Dev vs. Prod Considerations:**
- Local databases for development (MongoDB container, local PostgreSQL)
- Read replicas and connection pooling in production
- Automated backups and disaster recovery in production
- Reduced logging verbosity in production

## 3. Multi-Agent System Implementation

### 3.1 Foundation Setup

**Implementation Details:**
- Set up LangChain, LangGraph, and LangMem
- Create model provider abstraction layer
- Implement prompt templates and management system
- Design the agent orchestration framework

**Code Example: Model Provider Abstraction**
```python
# filepath: app/ai/model_provider.py
from enum import Enum
from langchain.chat_models import ChatOpenAI, ChatAnthropic
from langchain.schema.language_model import BaseLanguageModel

class ModelProvider(str, Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    # Add others as needed

def get_model_for_provider(
    provider: ModelProvider,
    temperature: float = 0.0,
    model_name: str = None
) -> BaseLanguageModel:
    """
    Returns a language model instance based on the specified provider.
    """
    if provider == ModelProvider.OPENAI:
        return ChatOpenAI(
            temperature=temperature,
            model_name=model_name or "gpt-4o"
        )
    elif provider == ModelProvider.ANTHROPIC:
        return ChatAnthropic(
            temperature=temperature,
            model_name=model_name or "claude-3-opus-20240229"
        )
    else:
        raise ValueError(f"Unsupported model provider: {provider}")
```

### 3.2 Agent Implementation

#### Context Retrieval Agent
**Implementation Details:**
- Develop query understanding module
- Implement vector search utilities
- Create context ranking and selection logic
- Design caching mechanisms for frequent queries

**Code Example: Vector Search Function**
```python
# filepath: app/ai/retrieval.py
from app.database.vector_store import get_vector_store
from langchain.embeddings import OpenAIEmbeddings
import asyncio

embeddings = OpenAIEmbeddings()
vector_store = get_vector_store()

async def retrieve_relevant_context(query: str, top_k: int = 5):
    """
    Retrieves relevant context for a given query using vector similarity search.
    """
    # Generate embedding for the query
    query_embedding = await asyncio.to_thread(
        embeddings.embed_query, query
    )
    
    # Retrieve similar vectors
    results = vector_store.query(
        vector=query_embedding,
        top_k=top_k,
        include_metadata=True
    )
    
    return [match.metadata for match in results.matches]
```

#### Reasoning & Guidance Agent
**Implementation Details:**
- Develop solution verification module
- Implement hint generation system with progressive disclosure
- Create memory management for tracking understanding
- Design fallback mechanisms for when primary approach fails

**Code Example: Hint Generation**
```python
# filepath: app/ai/reasoning.py
from langchain.prompts import ChatPromptTemplate
from app.ai.model_provider import get_model_for_provider, ModelProvider

hint_generation_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a mathematics tutor helping a student. Generate a hint that guides them toward the solution without giving it away completely."),
    ("user", "Problem: {problem}\nStudent's work so far: {student_work}\nHint level (1-3, where 3 is most revealing): {hint_level}")
])

async def generate_progressive_hint(problem: str, student_work: str, hint_level: int):
    """
    Generates a progressive hint for a given problem based on the student's work.
    """
    model = get_model_for_provider(
        ModelProvider.OPENAI,
        temperature=0.3,
        model_name="gpt-4o"
    )
    
    chain = hint_generation_prompt | model
    
    response = await chain.ainvoke({
        "problem": problem,
        "student_work": student_work,
        "hint_level": hint_level
    })
    
    return response.content
```

### 3.3 Agent Orchestration with LangGraph
**Implementation Details:**
- Define agent workflow as a directed graph
- Implement state persistence between steps
- Create agent communication protocols
- Design error handling and graceful degradation

**Code Example: Agent Graph Definition**
```python
# filepath: app/ai/agent_graph.py
from langgraph.graph import StateGraph
from typing import TypedDict, List, Dict, Any
from app.ai.retrieval import retrieve_relevant_context
from app.ai.reasoning import generate_progressive_hint
from app.ai.memory import track_memory

# Define the state schema
class AgentState(TypedDict):
    query: str
    context: List[Dict[str, Any]]
    guidance: str
    memory: Dict[str, Any]

# Create the graph
def create_agent_graph():
    graph = StateGraph(AgentState)
    
    # Add nodes
    graph.add_node("retrieve_context", retrieve_relevant_context)
    graph.add_node("generate_guidance", generate_progressive_hint)
    graph.add_node("track_memory", track_memory)
    
    # Build edges
    graph.add_edge("start", "retrieve_context")
    graph.add_edge("retrieve_context", "generate_guidance")
    graph.add_edge("generate_guidance", "track_memory")
    graph.add_edge("track_memory", "end")
    
    # Compile
    agent_workflow = graph.compile()
    
    return agent_workflow
```

**Dev vs. Prod Considerations:**
- Test models in development with reduced costs (e.g., GPT-3.5 vs GPT-4)
- Comprehensive logging in development, minimal in production
- Synthetic users for testing in development
- Performance monitoring and optimization in production

## 4. Authentication and Authorization Implementation

### 4.1 Firebase Authentication Setup
**Implementation Details:**
- Configure Firebase Authentication with desired providers
- Implement custom claims for role-based access control
- Create authentication middleware for Express
- Design token refresh mechanisms

**Code Example: Auth Middleware**
```typescript
import { Request, Response, NextFunction } from 'express';
import { auth } from '../config/firebase';

export interface AuthenticatedRequest extends Request {
  user?: {
    uid: string;
    roles: string[];
  };
}

export const authenticate = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    const token = authHeader.split('Bearer ')[1];
    const decodedToken = await auth.verifyIdToken(token);
    
    req.user = {
      uid: decodedToken.uid,
      roles: decodedToken.roles || []
    };
    
    return next();
  } catch (error) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
};
```

### 4.2 Role-Based Access Control Implementation
**Implementation Details:**
- Define role hierarchy and permissions
- Implement role checking middleware
- Create admin interface for role management
- Design audit logging for permission changes

**Code Example: Role Middleware**
```typescript
import { Response, NextFunction } from 'express';
import { AuthenticatedRequest } from './authenticate';

export const requireRole = (role: string) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    if (!req.user.roles.includes(role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    return next();
  };
};
```

**Dev vs. Prod Considerations:**
- Local emulators for Firebase services in development
- Strict token expiration and security in production
- Test accounts with various roles in development
- Regular security audits in production

## 5. API Implementation

### 5.1 REST API Development
**Implementation Details:**
- Design RESTful endpoints following standard conventions
- Implement request validation using a library like Joi or Zod
- Create consistent error response format
- Implement pagination, filtering, and sorting

**Code Example: Validated Endpoint**
```typescript
import { Router } from 'express';
import { z } from 'zod';
import { validateRequest } from '../middleware/validateRequest';
import { requireRole } from '../middleware/requireRole';
import { createModule, getModules } from '../controllers/moduleController';

const router = Router();

const createModuleSchema = z.object({
  body: z.object({
    title: z.string().min(3).max(100),
    description: z.string().min(10).max(500),
    level: z.number().int().min(1).max(5),
    topics: z.array(z.string()).nonempty()
  })
});

router.post(
  '/',
  validateRequest(createModuleSchema),
  requireRole('admin'),
  createModule
);

router.get('/', getModules);

export default router;
```

### 5.2 GraphQL Implementation
**Implementation Details:**
- Set up Apollo Server or alternative GraphQL server
- Define GraphQL schema with types, queries, and mutations
- Implement resolvers with appropriate data fetching
- Configure dataloaders for efficient database access

**Code Example: GraphQL Schema and Resolver**
```typescript
import { gql } from 'apollo-server-express';
import { ModuleModel } from '../models/Module';

export const typeDefs = gql`
  type Module {
    id: ID!
    title: String!
    description: String!
    level: Int!
    topics: [String!]!
    createdAt: String!
    updatedAt: String!
  }
  
  type Query {
    modules(level: Int): [Module!]!
    module(id: ID!): Module
  }
`;

export const resolvers = {
  Query: {
    modules: async (_, { level }) => {
      const query = level ? { level } : {};
      return await ModuleModel.find(query);
    },
    module: async (_, { id }) => {
      return await ModuleModel.findById(id);
    }
  }
};
```

**Dev vs. Prod Considerations:**
- GraphQL Playground enabled in development, disabled in production
- Rate limiting and complexity analysis in production
- Query logging in development
- APM integration in production

## 6. Payment Integration

### 6.1 Stripe Implementation
**Implementation Details:**
- Set up Stripe account and API keys
- Implement payment intent creation and confirmation
- Design webhook handling for async events
- Create subscription management workflows

**Code Example: Stripe Payment Intent**
```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2022-11-15',
});

export async function createPaymentIntent(amount: number, currency: string, customerId: string) {
  return await stripe.paymentIntents.create({
    amount,
    currency,
    customer: customerId,
    payment_method_types: ['card'],
  });
}
```

### 6.2 Subscription Service
**Implementation Details:**
- Define subscription tiers and features
- Implement subscription creation and management
- Create upgrade/downgrade flows
- Design billing cycle and invoice handling

**Dev vs. Prod Considerations:**
- Use Stripe test keys in development
- Implement extensive logging for payment flows
- Create test subscriptions in development
- Configure webhook security in production

## 7. Testing Implementation

### 7.1 Test Infrastructure Setup
**Implementation Details:**
- Configure Jest or alternative test runner
- Set up test database connections
- Implement test utilities and fixtures
- Create CI pipeline for automated testing

**Code Example: API Test**
```typescript
import request from 'supertest';
import { app } from '../src/app';
import { connectTestDB, disconnectTestDB, clearTestDB } from './utils/testDb';

beforeAll(async () => {
  await connectTestDB();
});

afterEach(async () => {
  await clearTestDB();
});

afterAll(async () => {
  await disconnectTestDB();
});

describe('Module API', () => {
  it('should create a new module when authenticated as admin', async () => {
    const moduleData = {
      title: 'Algebra Basics',
      description: 'Introduction to algebraic concepts',
      level: 2,
      topics: ['equations', 'expressions']
    };
    
    const res = await request(app)
      .post('/api/modules')
      .set('Authorization', `Bearer ${adminToken}`)
      .send(moduleData);
    
    expect(res.status).toBe(201);
    expect(res.body).toHaveProperty('id');
    expect(res.body.title).toBe(moduleData.title);
  });
});
```

### 7.2 AI Evaluation Framework
**Implementation Details:**
- Create ground truth datasets for AI evaluation
- Implement evaluation metrics and scoring
- Design automated testing for agent responses
- Create reporting dashboard for AI performance

**Dev vs. Prod Considerations:**
- Extensive AI testing in development environment
- Gradual rollout of AI changes in production
- Synthetic user testing in development
- A/B testing framework in production

## 8. Deployment Pipeline

### 8.1 Initial Firebase Deployment
**Implementation Details:**
- Configure Firebase project and services
- Set up Cloud Functions for API endpoints
- Implement deployment scripts and workflows
- Configure environment variables and secrets

**Code Example: Firebase Deployment Config**
```javascript
// firebase.json
{
  "functions": {
    "source": "functions",
    "predeploy": [
      "npm --prefix \"$RESOURCE_DIR\" run lint",
      "npm --prefix \"$RESOURCE_DIR\" run build"
    ]
  },
  "hosting": {
    "public": "public",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "/api/**",
        "function": "api"
      },
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

### 8.2 Containerized Deployment
**Implementation Details:**
- Create Docker configurations for each service
- Set up Kubernetes manifests for deployment
- Implement CI/CD pipeline using GitHub Actions
- Configure monitoring and logging services

**Code Example: Docker Compose for Development**
```yaml
version: '3'
services:
  api-gateway:
    build: ./gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - ./gateway:/app
      - /app/node_modules

  auth-service:
    build: ./services/auth
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
    volumes:
      - ./services/auth:/app
      - /app/node_modules

  # Additional services follow the same pattern
```

**Dev vs. Prod Considerations:**
- Local Docker Compose for development
- Kubernetes for staging and production
- Infrastructure as code for environment consistency
- Automated rollbacks in production

## 9. Monitoring and Logging Implementation

### 9.1 OpenTelemetry Setup
**Implementation Details:**
- Configure OpenTelemetry collectors
- Implement trace propagation across services
- Set up metrics collection and dashboard
- Create alerting based on key metrics

**Code Example: OpenTelemetry Setup**
```typescript
import { NodeTracerProvider } from '@opentelemetry/node';
import { BatchSpanProcessor } from '@opentelemetry/tracing';
import { CollectorTraceExporter } from '@opentelemetry/exporter-collector';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'learning-platform',
  }),
});

const exporter = new CollectorTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
});

provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();
```

### 9.2 Logging Implementation
**Implementation Details:**
- Set up structured logging with Winston or Pino
- Configure log aggregation with ELK stack or alternatives
- Implement log correlation with traces
- Create log-based alerts and dashboards

**Dev vs. Prod Considerations:**
- Verbose logging in development
- Structured, minimal logging in production
- Local log viewing in development
- Centralized log management in production

## 10. Conclusion and Next Steps

This roadmap provides a comprehensive guide for implementing the backend systems of the Personalised Learning Platform. By following the outlined approach, you'll create a scalable, maintainable, and efficient backend that supports the platform's educational objectives.

### Next Steps:
1. Begin with core infrastructure setup and base services
2. Implement the authentication and user management services
3. Develop the learning module and progress tracking components
4. Build the AI agent system incrementally
5. Integrate payment processing
6. Establish comprehensive testing and monitoring

Regular reviews of this roadmap are recommended as the project evolves and requirements become clearer through implementation experience.
