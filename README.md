# Salesforce Change Data Capture Python Client

A robust Python client for subscribing to Salesforce Change Data Capture (CDC) events using the Pub/Sub API. Eliminates race conditions and provides real-time data synchronization without the complexity of polling or record locking.

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![gRPC](https://img.shields.io/badge/gRPC-1.50+-green.svg)](https://grpc.io/)

## 🚨 The Problem

Traditional Salesforce data synchronization approaches suffer from critical issues:

### Common Anti-Patterns:
- **Checkbox flags** to mark "processed" records
- **Batch processes** polling every 5-15 minutes
- **REST/SOAP APIs** with complex retry logic
- **Record locking** with `FOR UPDATE` causing `UNABLE_TO_LOCK_ROW` exceptions

### The Pain Points:
- ❌ Race conditions between multiple processes
- ❌ Data inconsistency and lost updates
- ❌ Polling delays and missed changes
- ❌ Complex error handling and retry logic
- ❌ Performance degradation from lock contention
- ❌ Constant debugging of integration failures

## ✅ The Solution: Change Data Capture

CDC provides a modern, event-driven approach to data synchronization:

- 🚀 **Real-time updates** (sub-second latency)
- 🔄 **Multiple simultaneous subscribers**
- 📋 **Guaranteed event ordering** with sequence numbers
- 🔄 **Built-in replay capability** with replay IDs
- 🚫 **No polling or locking required**
- 🛡️ **Automatic failure recovery**

## 🏗️ Architecture

```
Salesforce → Change Data Capture → Pub/Sub API → Python Client → Your Application
     ↓              ↓                    ↓             ↓              ↓
   Record      Event Stream         gRPC Stream    Event Handler   Business Logic
   Changes      (Real-time)         (Reliable)     (Ordered)       (Your Code)
```

## 🚀 Quick Start

### 1. Prerequisites

- Python 3.8+
- Salesforce org with Change Data Capture enabled
- Valid Salesforce credentials

### 2. Installation

```bash
git clone https://github.com/david4eva/salesforce-change-data-capture-python.git
cd salesforce-cdc-python-client
pip install -r requirements.txt
```

### 3. Generate Protocol Buffer Files

```bash
# Download the Salesforce proto file
git clone https://github.com/forcedotcom/pub-sub-api.git
cp pub-sub-api/pubsub_api.proto proto/

# Generate Python files
python -m grpc_tools.protoc -I./proto --python_out=./proto --grpc_python_out=./proto ./proto/pubsub_api.proto
```

### 4. Configuration

Copy the example configuration:

```bash
cp config/config.yaml.example config/config.yaml
```

Edit `config/config.yaml`:

```yaml
salesforce:
  login_url: "https://login.salesforce.com"  # or your My Domain
  username: "your-username@example.com"
  password: "your-password-with-security-token"

pubsub:
  host: "api.pubsub.salesforce.com"
  port: 7443
  
topics:
  - "/data/AccountChangeEvent"
  - "/data/LeadChangeEvent"
  - "/data/OpportunityChangeEvent"

processing:
  batch_size: 10
  replay_from_stored: true
```

### 5. Basic Usage

```python
import grpc
import threading
import pubsub_api_pb2 as pb2
import pubsub_api_pb2_grpc as pb2_grpc

# Semaphore for flow control (from official Salesforce docs)
semaphore = threading.Semaphore(1)
latest_replay_id = None

# Authentication metadata (official format)
authmetadata = (
    ('accesstoken', 'your_session_id'),
    ('instanceurl', 'your_instance_url'), 
    ('tenantid', 'your_org_id')
)

# Create secure channel and stub
with grpc.secure_channel('api.pubsub.salesforce.com:7443', 
                       grpc.ssl_channel_credentials()) as channel:
    stub = pb2_grpc.PubSubStub(channel)
    
    # Subscribe to events
    def fetchReqStream(topic):
        while True:
            semaphore.acquire()
            yield pb2.FetchRequest(
                topic_name=topic,
                replay_preset=pb2.ReplayPreset.LATEST,
                num_requested=1
            )
    
    # Process event stream
    substream = stub.Subscribe(fetchReqStream('/data/Employee__ChangeEvent'), 
                              metadata=authmetadata)
    
    for event in substream:
        if event.events:
            for evt in event.events:
                # Decode event payload here
                print(f"Received event: {evt}")
        semaphore.release()
```

## 📚 Examples

### Simple Lead Monitoring

```python
import grpc
import threading
import io
import avro.schema
import avro.io
import pubsub_api_pb2 as pb2
import pubsub_api_pb2_grpc as pb2_grpc

# Global semaphore for flow control (official pattern)
semaphore = threading.Semaphore(1)

def decode_event(schema_json, payload):
    """Decode Avro payload using schema (required for Salesforce events)"""
    schema = avro.schema.parse(schema_json)
    buf = io.BytesIO(payload)
    decoder = avro.io.BinaryDecoder(buf)
    reader = avro.io.DatumReader(schema)
    return reader.read(decoder)

def fetchReqStream(topic):
    """Generator function for FetchRequest stream (official pattern)"""
    while True:
        semaphore.acquire()
        yield pb2.FetchRequest(
            topic_name=topic,
            replay_preset=pb2.ReplayPreset.LATEST,
            num_requested=1
        )

# Authentication setup (from official docs)
authmetadata = (
    ('accesstoken', 'your_session_id'),
    ('instanceurl', 'your_instance_url'),
    ('tenantid', 'your_org_id')
)

# Subscribe and process
with grpc.secure_channel('api.pubsub.salesforce.com:7443', 
                       grpc.ssl_channel_credentials()) as channel:
    stub = pb2_grpc.PubSubStub(channel)
    
    substream = stub.Subscribe(fetchReqStream('/data/Employee__ChangeEvent'), 
                              metadata=authmetadata)
    
    for event in substream:
        if event.events:
            for evt in event.events:
                # Get schema for decoding
                schema_request = pb2.SchemaRequest(schema_id=evt.event.schema_id)
                schema_response = stub.GetSchema(schema_request, metadata=authmetadata)
                
                # Decode the event
                decoded_event = decode_event(schema_response.schema_json, evt.event.payload)
                
                # Process based on change type
                change_header = decoded_event.get('ChangeEventHeader', {})
                change_type = change_header.get('changeType')
                
                if change_type == 'CREATE':
                    print(f"🆕 New employee: {decoded_event.get('Name')}")
                elif change_type == 'UPDATE':
                    print(f"📈 Employee updated: {decoded_event.get('Name')}")
                elif change_type == 'DELETE':
                    print(f"🗑️ Employee deleted")
        
        semaphore.release()
```

### Multi-Object Subscription

```python
from src.cdc_client import SalesforceCDCClient

class CRMDataProcessor:
    def handle_account_change(self, event):
        # Sync account data to external CRM
        self.sync_to_external_crm('account', event.data)
    
    def handle_contact_change(self, event):
        # Update contact in email marketing system
        self.update_email_marketing(event.data)
    
    def handle_opportunity_change(self, event):
        # Trigger sales analytics pipeline
        self.trigger_analytics_pipeline(event.data)

processor = CRMDataProcessor()
client = SalesforceCDCClient()

# Subscribe to multiple object types
client.subscribe('/data/AccountChangeEvent', processor.handle_account_change)
client.subscribe('/data/ContactChangeEvent', processor.handle_contact_change)
client.subscribe('/data/OpportunityChangeEvent', processor.handle_opportunity_change)

client.start_all_subscriptions()
```

### Error Handling and Retry

```python
from src.cdc_client import SalesforceCDCClient
import logging

def resilient_event_handler(event):
    try:
        # Your business logic
        process_business_logic(event)
        
    except Exception as e:
        logging.error(f"Error processing event {event.replay_id}: {e}")
        
        # Store failed event for retry
        store_failed_event(event)
        
        # Don't raise - let CDC continue processing other events

client = SalesforceCDCClient()
client.subscribe('/data/LeadChangeEvent', resilient_event_handler)
```

## 🔧 Configuration Options

### Complete Configuration Reference

```yaml
salesforce:
  # Authentication
  login_url: "https://login.salesforce.com"
  username: "your-username@example.com"
  password: "your-password-with-security-token"
  
  # Optional: OAuth (recommended for production)
  client_id: "your-connected-app-client-id"
  client_secret: "your-connected-app-client-secret"
  private_key_path: "path/to/server.key"

pubsub:
  host: "api.pubsub.salesforce.com"
  port: 7443
  
  # Connection settings
  max_receive_message_length: 1048576  # 1MB
  keepalive_time_ms: 30000
  keepalive_timeout_ms: 5000

topics:
  # Standard objects
  - "/data/AccountChangeEvent"
  - "/data/ContactChangeEvent"
  - "/data/LeadChangeEvent"
  - "/data/OpportunityChangeEvent"
  
  # Custom objects (replace CustomObject with your object name)
  - "/data/CustomObject__ChangeEvent"
  
  # All changes (high volume!)
  - "/data/ChangeEvents"

processing:
  # Flow control
  batch_size: 10                    # Events per request
  max_concurrent_events: 100        # Parallel processing limit
  
  # Replay settings
  replay_from_stored: true          # Resume from last processed event
  replay_storage_path: "./replay"   # Where to store replay IDs
  
  # Error handling
  max_retries: 3
  retry_delay_seconds: 5
  dead_letter_queue_path: "./failed_events"

logging:
  level: "INFO"
  format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  file: "cdc_client.log"
```

## 🔐 Authentication Methods

### 1. Username/Password (Development)

```yaml
salesforce:
  login_url: "https://login.salesforce.com"
  username: "your-username@example.com"
  password: "your-password-with-security-token"
```

### 2. OAuth JWT Bearer (Production Recommended)

```yaml
salesforce:
  login_url: "https://login.salesforce.com"
  client_id: "your-connected-app-client-id"
  username: "your-username@example.com"
  private_key_path: "path/to/server.key"
```

### 3. OAuth Web Server Flow

```python
from src.auth import OAuthWebServerFlow

# For web applications
auth = OAuthWebServerFlow(
    client_id='your-client-id',
    client_secret='your-client-secret',
    redirect_uri='https://your-app.com/oauth/callback'
)

authorization_url = auth.get_authorization_url()
# Redirect user to authorization_url
# Handle callback and exchange code for tokens
```

## 📊 Performance Considerations

### Throughput Guidelines

| Scenario | Recommended Settings | Expected Performance |
|----------|---------------------|---------------------|
| Low volume (< 1000 events/day) | batch_size: 1 | Real-time processing |
| Medium volume (1000-10000 events/day) | batch_size: 10 | < 1 second latency |
| High volume (> 10000 events/day) | batch_size: 50 | < 5 second latency |

### Memory Usage

- **Base client**: ~50MB
- **Per subscription**: ~10MB
- **Event buffering**: ~1KB per event
- **Replay storage**: ~100 bytes per processed event

### Network Requirements

- **Bandwidth**: ~1KB per event + overhead
- **Latency**: < 100ms to Salesforce (for optimal performance)
- **Reliability**: Persistent connection with automatic reconnection

## 🛠️ Advanced Features

### Custom Event Processors

```python
from src.event_processor import BaseEventProcessor

class CustomLeadProcessor(BaseEventProcessor):
    def __init__(self):
        super().__init__()
        self.lead_score_calculator = LeadScoringEngine()
    
    def process_create(self, event):
        """Handle new lead creation"""
        lead_data = event.data
        score = self.lead_score_calculator.calculate(lead_data)
        
        # Update lead score in Salesforce
        self.update_salesforce_record(event.record_id, {'Lead_Score__c': score})
    
    def process_update(self, event):
        """Handle lead updates"""
        if 'Email' in event.changed_fields:
            # Email changed - trigger email validation
            self.validate_email_async(event.data['Email'])
    
    def process_delete(self, event):
        """Handle lead deletion"""
        # Clean up external systems
        self.cleanup_external_data(event.record_id)

# Use custom processor
processor = CustomLeadProcessor()
client.subscribe('/data/LeadChangeEvent', processor.process)
```

### Batch Processing

```python
from src.batch_processor import BatchEventProcessor

def process_lead_batch(events):
    """Process multiple events together for efficiency"""
    
    # Group events by type
    creates = [e for e in events if e.change_type == 'CREATE']
    updates = [e for e in events if e.change_type == 'UPDATE']
    deletes = [e for e in events if e.change_type == 'DELETE']
    
    # Batch process each type
    if creates:
        bulk_create_external_records(creates)
    if updates:
        bulk_update_external_records(updates)
    if deletes:
        bulk_delete_external_records(deletes)

# Configure batch processing
batch_processor = BatchEventProcessor(
    batch_size=50,
    max_wait_time=5,  # seconds
    processor_function=process_lead_batch
)

client.subscribe('/data/LeadChangeEvent', batch_processor.add_event)
```

### Event Filtering

```python
from src.filters import EventFilter

# Create filters
high_value_filter = EventFilter(
    field='Annual_Revenue__c',
    operator='greater_than',
    value=1000000
)

status_filter = EventFilter(
    field='Status',
    operator='in',
    value=['Qualified', 'Converted']
)

# Apply filters
filtered_processor = FilteredEventProcessor([high_value_filter, status_filter])
client.subscribe('/data/LeadChangeEvent', filtered_processor.process)
```

## 🧪 Testing

### Unit Tests

```bash
# Run all tests
python -m pytest tests/

# Run specific test file
python -m pytest tests/test_cdc_client.py

# Run with coverage
python -m pytest --cov=src tests/
```

### Integration Tests

```bash
# Test against Salesforce sandbox (requires valid credentials)
python -m pytest tests/integration/ --sandbox

# Test replay functionality
python -m pytest tests/test_replay.py
```

### Mock Testing

```python
from tests.mock_salesforce import MockSalesforceServer

def test_event_processing():
    """Test event processing with mock data"""
    mock_server = MockSalesforceServer()
    
    # Generate test events
    mock_server.generate_lead_change_event({
        'Id': '00Q000000000001',
        'Name': 'Test Lead',
        'Status': 'New'
    })
    
    # Test client processing
    events_processed = []
    
    def test_handler(event):
        events_processed.append(event)
    
    client = SalesforceCDCClient(mock_server.config)
    client.subscribe('/data/LeadChangeEvent', test_handler)
    
    # Verify event was processed
    assert len(events_processed) == 1
    assert events_processed[0].data['Name'] == 'Test Lead'
```

## 🚨 Troubleshooting

### Common Issues

#### 1. Authentication Errors

```
Error: "INVALID_LOGIN: Invalid username, password, security token"
```

**Solutions:**
- Verify username and password
- Ensure security token is appended to password
- Check if IP is in trusted range or reset security token
- Verify login URL (production vs sandbox)

#### 2. gRPC Connection Issues

```
Error: "failed to connect to all addresses"
```

**Solutions:**
- Check firewall settings (allow port 7443)
- Verify network connectivity to api.pubsub.salesforce.com
- Try alternative port 443
- Check proxy settings

#### 3. Missing Events

```
Issue: Not receiving expected change events
```

**Solutions:**
- Verify Change Data Capture is enabled for the object
- Check object permissions for the authenticated user
- Confirm correct topic name format
- Review Salesforce setup logs

#### 4. Replay Issues

```
Error: "INVALID_REPLAY_ID"
```

**Solutions:**
- Use ReplayPreset.LATEST for new subscriptions
- Clear stored replay IDs if schema changed
- Verify replay ID format and age

### Debug Mode

Enable detailed logging:

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('salesforce_cdc')

client = SalesforceCDCClient(debug=True)
```

### Performance Monitoring

```python
from src.monitoring import PerformanceMonitor

monitor = PerformanceMonitor()
client = SalesforceCDCClient(monitor=monitor)

# View metrics
print(f"Events processed: {monitor.events_processed}")
print(f"Average latency: {monitor.average_latency}ms")
print(f"Error rate: {monitor.error_rate}%")
```

## 📈 Production Deployment

### Docker Container

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "src/cdc_client.py"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: salesforce-cdc-client
spec:
  replicas: 3
  selector:
    matchLabels:
      app: salesforce-cdc-client
  template:
    metadata:
      labels:
        app: salesforce-cdc-client
    spec:
      containers:
      - name: cdc-client
        image: your-registry/salesforce-cdc-client:latest
        env:
        - name: SALESFORCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: salesforce-credentials
              key: username
        - name: SALESFORCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: salesforce-credentials
              key: password
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Monitoring and Alerting

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge

events_processed = Counter('salesforce_events_processed_total', 'Total events processed')
event_processing_time = Histogram('salesforce_event_processing_seconds', 'Event processing time')
active_subscriptions = Gauge('salesforce_active_subscriptions', 'Number of active subscriptions')

def monitored_event_handler(event):
    with event_processing_time.time():
        try:
            process_event(event)
            events_processed.inc()
        except Exception as e:
            events_failed.inc()
            raise
```

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Clone and setup development environment
git clone https://github.com/yourusername/salesforce-cdc-python-client.git
cd salesforce-cdc-python-client

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install development dependencies
pip install -r requirements-dev.txt

# Install pre-commit hooks
pre-commit install

# Run tests
python -m pytest
```

### Code Style

We use:
- **Black** for code formatting
- **isort** for import sorting
- **flake8** for linting
- **mypy** for type checking

```bash
# Format code
black src/ tests/
isort src/ tests/

# Check style
flake8 src/ tests/
mypy src/
```

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Salesforce](https://developer.salesforce.com/) for the Pub/Sub API and excellent documentation
- [gRPC](https://grpc.io/) for the robust communication framework
- The Salesforce developer community for inspiration and feedback

## 📞 Support

- **Documentation**: [Salesforce Change Data Capture Guide](https://developer.salesforce.com/docs/platform/change-data-capture/)
- **Issues**: [GitHub Issues](https://github.com/yourusername/salesforce-cdc-python-client/issues)
- **Discussions**: [GitHub Discussions](https://github.com/yourusername/salesforce-cdc-python-client/discussions)

## 🗺️ Roadmap

### Version 2.0 (Coming Soon)
- [ ] GraphQL subscription support
- [ ] Built-in event filtering and transformation
- [ ] Redis-based replay ID storage
- [ ] Kubernetes operator
- [ ] Prometheus metrics integration

### Version 2.1
- [ ] Support for Platform Events
- [ ] Event replay from specific timestamps
- [ ] Dead letter queue with retry policies
- [ ] Multi-org subscription support

---

**⭐ If this project helps you, please consider starring it on GitHub!**

Made with ❤️ by the Salesforce developer community
