# Acoustic Guard - Infrastructure

Acoustic Guard is a distributed acoustic monitoring platform for real-time threat detection (UAV, explosions, sirens, generators, trucks) using edge agents, AI classification, and a central hub.

## Architecture Overview

The system consists of 4 main microservices:

1. **edge-agent (Rust)**: Audio capture from file/microphone, FFT processing, AMQP publishing to RabbitMQ
2. **classifier (Python)**: gRPC server running YAMNet/TensorFlow Lite for threat classification
3. **hub (Java/Spring Boot)**: Central orchestrator, RabbitMQ consumer, PostGIS persistence, WebSocket broadcaster
4. **frontend (React/Leaflet)**: Real-time map dashboard with analytics

### Data Flow

```
edge-agent (Rust) → RabbitMQ → hub (Java) → gRPC → classifier (Python)
                                    ↓
                              PostGIS DB
                                    ↓
                              WebSocket → frontend (React)
```

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- 8GB RAM minimum
- 20GB disk space

## Environment Variables Configuration

### Docker Compose Environment Variables

The system uses environment variables configured in `docker-compose.yaml`:

#### Classifier Service
- `CLASSIFIER_PORT=50051` - gRPC server port
- `LOG_LEVEL=INFO` - Logging level
- `EXPLOSION_THRESHOLD_DB=-25.0` - Explosion detection threshold (unused in current implementation)
- `MODEL_PATH=models/random_forest_v1.joblib` - Random Forest model path
- `YAMNET_EXPLOSION_CONFIDENCE=0.20` - YAMNet confidence threshold for explosions
- `YAMNET_UAV_CONFIDENCE=0.35` - YAMNet confidence threshold for UAV
- `YAMNET_SIREN_CONFIDENCE=0.10` - YAMNet confidence threshold for sirens
- `YAMNET_TRUCK_CONFIDENCE=0.30` - YAMNet confidence threshold for trucks
- `YAMNET_GENERATOR_CONFIDENCE=0.20` - YAMNet confidence threshold for generators

#### Java Hub Service
- `SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/acousticguard` - PostgreSQL connection string
- `SPRING_RABBITMQ_HOST=rabbitmq` - RabbitMQ host
- `SPRING_RABBITMQ_PORT=5672` - RabbitMQ port
- `SPRING_GRPC_CLIENT_CHANNELS_PYTHON_CLASSIFIER_ADDRESS=static://classifier:50051` - gRPC classifier address
- `SENSOR_HEARTBEAT_TIMEOUT_SECS=60` - Sensor heartbeat timeout in seconds (default: 60)

#### Edge Agent Services
- `RUST_LOG=info` - Rust logging level
- `AMQP_URI=amqp://guest:guest@rabbitmq:5672/%2f` - RabbitMQ connection URI
- `DEVICE_ID=node-XXX` - Unique sensor identifier
- `LATITUDE=50.XXXX` - Sensor latitude (Kyiv coordinates)
- `LONGITUDE=30.XXXX` - Sensor longitude (Kyiv coordinates)
- `AUDIO_SOURCE=FILE` - Audio source mode (FILE or MIC)
- `AUDIO_FILE_PATH=assets/processed/wind-001.wav` - Path to audio file (if FILE mode)
- `ANOMALY_THRESHOLD_DB=-20.0` - Anomaly detection threshold in dB
- `TELEMETRY_INTERVAL_SECS=20` - Telemetry reporting interval in seconds

### Additional Configuration

#### Java Hub (application.yml)
Located in `hub/src/main/resources/application.yml`:
- `acoustic.classifier.confidence-threshold=0.10` - Global confidence threshold for alert creation
- `acoustic.sensor.heartbeat-timeout-seconds=${SENSOR_HEARTBEAT_TIMEOUT_SECS:60}` - Sensor heartbeat timeout (configurable via SENSOR_HEARTBEAT_TIMEOUT_SECS env var, default: 60)
- `acoustic.incident.spatial-threshold-meters=500` - Spatial clustering threshold
- `acoustic.incident.temporal-threshold-seconds=300` - Temporal clustering threshold
- `jwt.secret` - JWT signing secret (configure via JWT_SECRET env var)

## Quick Start

### 1. Clone Repository

```bash
git clone <repository-url>
cd acoustic_guard/infra
```

### 2. Start All Services

```bash
docker-compose up -d
```

This starts:
- RabbitMQ (ports 5672, 15672)
- PostgreSQL with PostGIS (port 5432)
- Python classifier (port 50051)
- Java hub (port 8080)
- 13 edge agents (simulated sensors across Kyiv)

### 3. Verify Services

```bash
# Check RabbitMQ Management UI
open http://localhost:15672
# Username: guest, Password: guest

# Check Java Hub Health
curl http://localhost:8080/actuator/health

# Check Swagger UI
open http://localhost:8080/swagger-ui.html
```

### 4. Access Frontend

The React frontend should be accessible at:
```
http://localhost:3000
```

Default credentials:
- Admin: (configure via JWT)
- User: (configure via JWT)

## Service Details

### Edge Agents (13 simulated sensors)

The system includes 13 edge agents positioned across Kyiv:

**Background Noise (Darnytsia, Osokorky, Lisnyi Masyv):**
- node-001, node-002, node-003
- Audio: wind-001.wav
- Threshold: -20.0 dB

**Explosions & UAVs (Pozniaky):**
- node-004, node-005, node-006, node-007
- Audio: exp-001.wav, uav.wav
- Threshold: -55.0 dB

**Trucks (Rembaza):**
- node-008, node-009
- Audio: horn.wav
- Threshold: -55.0 dB

**Sirens & Generators (Bortnychi, Nyzhni Sady):**
- node-010, node-011, node-012, node-013
- Audio: siren.wav, generator.wav
- Threshold: -55.0 dB

### Database Schema

PostgreSQL with PostGIS extension includes:
- `alerts` - Real-time threat alerts
- `incidents` - Spatial incident records with status lifecycle
- `sensors` - Sensor registry with heartbeat tracking
- `telemetry` - Noise level telemetry data

### API Endpoints

**REST API (Base URL: http://localhost:8080/api/v1):**
- `GET /incidents` - Retrieve active incidents
- `GET /incidents/{id}` - Get specific incident
- `PATCH /incidents/{id}/status` - Update incident status
- `GET /analytics` - Get analytics data
- `GET /alerts` - Get alerts (admin only)
- `POST /auth/login` - JWT authentication

**WebSocket:**
- `/topic/alerts` - Real-time alert broadcasts
- `/topic/incidents` - Real-time incident updates
- `/topic/sensors` - Sensor status changes

**gRPC:**
- `classifier.v1.AudioClassifier/Classify` - Audio classification (port 50051)

## Live Deployment (docker-compose.live.yml)

For production deployment with live microphone input:

```bash
docker-compose -f docker-compose.live.yml up -d
```

This configuration:
- Uses real microphone input instead of pre-recorded files
- Adjusts anomaly thresholds for live environment
- Enables production-grade logging

## Troubleshooting

### Services Not Starting

```bash
# Check logs
docker-compose logs <service-name>

# Restart specific service
docker-compose restart <service-name>

# Rebuild service
docker-compose up -d --build <service-name>
```

### RabbitMQ Connection Issues

```bash
# Check RabbitMQ status
docker-compose exec rabbitmq rabbitmq-diagnostics -q ping

# View queues
docker-compose exec rabbitmq rabbitmqadmin list queues
```

### Database Connection Issues

```bash
# Check PostgreSQL status
docker-compose exec postgres pg_isready -U postgres

# Connect to database
docker-compose exec postgres psql -U postgres -d acousticguard
```

### Classifier Not Responding

```bash
# Check classifier logs
docker-compose logs classifier

# Test gRPC connection
docker-compose exec java-hub curl classifier:50051
```

## Stopping Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (deletes all data)
docker-compose down -v
```

## Monitoring

### RabbitMQ Management UI
- URL: http://localhost:15672
- Username: guest
- Password: guest

### PostgreSQL
- Host: localhost
- Port: 5432
- Database: acousticguard
- Username: postgres
- Password: password

### Logs

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f java-hub
docker-compose logs -f classifier
docker-compose logs -f edge-agent-1
```

## Development

### Rebuild Specific Service

```bash
docker-compose up -d --build <service-name>
```

### Access Service Shell

```bash
# Java Hub
docker-compose exec java-hub bash

# Python Classifier
docker-compose exec classifier bash

# Edge Agent
docker-compose exec edge-agent-1 sh
```

## Security Notes

⚠️ **Production Deployment Required Changes:**

1. Change default credentials (RabbitMQ, PostgreSQL, JWT secret)
2. Enable HTTPS/TLS for all communications
3. Configure firewall rules
4. Implement secrets management (e.g., HashiCorp Vault)
5. Enable authentication on RabbitMQ Management UI
6. Configure CORS for frontend domain
7. Implement rate limiting on API endpoints

## Support

For issues or questions, refer to the main project documentation or contact the development team.
