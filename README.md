# AirSense AI — Air Quality Monitoring System

AirSense AI is a full-stack, real-time air quality monitoring platform that combines live environmental data ingestion, machine learning-based forecasting, and an interactive citizen-facing dashboard. The system is designed to make air quality data accessible, interpretable, and actionable for everyday users — going beyond raw numbers to deliver health guidance, source-level analysis, and forward-looking predictions.

---

## What This System Does

Most government air quality portals present raw AQI numbers with little context and no forecasting. AirSense AI changes that by building three layers of intelligence on top of the raw data:

- It continuously pulls live pollutant readings and weather data from external providers
- It runs those readings through a trained TensorFlow model to produce a 72-hour forecast
- It surfaces everything through a responsive dashboard with health recommendations, source attribution, and an interactive city-wide map

The result is a platform where a user can open their browser, see exactly what the air is like right now, understand what is causing it, know whether it will get better or worse over the next three days, and receive clear guidance on what precautions to take.

---

## Architectural Overview

The system is divided into four distinct layers that each own a specific concern. They are loosely coupled, meaning changes in one layer do not require changes in others.

```
+---------------------------+
|     React Frontend        |  <-- User interface (port 3000)
|  Dashboard / Map / Chart  |
+------------+--------------+
             |
             | HTTP (Axios)
             |
+------------v--------------+
|     FastAPI Backend        |  <-- REST API server (port 8000)
|  /api/v1/aqi/current       |
|  /api/v1/forecast          |
|  /api/v1/source-attribution|
|  /api/v1/alerts            |
+------+----------+----------+
       |          |
       |          | Inference call
       |     +----v-----------+
       |     |   ML Models    |  <-- TensorFlow (72-hour AQI forecasting)
       |     |  (trained_models/) |
       |     +----------------+
       |
+------v-----------+
|   Data Pipeline  |  <-- Scheduled collectors
|  aqi_collector   |  <-- Pulls from AQICN API
|  weather_collector| <-- Pulls from OpenWeatherMap
+------------------+
       |
+------v-----------+
|  External APIs   |
|  AQICN           |
|  OpenWeatherMap  |
+------------------+
```

---

## Data Flow

Understanding how data moves through the system from raw source to rendered dashboard is the most important thing to grasp about AirSense AI.

**Ingestion**

The data pipeline runs two collector scripts on a schedule. `aqi_collector.py` connects to the AQICN API using the configured token and retrieves current pollutant readings — PM2.5, PM10, NO2, SO2, CO, and O3 — for the configured latitude and longitude. `weather_collector.py` connects to OpenWeatherMap and retrieves temperature and humidity readings for the same location. Both collectors write their results to the `data/raw/` directory.

**Processing**

Raw collected data is cleaned, normalised, and structured by the pipeline before being written to `data/processed/`. This processed data is what the backend reads when it responds to API requests. It is also the input format expected by the ML model when generating forecasts.

**Inference**

When the frontend requests a forecast, the backend loads the trained TensorFlow model from `ml-models/trained_models/` and runs inference against the most recent processed data. The model outputs predicted AQI values at hourly intervals for the next 72 hours. These predictions are returned directly in the API response as a time-series array.

**API Layer**

The FastAPI backend exposes four versioned endpoints under `/api/v1/`. Each endpoint reads from the processed data store and, where applicable, triggers model inference before composing and returning a structured JSON response. All four endpoints accept `latitude` and `longitude` as query parameters, making the system location-aware without requiring per-user data storage.

**Frontend Rendering**

The React frontend calls each endpoint through Axios on component mount and then on a configurable auto-refresh interval. The three main components each own a slice of the dashboard:

- `AQIDashboard.jsx` reads from `/api/v1/aqi/current` and renders the live AQI card, pollutant breakdown table, and current weather strip.
- `AQIForecast.jsx` reads from `/api/v1/forecast` and feeds the response into a Recharts time-series chart showing predicted AQI across the next 72 hours.
- `AQIMap.jsx` reads multiple monitoring point readings and renders them as colour-coded markers on a Leaflet map, allowing the user to explore spatial variation across a city.

Health alerts from `/api/v1/alerts` and source attribution from `/api/v1/source-attribution` are rendered as contextual panels beneath the main AQI card.

---

## Component Breakdown

### Backend (Python / FastAPI)

The backend is structured into four internal packages:

`api/` contains the route handlers. Each handler validates incoming request parameters, calls into the service layer, and serialises the result into a JSON response.

`core/` holds application-wide configuration loading and shared constants. Environment variables from the `.env` file are read here and made available to the rest of the application.

`services/` contains the business logic. This is where data is read from the processed store, where the ML model is invoked, where health thresholds are evaluated against current readings, and where source attribution is computed. The service layer is the only part of the backend that knows about the data files and the model.

`models/` defines the Pydantic data models that describe the shape of every request and response. These models ensure consistency between what the service layer produces and what the API layer serialises.

### Machine Learning

The ML model is a TensorFlow time-series model trained on historical AQI readings. Training scripts live in `ml-models/scripts/` alongside the serialised model files in `ml-models/trained_models/`. The model takes a window of recent processed readings as input and outputs hourly AQI predictions for the next 72 hours. NumPy and Pandas handle the data preparation steps before and after inference.

### Data Pipeline

The pipeline is intentionally simple — two independent collectors and a shared configuration module. `config.py` reads coordinates, API keys, and output paths from the shared `.env` file. Each collector is a standalone script that can be run on a schedule or triggered manually. This design means the pipeline has no dependency on the backend or the frontend; it only writes files to disk.

### Frontend (React / Tailwind CSS)

The frontend is a single-page React 19 application. Component state is local — there is no global state manager — because the data dependencies are straightforward: each of the three dashboard components fetches its own data independently. Tailwind CSS handles all styling. The auto-refresh mechanism is implemented as a `setInterval` inside a `useEffect` hook in each component, polling the relevant endpoint at a fixed interval and updating local state on each successful response.

---

## API Reference

| Method | Endpoint | Query Parameters | Description |
|--------|----------|-----------------|-------------|
| GET | `/api/v1/aqi/current` | `latitude`, `longitude` | Current AQI reading and full pollutant breakdown |
| GET | `/api/v1/forecast` | `latitude`, `longitude`, `hours` (default 72) | Time-series AQI forecast from the ML model |
| GET | `/api/v1/source-attribution` | `latitude`, `longitude` | Estimated contribution breakdown by pollution source |
| GET | `/api/v1/alerts` | — | Active health alerts based on current AQI thresholds |

The interactive API documentation generated by FastAPI is available at `http://localhost:8000/docs` while the server is running. Every endpoint, parameter, and response schema is browsable and testable there.

---

## AQI Health Thresholds

The alerts endpoint maps current AQI readings to health guidance using the following standard thresholds:

| AQI Range | Category | Health Implication |
|-----------|----------|--------------------|
| 0 – 50 | Good | Air quality is satisfactory; no health risk |
| 51 – 100 | Moderate | Acceptable quality; unusually sensitive people may be affected |
| 101 – 150 | Unhealthy for Sensitive Groups | Sensitive individuals should limit prolonged outdoor exertion |
| 151 – 200 | Unhealthy | Everyone may begin to experience health effects |
| 201 – 300 | Very Unhealthy | Health alert; everyone may experience serious effects |
| 301+ | Hazardous | Emergency conditions; entire population is affected |

---

## Pollutants Monitored

| Pollutant | Full Name | Primary Source |
|-----------|-----------|---------------|
| PM2.5 | Fine particulate matter (< 2.5 µm) | Vehicle exhaust, industrial combustion |
| PM10 | Coarse particulate matter (< 10 µm) | Dust, construction, road debris |
| NO2 | Nitrogen Dioxide | Fuel combustion, traffic |
| SO2 | Sulphur Dioxide | Industrial processes, power plants |
| CO | Carbon Monoxide | Incomplete combustion, vehicle exhaust |
| O3 | Ground-level Ozone | Photochemical reaction of NOx and VOCs |

---

## Project Structure

```
AIR_SENSE/
+-- backend/
|   +-- app/
|   |   +-- main.py                 Entry point; starts the Uvicorn server
|   |   +-- api/                    Route handlers for each endpoint
|   |   +-- core/                   Configuration loading and shared constants
|   |   +-- models/                 Pydantic request and response models
|   |   +-- services/               Business logic: data access, ML inference, alert evaluation
|   +-- requirements.txt
|
+-- frontend/
|   +-- citizen-app/
|       +-- src/
|       |   +-- components/
|       |   |   +-- AQIDashboard.jsx     Current AQI card, pollutants, weather
|       |   |   +-- AQIForecast.jsx      72-hour forecast chart
|       |   |   +-- AQIMap.jsx           Interactive city-wide AQI map
|       |   +-- App.js
|       |   +-- index.js
|       +-- package.json
|
+-- data-pipeline/
|   +-- collectors/
|   |   +-- aqi_collector.py        Fetches pollutant data from AQICN
|   |   +-- weather_collector.py    Fetches weather data from OpenWeatherMap
|   +-- config.py                   Shared configuration for collectors
|
+-- ml-models/
|   +-- trained_models/             Serialised TensorFlow model files
|   +-- scripts/                    Training and evaluation scripts
|
+-- data/
|   +-- raw/                        Unprocessed collector output
|   +-- processed/                  Cleaned, normalised data ready for inference
|
+-- .env                            API keys, coordinates, city name
```

---

## Technology Stack

| Layer | Technology | Role |
|-------|-----------|------|
| Frontend framework | React 19 | Component-based UI |
| Charting | Recharts | 72-hour forecast visualisation |
| Mapping | Leaflet | Interactive AQI station map |
| HTTP client | Axios | Frontend-to-backend API calls |
| Styling | Tailwind CSS | Utility-first responsive design |
| Backend framework | FastAPI | REST API server |
| ASGI server | Uvicorn | Serves the FastAPI application |
| ML framework | TensorFlow | 72-hour AQI forecasting model |
| Data processing | Pandas + NumPy | Pipeline data cleaning and model input preparation |
| External AQI data | AQICN API | Live pollutant readings |
| External weather data | OpenWeatherMap API | Temperature and humidity |

---

## License

This project is licensed under the MIT License.
