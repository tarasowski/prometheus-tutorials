const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = promClient.register;

// Create a counter metric for HTTP requests
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'status']
});

// Create a histogram for response durations
const httpDurationHistogram = new promClient.Histogram({
  name: 'http_duration_seconds',
  help: 'Histogram of HTTP request durations in seconds'
});

// Middleware to count HTTP requests
app.use((req, res, next) => {
  res.on('finish', () => {
    httpRequestsTotal.labels(req.method, res.statusCode).inc();
  });
  next();
});

// Middleware to measure request duration
app.use((req, res, next) => {
  const end = httpDurationHistogram.startTimer();
  res.on('finish', () => {
    end({ method: req.method, status: res.statusCode });
  });
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Basic route
app.get('/', (req, res) => {
  res.send('Hello, World!');
});

app.listen(3000, () => console.log('Express server listening on port 3000'));
