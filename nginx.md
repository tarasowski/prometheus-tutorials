To monitor your NGINX server running on EC2 using Prometheus, you need to follow a few steps to set up monitoring and collect metrics from NGINX. Below are detailed instructions to get you started:

### Prerequisites:
1. **EC2 Instance running NGINX** – Make sure NGINX is installed and running on your EC2 instance.
2. **Prometheus and Grafana** – Prometheus should be set up to collect metrics, and Grafana should be configured for visualization (optional, but highly recommended).
3. **Prometheus NGINX Exporter** – This is a Prometheus exporter that collects NGINX metrics and exposes them in a format that Prometheus can scrape.

### Steps:

#### 1. **Install NGINX on your EC2 instance (if not already installed)**
If you haven’t installed NGINX, you can do it by following the instructions below based on your EC2 instance's OS (e.g., Ubuntu, Amazon Linux):

For **Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install nginx
```

For **Amazon Linux**:
```bash
sudo yum install nginx
```

Ensure NGINX is up and running:
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### 2. **Install Prometheus NGINX Exporter**
The **NGINX Exporter** exposes NGINX status metrics in a format Prometheus understands. Follow these steps to install it:

##### Step 2.1: **Download and Install NGINX Exporter**

On your EC2 instance, download the latest release of the NGINX exporter:

```bash
# Install dependencies
sudo apt-get install -y wget

# Download NGINX Exporter
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v0.10.0/nginx-prometheus-exporter-0.10.0-linux-amd64.tar.gz

# Extract the tar.gz file
tar -xzvf nginx-prometheus-exporter-0.10.0-linux-amd64.tar.gz

# Move the binary to /usr/local/bin
sudo mv nginx-prometheus-exporter-0.10.0-linux-amd64/nginx-prometheus-exporter /usr/local/bin/
```

##### Step 2.2: **Configure NGINX for the Prometheus Exporter**

The NGINX exporter requires the **stub_status** module, which provides the necessary metrics. Ensure your NGINX configuration includes the stub status page.

Edit the NGINX configuration file (`/etc/nginx/nginx.conf` or `/etc/nginx/sites-available/default`), and add a new location block like this:

```nginx
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;   # Allow access from localhost only
        deny all;           # Deny access from other IPs
    }
}
```

This configuration exposes the `/nginx_status` page at `127.0.0.1:8080`, which the exporter will use to scrape metrics.

##### Step 2.3: **Restart NGINX**

After modifying the configuration, restart NGINX to apply the changes:

```bash
sudo systemctl restart nginx
```

##### Step 2.4: **Run NGINX Exporter**

You can now start the NGINX exporter to collect metrics. The exporter will scrape data from the NGINX status page.

```bash
# Run the NGINX exporter
nginx-prometheus-exporter -nginx.scrape-uri=http://127.0.0.1:8080/nginx_status
```

To make it run as a service, create a systemd service file for the exporter.

Create the file `/etc/systemd/system/nginx-prometheus-exporter.service`:

```ini
[Unit]
Description=NGINX Prometheus Exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/nginx-prometheus-exporter -nginx.scrape-uri=http://127.0.0.1:8080/nginx_status
Restart=always
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl enable nginx-prometheus-exporter
sudo systemctl start nginx-prometheus-exporter
```

#### 3. **Set up Prometheus to Scrape NGINX Metrics**

Next, configure Prometheus to scrape metrics from the NGINX exporter. You need to modify the Prometheus configuration file (`prometheus.yml`).

##### Step 3.1: **Edit Prometheus Configuration**

Open `prometheus.yml` for editing. You can typically find this file in `/etc/prometheus/prometheus.yml` or wherever Prometheus is installed:

```yaml
scrape_configs:
  - job_name: 'nginx'
    static_configs:
      - targets: ['<EC2_PUBLIC_IP>:9113']
```

Here, replace `<EC2_PUBLIC_IP>` with your EC2 instance's public IP address.

##### Step 3.2: **Restart Prometheus**

Once the configuration is updated, restart Prometheus to apply the changes:

```bash
sudo systemctl restart prometheus
```

#### 4. **Verify Data Collection**

To verify if Prometheus is successfully scraping NGINX metrics:

1. Open Prometheus’ web UI (usually `http://<prometheus-server-ip>:9090`).
2. Navigate to the **Targets** page (`Status > Targets`).
3. You should see the NGINX exporter listed under **scrape_configs** with a status of `last scrape: ok`.

#### 5. **Visualize Metrics with Grafana (Optional)**

If you'd like to visualize NGINX metrics with Grafana, follow these steps:

##### Step 5.1: **Install Grafana**

If you don’t have Grafana installed, you can install it on your EC2 or another server. Follow the installation instructions for your operating system: https://grafana.com/docs/grafana/latest/installation/

##### Step 5.2: **Configure Grafana to Use Prometheus as a Data Source**

1. Log into Grafana (default URL: `http://<grafana-ip>:3000`).
2. Go to **Configuration > Data Sources**.
3. Add **Prometheus** as a data source, and enter the URL of your Prometheus server (e.g., `http://<prometheus-server-ip>:9090`).

##### Step 5.3: **Import an NGINX Dashboard**

Grafana has pre-built dashboards for NGINX metrics. To import a dashboard:

1. Go to **Create > Import**.
2. Enter the dashboard ID (e.g., **1787** for the official NGINX dashboard) or search for it.
3. Select your Prometheus data source and click **Import**.

You should now have a Grafana dashboard displaying various NGINX metrics, such as request counts, response times, and active connections.

#### 6. **Troubleshooting**

If metrics are not appearing:

- **Check the NGINX status page**: Access `http://<ec2-ip>:8080/nginx_status` to see if it shows NGINX status.
- **Prometheus scraping logs**: Check Prometheus logs for scraping errors.
- **Exporter logs**: Check the logs for `nginx-prometheus-exporter` to ensure it is running correctly.

By following these steps, you should have a full-fledged monitoring setup for your NGINX server on EC2 with Prometheus and optional Grafana dashboards for visualization.
