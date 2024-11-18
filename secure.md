To secure the Prometheus `/metrics` endpoint with Basic Authentication, you'll need to add an authentication layer to the service exposing the metrics. This could be an Express.js application or an NGINX server acting as a reverse proxy to the Prometheus metrics endpoint. I'll describe both methods below in detail.

### **Option 1: Secure Prometheus Metrics Endpoint in Express.js with Basic Authentication**

If you're exposing Prometheus metrics from an Express.js application, you can use the `express-basic-auth` middleware to implement Basic Authentication. This middleware will require users to provide a username and password before they can access the `/metrics` endpoint.

#### **Steps:**

1. **Install the `express-basic-auth` Middleware:**
   First, you'll need to install the `express-basic-auth` module.

   ```bash
   npm install express-basic-auth
   ```

2. **Set Up Basic Authentication in Express.js:**

   Modify your Express.js application to use the `express-basic-auth` middleware on the `/metrics` route. This ensures that users must authenticate themselves before accessing the metrics.

   Here's how you can secure the `/metrics` endpoint:

   ```javascript
   const express = require('express');
   const promClient = require('prom-client');
   const basicAuth = require('express-basic-auth');
   const app = express();

   // Create a new registry
   const register = new promClient.Registry();

   // Create a counter metric
   const counter = new promClient.Counter({
     name: 'express_requests_total',
     help: 'Total number of requests',
   });
   register.registerMetric(counter);

   // Middleware to increment counter on each request
   app.use((req, res, next) => {
     counter.inc();
     next();
   });

   // Apply basic authentication on the /metrics route
   app.use('/metrics', basicAuth({
     users: { 'admin': 'password' }, // Replace 'admin' and 'password' with your credentials
     challenge: true, // Prompt for credentials if not provided
   }), async (req, res) => {
     res.set('Content-Type', register.contentType);
     res.end(await register.metrics());
   });

   app.listen(3000, () => {
     console.log('Express server listening on port 3000');
   });
   ```

   **Explanation of the Code:**
   - **`basicAuth` middleware**: It adds basic authentication to the `/metrics` endpoint. The `users` object contains the username and password pairs, where `'admin'` is the username and `'password'` is the password.
   - **`challenge: true`**: If the user doesn't provide the correct credentials, this option will prompt for credentials using a basic authentication dialog in the browser or HTTP client.
   - **Metrics endpoint**: Once the user is authenticated, the `/metrics` endpoint returns the Prometheus metrics.

3. **Testing the Setup:**
   - To test, you can open a browser or use `curl` to access the metrics endpoint. When accessing `http://<EC2_PUBLIC_IP>:3000/metrics`, you should be prompted for the username and password you configured (`admin` and `password` in this example).
   - For `curl`, use the following command:
     ```bash
     curl -u admin:password http://<EC2_PUBLIC_IP>:3000/metrics
     ```

   If the credentials are correct, Prometheus metrics will be returned.

---

### **Option 2: Secure Prometheus Metrics Endpoint in NGINX with Basic Authentication**

If you're using NGINX as a reverse proxy for your Prometheus metrics (e.g., for NGINX Exporter or Express.js), you can secure the `/metrics` endpoint by enabling Basic Authentication in NGINX.

#### **Steps:**

1. **Install Apache HTTP Server Utils to Generate Password File:**
   You will need to create a `.htpasswd` file that contains the username and password. On an Ubuntu-based system, install `apache2-utils` to use the `htpasswd` command:

   ```bash
   sudo apt-get install apache2-utils
   ```

2. **Create the `.htpasswd` File:**
   Use the `htpasswd` command to create a password file.

   ```bash
   sudo htpasswd -c /etc/nginx/.htpasswd admin
   ```

   Replace `admin` with the desired username. The system will prompt you to enter and confirm a password.

3. **Configure NGINX for Basic Authentication:**
   Update your NGINX configuration to protect the `/metrics` endpoint with Basic Authentication.

   Open your NGINX configuration file (typically located at `/etc/nginx/nginx.conf` or `/etc/nginx/sites-available/default`) and add the following configuration to the server block:

   ```nginx
   server {
       listen 443 ssl;
       server_name your_domain;  # Replace with your domain or public IP

       ssl_certificate /path/to/certificate.pem;
       ssl_certificate_key /path/to/private_key.pem;

       # Protect the /metrics endpoint with Basic Authentication
       location /metrics {
           auth_basic "Restricted Access";  # Prompt message
           auth_basic_user_file /etc/nginx/.htpasswd;  # Path to .htpasswd file
           proxy_pass http://localhost:9113;  # Pass the request to NGINX exporter or your application
       }
   }
   ```

   **Explanation of the Configuration:**
   - **`auth_basic "Restricted Access"`**: This directive enables Basic Authentication and displays the message `"Restricted Access"` when the user is prompted to enter credentials.
   - **`auth_basic_user_file /etc/nginx/.htpasswd`**: This specifies the location of the `.htpasswd` file that contains the username and password pairs. You should replace `/etc/nginx/.htpasswd` with the path where you created the `.htpasswd` file.
   - **`proxy_pass http://localhost:9113`**: This forwards requests to the NGINX Exporter (or any other backend service exposing Prometheus metrics). You can replace this with your Express.js or other service.

4. **Test the Setup:**
   - Once you restart NGINX (`sudo systemctl restart nginx`), you can try accessing the `/metrics` endpoint.
   - If you're accessing it through a browser, you'll be prompted to enter the username and password.
   - You can also test with `curl`:

     ```bash
     curl -u admin:password https://your_domain/metrics
     ```

   Replace `admin` and `password` with the username and password you set up, and `your_domain` with your domain or EC2 public IP address.

---

### **Security Considerations:**
1. **Use HTTPS**: Basic Authentication transmits credentials in an encoded format (not encrypted). To protect your credentials from being intercepted, ensure that you access the `/metrics` endpoint over HTTPS. This is important, especially if your Prometheus metrics contain sensitive data or are accessible over the internet.
2. **Strong Passwords**: Use strong and unique passwords for Basic Authentication to avoid brute force attacks.
3. **Limit Access**: If possible, limit access to the `/metrics` endpoint by IP address, so only your Prometheus server can scrape the metrics.

### **Conclusion:**

Securing your Prometheus metrics endpoint with Basic Authentication is an essential step to ensure that only authorized users can access the sensitive monitoring data. The two approaches—using Express.js middleware (`express-basic-auth`) and configuring NGINX as a reverse proxy—both work well depending on your architecture. Be sure to also enable HTTPS to prevent password interception during transmission.
