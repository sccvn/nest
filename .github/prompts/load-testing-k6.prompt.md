# Load Testing Expertise (K6, JMeter, Locust)

## Overview
This prompt provides comprehensive expertise in performance and load testing methodologies, tools, and best practices for ensuring system reliability under various load conditions.

## K6 Load Testing

### Installation & Setup
```bash
# Install K6
brew install k6  # macOS
# or
curl https://github.com/grafana/k6/releases/download/v0.45.0/k6-v0.45.0-linux-amd64.tar.gz -L | tar xvz

# Run basic test
k6 run script.js
```

### Basic Test Structure
```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const orderLatency = new Trend('order_latency');
const successfulOrders = new Counter('successful_orders');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 200 },   // Ramp up to 200 users
    { duration: '5m', target: 200 },   // Stay at 200 users
    { duration: '2m', target: 0 },     // Ramp down to 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],                   // Error rate < 1%
    errors: ['rate<0.05'],                            // Custom error rate < 5%
    order_latency: ['p(95)<800'],                     // Order latency threshold
  },
};

// Setup function - runs once before test
export function setup() {
  const loginRes = http.post('https://api.example.com/auth/login', JSON.stringify({
    email: 'test@example.com',
    password: 'password',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  
  return {
    token: JSON.parse(loginRes.body).token,
  };
}

// Default function - main test logic
export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Product Browsing', () => {
    // List products
    const listRes = http.get('https://api.example.com/products', { headers });
    check(listRes, {
      'products list status is 200': (r) => r.status === 200,
      'products list has items': (r) => JSON.parse(r.body).length > 0,
    });

    sleep(1); // User think time

    // View product detail
    const productId = JSON.parse(listRes.body)[0].id;
    const detailRes = http.get(`https://api.example.com/products/${productId}`, { headers });
    check(detailRes, {
      'product detail status is 200': (r) => r.status === 200,
    });
  });

  group('Order Creation', () => {
    const startTime = new Date();
    
    const orderRes = http.post('https://api.example.com/orders', JSON.stringify({
      items: [{ productId: 'prod-1', quantity: 2 }],
      shippingAddress: {
        street: '123 Test St',
        city: 'Test City',
        zip: '12345',
      },
    }), { headers });

    const duration = new Date() - startTime;
    orderLatency.add(duration);

    const success = check(orderRes, {
      'order created successfully': (r) => r.status === 201,
      'order has id': (r) => JSON.parse(r.body).id !== undefined,
    });

    if (success) {
      successfulOrders.add(1);
    } else {
      errorRate.add(1);
    }
  });

  sleep(Math.random() * 3 + 1); // Random sleep 1-4 seconds
}

// Teardown function - runs once after test
export function teardown(data) {
  // Cleanup if needed
  console.log('Test completed');
}
```

### Advanced Scenarios

#### Spike Testing
```javascript
export const options = {
  stages: [
    { duration: '10s', target: 100 },   // Normal load
    { duration: '1m', target: 100 },
    { duration: '10s', target: 1400 },  // Spike to 1400 users
    { duration: '3m', target: 1400 },   // Stay at spike
    { duration: '10s', target: 100 },   // Scale back
    { duration: '3m', target: 100 },    // Recovery period
    { duration: '10s', target: 0 },
  ],
};
```

#### Stress Testing
```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '5m', target: 300 },
    { duration: '2m', target: 400 },
    { duration: '5m', target: 400 },  // Push to breaking point
    { duration: '10m', target: 0 },   // Recovery
  ],
};
```

#### Soak Testing
```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up
    { duration: '4h', target: 100 },   // Stay at 100 for 4 hours
    { duration: '5m', target: 0 },     // Ramp down
  ],
};
```

### Scenarios with Different Load Profiles
```javascript
export const options = {
  scenarios: {
    // Scenario 1: Constant VUs browsing
    browsers: {
      executor: 'constant-vus',
      vus: 50,
      duration: '10m',
      exec: 'browseProducts',
    },
    // Scenario 2: Ramping arrivals for orders
    orders: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1m',
      preAllocatedVUs: 50,
      maxVUs: 200,
      stages: [
        { duration: '5m', target: 50 },
        { duration: '5m', target: 100 },
        { duration: '5m', target: 50 },
      ],
      exec: 'createOrder',
    },
    // Scenario 3: Constant arrival rate for health checks
    healthcheck: {
      executor: 'constant-arrival-rate',
      rate: 60,  // 60 iterations per timeUnit
      timeUnit: '1m',
      duration: '10m',
      preAllocatedVUs: 5,
      exec: 'healthCheck',
    },
  },
};

export function browseProducts() {
  http.get('https://api.example.com/products');
  sleep(2);
}

export function createOrder() {
  http.post('https://api.example.com/orders', JSON.stringify({ items: [] }), {
    headers: { 'Content-Type': 'application/json' },
  });
}

export function healthCheck() {
  http.get('https://api.example.com/health');
}
```

### WebSocket Testing
```javascript
import ws from 'k6/ws';
import { check } from 'k6';

export default function() {
  const url = 'wss://api.example.com/ws';
  const params = { tags: { name: 'websocket' } };

  const res = ws.connect(url, params, function(socket) {
    socket.on('open', () => {
      console.log('Connected');
      socket.send(JSON.stringify({ type: 'subscribe', channel: 'orders' }));
    });

    socket.on('message', (msg) => {
      const data = JSON.parse(msg);
      check(data, {
        'message has type': (d) => d.type !== undefined,
      });
    });

    socket.on('close', () => console.log('Disconnected'));

    socket.setTimeout(() => {
      socket.close();
    }, 30000);  // Close after 30s
  });

  check(res, { 'status is 101': (r) => r && r.status === 101 });
}
```

### CI/CD Integration
```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  schedule:
    - cron: '0 2 * * *'  # Run daily at 2 AM
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run load test
        run: k6 run --out json=results.json tests/load/main.js
        env:
          BASE_URL: ${{ secrets.TEST_BASE_URL }}
          API_KEY: ${{ secrets.TEST_API_KEY }}
      
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: load-test-results
          path: results.json
      
      - name: Check thresholds
        run: |
          if grep -q '"type":"Point","metric":"http_req_failed"' results.json && \
             jq -e '.data.value > 0.01' results.json; then
            echo "Error rate exceeded threshold"
            exit 1
          fi
```

## JMeter Load Testing

### Test Plan Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="E-Commerce Load Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL" elementType="Argument">
            <stringProp name="Argument.name">BASE_URL</stringProp>
            <stringProp name="Argument.value">${__P(baseUrl,https://api.example.com)}</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Users">
        <intProp name="ThreadGroup.num_threads">100</intProp>
        <intProp name="ThreadGroup.ramp_time">60</intProp>
        <boolProp name="ThreadGroup.same_user_on_next_iteration">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <!-- HTTP Request Defaults -->
        <ConfigTestElement guiclass="HttpDefaultsGui" testclass="ConfigTestElement" testname="HTTP Defaults">
          <stringProp name="HTTPSampler.domain">${BASE_URL}</stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.contentEncoding">UTF-8</stringProp>
        </ConfigTestElement>
        
        <!-- HTTP Header Manager -->
        <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="Headers">
          <collectionProp name="HeaderManager.headers">
            <elementProp name="Content-Type" elementType="Header">
              <stringProp name="Header.name">Content-Type</stringProp>
              <stringProp name="Header.value">application/json</stringProp>
            </elementProp>
          </collectionProp>
        </HeaderManager>
        
        <!-- Login Request -->
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Login">
          <stringProp name="HTTPSampler.path">/auth/login</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
          <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments">
              <elementProp name="" elementType="HTTPArgument">
                <stringProp name="Argument.value">{"email":"user@test.com","password":"test123"}</stringProp>
              </elementProp>
            </collectionProp>
          </elementProp>
        </HTTPSamplerProxy>
        <hashTree>
          <!-- JSON Extractor for Token -->
          <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="Extract Token">
            <stringProp name="JSONPostProcessor.referenceNames">token</stringProp>
            <stringProp name="JSONPostProcessor.jsonPathExprs">$.token</stringProp>
          </JSONPostProcessor>
        </hashTree>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Running JMeter from CLI
```bash
# Run test in non-GUI mode
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# With properties
jmeter -n -t test-plan.jmx \
  -JbaseUrl=https://api.staging.example.com \
  -Jthreads=200 \
  -Jrampup=120 \
  -l results.jtl

# Generate HTML report from results
jmeter -g results.jtl -o report/
```

## Locust Load Testing (Python)

### Basic Test
```python
# locustfile.py
from locust import HttpUser, task, between, events
from locust.runners import MasterRunner
import json
import logging

class ECommerceUser(HttpUser):
    wait_time = between(1, 5)  # Random wait 1-5 seconds
    token = None

    def on_start(self):
        """Called when user starts - login and get token"""
        response = self.client.post("/auth/login", json={
            "email": "test@example.com",
            "password": "password"
        })
        if response.status_code == 200:
            self.token = response.json()["token"]
        else:
            logging.error(f"Login failed: {response.status_code}")

    @task(3)
    def browse_products(self):
        """Browse products - weight 3 (most common)"""
        headers = {"Authorization": f"Bearer {self.token}"}
        
        with self.client.get("/products", headers=headers, catch_response=True) as response:
            if response.status_code == 200:
                products = response.json()
                if len(products) > 0:
                    response.success()
                else:
                    response.failure("No products returned")
            else:
                response.failure(f"Status: {response.status_code}")

    @task(2)
    def view_product_detail(self):
        """View single product - weight 2"""
        headers = {"Authorization": f"Bearer {self.token}"}
        product_id = "prod-123"  # In real test, pick from browsed products
        
        self.client.get(f"/products/{product_id}", headers=headers, name="/products/[id]")

    @task(1)
    def create_order(self):
        """Create order - weight 1 (least common)"""
        headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }
        
        order_data = {
            "items": [{"productId": "prod-123", "quantity": 1}],
            "shippingAddress": {
                "street": "123 Test St",
                "city": "Test City",
                "zip": "12345"
            }
        }
        
        with self.client.post("/orders", json=order_data, headers=headers, catch_response=True) as response:
            if response.status_code == 201:
                order = response.json()
                if "id" in order:
                    response.success()
                else:
                    response.failure("Order ID not in response")
            else:
                response.failure(f"Order creation failed: {response.status_code}")

    def on_stop(self):
        """Called when user stops"""
        pass


class AdminUser(HttpUser):
    """Separate user class for admin operations"""
    wait_time = between(5, 10)
    weight = 1  # 1 admin for every 10 regular users (default weight is 10)

    @task
    def check_analytics(self):
        self.client.get("/admin/analytics")


# Custom event handlers
@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    logging.info("Load test starting...")
    if isinstance(environment.runner, MasterRunner):
        logging.info("Running in distributed mode")


@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    logging.info("Load test completed")
    logging.info(f"Total requests: {environment.stats.total.num_requests}")
    logging.info(f"Total failures: {environment.stats.total.num_failures}")


@events.request.add_listener
def on_request(request_type, name, response_time, response_length, response, context, exception, **kwargs):
    if exception:
        logging.error(f"Request failed: {name} - {exception}")
```

### Running Locust
```bash
# Run locally with web UI
locust -f locustfile.py --host=https://api.example.com

# Headless mode
locust -f locustfile.py --host=https://api.example.com \
  --headless -u 100 -r 10 --run-time 5m

# Distributed mode - master
locust -f locustfile.py --master

# Distributed mode - workers
locust -f locustfile.py --worker --master-host=master-ip
```

## Performance Metrics & Thresholds

### Key Metrics to Monitor
| Metric | Description | Typical Threshold |
|--------|-------------|-------------------|
| Response Time (p50) | Median response time | < 200ms |
| Response Time (p95) | 95th percentile | < 500ms |
| Response Time (p99) | 99th percentile | < 1000ms |
| Error Rate | % of failed requests | < 1% |
| Throughput | Requests per second | Depends on SLA |
| Concurrent Users | Active users at peak | Depends on capacity |
| Apdex Score | Application Performance Index | > 0.9 |

### SLA Example
```javascript
// K6 thresholds matching SLA
export const options = {
  thresholds: {
    // Response time SLA
    'http_req_duration': [
      'p(50)<200',   // 50% of requests under 200ms
      'p(95)<500',   // 95% under 500ms
      'p(99)<1000',  // 99% under 1s
      'max<3000',    // No request over 3s
    ],
    // Error rate SLA
    'http_req_failed': ['rate<0.01'],  // Less than 1% errors
    // Throughput SLA (handled by VU count)
    'http_reqs': ['rate>100'],  // At least 100 req/s
  },
};
```

## Test Reports

### HTML Report Template
```html
<!DOCTYPE html>
<html>
<head>
  <title>Load Test Report</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    .metric { display: inline-block; margin: 10px; padding: 20px; background: #f5f5f5; border-radius: 8px; }
    .metric-value { font-size: 24px; font-weight: bold; }
    .metric-label { color: #666; }
    .pass { color: green; }
    .fail { color: red; }
    table { border-collapse: collapse; width: 100%; margin-top: 20px; }
    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
    th { background: #4CAF50; color: white; }
  </style>
</head>
<body>
  <h1>Load Test Report - {{date}}</h1>
  
  <h2>Summary</h2>
  <div class="metric">
    <div class="metric-value">{{totalRequests}}</div>
    <div class="metric-label">Total Requests</div>
  </div>
  <div class="metric">
    <div class="metric-value {{errorRateClass}}">{{errorRate}}%</div>
    <div class="metric-label">Error Rate</div>
  </div>
  <div class="metric">
    <div class="metric-value">{{avgResponseTime}}ms</div>
    <div class="metric-label">Avg Response Time</div>
  </div>
  <div class="metric">
    <div class="metric-value">{{p95ResponseTime}}ms</div>
    <div class="metric-label">P95 Response Time</div>
  </div>
  <div class="metric">
    <div class="metric-value">{{throughput}} req/s</div>
    <div class="metric-label">Throughput</div>
  </div>

  <h2>Threshold Results</h2>
  <table>
    <tr>
      <th>Threshold</th>
      <th>Expected</th>
      <th>Actual</th>
      <th>Status</th>
    </tr>
    {{#thresholds}}
    <tr>
      <td>{{name}}</td>
      <td>{{expected}}</td>
      <td>{{actual}}</td>
      <td class="{{statusClass}}">{{status}}</td>
    </tr>
    {{/thresholds}}
  </table>

  <h2>Endpoint Breakdown</h2>
  <table>
    <tr>
      <th>Endpoint</th>
      <th>Requests</th>
      <th>Avg (ms)</th>
      <th>P95 (ms)</th>
      <th>Errors</th>
    </tr>
    {{#endpoints}}
    <tr>
      <td>{{name}}</td>
      <td>{{requests}}</td>
      <td>{{avgTime}}</td>
      <td>{{p95Time}}</td>
      <td>{{errors}}</td>
    </tr>
    {{/endpoints}}
  </table>
</body>
</html>
```

## Best Practices

### Test Design
1. **Realistic scenarios** - Model actual user behavior
2. **Think time** - Include realistic pauses between actions
3. **Data variety** - Use diverse test data
4. **Ramp up gradually** - Avoid sudden load spikes
5. **Run sufficient duration** - At least 5-10 minutes for steady state

### Environment
1. **Isolated environment** - Avoid affecting production
2. **Similar to production** - Same infrastructure scale
3. **Monitor all components** - Database, cache, queues
4. **Clear baseline** - Establish performance baselines

### Analysis
1. **Compare against baseline** - Track regression
2. **Identify bottlenecks** - CPU, memory, I/O, network
3. **Check error patterns** - Timeouts, 5xx errors
4. **Review logs** - Application and infrastructure
5. **Document findings** - Create actionable reports
