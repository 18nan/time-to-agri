# Design Document: TimeToAgri

## Overview

TimeToAgri (TTA) is a web-based agricultural decision support system that helps farmers optimize planting and selling decisions. The system follows a three-tier architecture:

1. **Frontend**: Next.js/React application providing responsive UI
2. **Backend**: FastAPI Python service handling business logic and ML predictions
3. **Database**: PostgreSQL for persistent data storage

The design emphasizes modularity, testability, and clear separation of concerns. The backend exposes RESTful APIs consumed by the frontend, with ML models encapsulated in dedicated service modules.

## Architecture

### System Components

3```mermaid
graph TB
    User[User Browser]
    FE[Next.js Frontend]
    API[FastAPI Backend]
    ML[ML Service Layer]
    DB[(PostgreSQL Database)]
    
    User -->|HTTPS| FE
    FE -->|REST API| API
    API -->|Query/Update| DB
    API -->|Predictions| ML
    ML -->|Read| DB
```

### Technology Stack

**Frontend**:
- Next.js 14 with App Router
- React 18 with TypeScript
- TailwindCSS for styling
- React Query for data fetching and caching
- Zod for runtime validation

**Backend**:
- FastAPI (Python 3.11+)
- Pydantic for data validation
- SQLAlchemy for ORM
- Alembic for database migrations
- scikit-learn/PyTorch for ML models
- Celery for async task processing (future enhancement)

**Database**:
- PostgreSQL 15+
- TimescaleDB extension for time-series data (price history, weather)

**Infrastructure**:
- Docker containers for deployment
- HTTPS/TLS for all communications
- JWT for authentication tokens

## Components and Interfaces

### Frontend Components

#### Authentication Module
- `LoginForm`: Handles user login with email/password
- `RegisterForm`: Handles new user registration
- `AuthProvider`: React context for authentication state
- `ProtectedRoute`: HOC for route protection

#### Farm Configuration Module
- `FarmProfileForm`: Input form for location, size, soil type
- `LocationPicker`: Interactive map or coordinate input
- `FarmDashboard`: Overview of saved farm information

#### Recommendations Module
- `PlantingWindowDisplay`: Shows optimal planting periods with confidence
- `YieldEstimateChart`: Visualizes yield predictions with confidence intervals
- `PriceForecastGraph`: Time-series chart of price predictions
- `StrategyComparison`: Side-by-side comparison of selling strategies

#### Shared Components
- `LoadingSpinner`: Indicates processing state
- `ErrorBoundary`: Catches and displays errors gracefully
- `Toast`: Notification system for user feedback

### Backend API Endpoints

#### Authentication Endpoints
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
GET /api/auth/me
PUT /api/auth/profile
```

#### Farm Configuration Endpoints
```
POST /api/farms
GET /api/farms/{farm_id}
PUT /api/farms/{farm_id}
DELETE /api/farms/{farm_id}
```

#### Recommendation Endpoints
```
GET /api/recommendations/planting-windows?crop={crop}&location={location}
GET /api/recommendations/yield-estimate?crop={crop}&farm_id={farm_id}
GET /api/recommendations/price-forecast?crop={crop}&horizon={days}
GET /api/recommendations/selling-strategies?crop={crop}&farm_id={farm_id}
```

### Backend Service Layer

#### AuthService
- `register_user(email, password, profile_data) -> User`
- `authenticate_user(email, password) -> Token`
- `validate_token(token) -> User`
- `update_profile(user_id, profile_data) -> User`

#### FarmService
- `create_farm(user_id, location, size, soil_type) -> Farm`
- `get_farm(farm_id, user_id) -> Farm`
- `update_farm(farm_id, user_id, updates) -> Farm`
- `validate_location(location) -> bool`

#### RecommendationService
- `get_planting_windows(crop, location) -> List[PlantingWindow]`
- `estimate_yield(crop, farm_id) -> YieldEstimate`
- `forecast_prices(crop, horizon_days) -> List[PriceForecast]`
- `compare_strategies(crop, farm_id) -> List[SellingStrategy]`

#### MLService
- `predict_yield(crop, location, soil_type, climate_data) -> float`
- `predict_price(crop, date, historical_data) -> float`
- `calculate_planting_windows(crop, location, climate_data) -> List[Window]`
- `optimize_strategy(yield, price_forecast, storage_cost) -> Strategy`

### Database Schema

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Farms Table
```sql
CREATE TABLE farms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    location GEOGRAPHY(POINT) NOT NULL,
    size_hectares DECIMAL(10,2),
    soil_type VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Crops Table
```sql
CREATE TABLE crops (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    scientific_name VARCHAR(200),
    growing_days_min INTEGER,
    growing_days_max INTEGER,
    optimal_temp_min DECIMAL(5,2),
    optimal_temp_max DECIMAL(5,2)
);
```

#### Historical Yields Table
```sql
CREATE TABLE historical_yields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id UUID REFERENCES farms(id) ON DELETE CASCADE,
    crop_id UUID REFERENCES crops(id),
    harvest_date DATE NOT NULL,
    yield_per_hectare DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### Price History Table (TimescaleDB Hypertable)
```sql
CREATE TABLE price_history (
    time TIMESTAMPTZ NOT NULL,
    crop_id UUID REFERENCES crops(id),
    price_per_unit DECIMAL(10,2) NOT NULL,
    market_location VARCHAR(100),
    PRIMARY KEY (time, crop_id, market_location)
);

SELECT create_hypertable('price_history', 'time');
```

## Data Models

### Frontend TypeScript Types

```typescript
interface User {
  id: string;
  email: string;
  createdAt: string;
}

interface Farm {
  id: string;
  userId: string;
  location: {
    latitude: number;
    longitude: number;
  };
  sizeHectares: number;
  soilType: string;
}

interface PlantingWindow {
  startDate: string;
  endDate: string;
  confidence: number; // 0-1
  suitabilityRank: number;
}

interface YieldEstimate {
  crop: string;
  yieldPerHectare: number;
  confidenceInterval: {
    lower: number;
    upper: number;
  };
  historicalYields?: number[];
}

interface PriceForecast {
  date: string;
  predictedPrice: number;
  confidenceInterval: {
    lower: number;
    upper: number;
  };
}

interface SellingStrategy {
  name: string;
  description: string;
  expectedRevenue: number;
  riskLevel: 'low' | 'medium' | 'high';
  timing: string;
  recommended: boolean;
}
```

### Backend Pydantic Models

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime, date
from typing import Optional, List
from enum import Enum

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

class UserResponse(BaseModel):
    id: str
    email: str
    created_at: datetime

class Location(BaseModel):
    latitude: float = Field(ge=-90, le=90)
    longitude: float = Field(ge=-180, le=180)

class FarmCreate(BaseModel):
    location: Location
    size_hectares: float = Field(gt=0)
    soil_type: str

class FarmResponse(BaseModel):
    id: str
    user_id: str
    location: Location
    size_hectares: float
    soil_type: str
    created_at: datetime

class PlantingWindow(BaseModel):
    start_date: date
    end_date: date
    confidence: float = Field(ge=0, le=1)
    suitability_rank: int

class YieldEstimate(BaseModel):
    crop: str
    yield_per_hectare: float
    confidence_interval: tuple[float, float]
    historical_yields: Optional[List[float]] = None

class PriceForecast(BaseModel):
    date: date
    predicted_price: float
    confidence_interval: tuple[float, float]

class RiskLevel(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

class SellingStrategy(BaseModel):
    name: str
    description: str
    expected_revenue: float
    risk_level: RiskLevel
    timing: str
    recommended: bool
```

## Correctness Properties

