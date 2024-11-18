Below is a detailed guide on how to set up **Prometheus** with an **Express.js** server, including the installation of dependencies, configuration of Prometheus, and visualization of the metrics using **Grafana**.

### **Step 1: Install Node.js and Express**

To get started, ensure you have **Node.js** installed. You can check by running `node -v` and `npm -v`. If you don't have them installed, download and install them from [Node.js official website](https://nodejs.org/).

Once Node.js is installed, create a new project and set up the Express.js server:

```bash
# Initialize a new Node.js project
mkdir express-prometheus
cd express-prometheus
npm init -y

# Install dependencies
npm install express prom-client
```

- `express`: The web server framework.
- `prom-client`: A client library to collect Prometheus metrics.

### **Step 2: Set Up the Express.js Server with Prometheus Metrics**

Create a file called `server.js`:

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = promClient.register;

// Create a counter metric to track HTTP requests
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests made',
  labelNames: ['method', 'status']
});

// Create a histogram metric to track request durations
const httpDurationHistogram = new promClient.Histogram({
  name: 'http_duration_seconds',
  help: 'Histogram of HTTP request durations in seconds',
  buckets: [0.1, 0.5, 1, 2, 5, 10] // Customize these buckets based on your latency needs
});

// Middleware to count HTTP requests
app.use((req, res, next) => {
  res.on('finish', () => {
    httpRequestsTotal.labels(req.method, res.statusCode).inc();  // Increment counter for each HTTP request
  });
  next();
});

// Middleware to measure request duration
app.use((req, res, next) => {
  const end = httpDurationHistogram.startTimer();  // Start timer when a request is received
  res.on('finish', () => {
    end({ method: req.method, status: res.statusCode });  // Record the request duration when finished
  });
  next();
});

// Route that simulates some latency
app.get('/', (req, res) => {
  setTimeout(() => {
    res.send('Hello, World!');
  }, Math.random() * 1000);  // Simulate random delay
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);  // Set the content type for Prometheus to scrape
  res.end(await register.metrics());  // Respond with the current metrics
});

// Start the Express server
app.listen(3000, () => {
  console.log('Express server listening on port 3000');
});
```

### **Explanation:**
- **Counter Metric (`http_requests_total`)**: This metric tracks the total number of HTTP requests made. It increments with each request and tags them by method (GET, POST) and status code (200, 500).
- **Histogram Metric (`http_duration_seconds`)**: This metric tracks the duration of HTTP requests. It records how long each request takes and stores it in buckets. The buckets are predefined ranges of time (e.g., `0.1s`, `0.5s`, `1s`).
- **Prometheus `/metrics` endpoint**: This is where Prometheus scrapes the metrics. It uses the `prom-client` library to expose the metrics in a format Prometheus understands.

### **Step 3: Run the Express Server**

Now that you have your server setup, run it with:

```bash
node server.js
```

Your Express server should be up and running, and the `/metrics` endpoint will expose the metrics that Prometheus can scrape. Visit `http://localhost:3000` to see the "Hello, World!" message, and `http://localhost:3000/metrics` to see the raw Prometheus metrics.

### **Step 4: Configure Prometheus to Scrape the Metrics**

You need to configure **Prometheus** to scrape the `/metrics` endpoint of your Express server. 

1. **Install Prometheus** if you don’t have it yet by downloading it from the [Prometheus website](https://prometheus.io/download/).
2. **Create a Prometheus configuration file** (`prometheus.yml`) with the following content:

```yaml
global:
  scrape_interval: 10s  # Scrape metrics every 10 seconds

scrape_configs:
  - job_name: 'express-server'
    static_configs:
      - targets: ['localhost:3000']
```

- `scrape_interval: 10s`: Tells Prometheus to scrape the metrics every 10 seconds.
- `targets: ['localhost:3000']`: Specifies that Prometheus should scrape metrics from the Express server at `http://localhost:3000/metrics`.

3. **Run Prometheus** with this configuration file:

```bash
./prometheus --config.file=prometheus.yml
```

Prometheus should start running on `http://localhost:9090`.

### **Step 5: Verify Prometheus is Scraping Metrics**

Open your Prometheus web UI by going to `http://localhost:9090` in your browser.

1. In the **Targets** tab (under **Status > Targets**), you should see the `express-server` target with a "last scrape" timestamp and a status of "last scrape was successful."
2. You can also run Prometheus queries directly in the **Graph** tab to see the metrics. For example, try the following queries:
   - `http_requests_total`: Shows the total number of requests.
   - `http_duration_seconds_bucket`: Shows the duration histogram.

### **Step 6: Set Up Grafana to Visualize the Metrics**

1. **Install Grafana** by following the instructions from [Grafana's official website](https://grafana.com/get).
2. **Start Grafana** by running:

```bash
sudo systemctl start grafana-server
```

3. **Log into Grafana** (default credentials: `admin`/`admin`), and set up a data source:
   - Go to **Configuration > Data Sources**.
   - Choose **Prometheus**.
   - Set the URL to `http://localhost:9090` (Prometheus instance).
   - Click **Save & Test** to confirm the connection.

4. **Create a Dashboard**:
   - Go to **Create > Dashboard**.
   - Add a new panel, select **Prometheus** as the data source, and use a query like `http_requests_total` to visualize the number of requests.
   - You can also add more panels for metrics like latency using the `http_duration_seconds_bucket` query.

### **Step 7: View Metrics in Grafana**

Once your dashboard is set up, you can view the following metrics:
- **Total requests** over time.
- **HTTP request latency** (histogram).
- **Request counts by method (GET, POST, etc.)** and status code.

### **Step 8: Monitor and Scale**

As the number of requests to your Express.js server increases, Prometheus will keep scraping metrics at the specified interval. In Grafana, you can monitor metrics such as:
- **Average request duration**
- **Request rate per method/status code**
- **Total requests over time**

This is a basic setup, but Prometheus and Grafana provide a lot of flexibility, so you can add more complex metrics and monitoring features, such as alerting, over time.
