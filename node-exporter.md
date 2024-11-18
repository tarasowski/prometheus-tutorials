**Use dashboard:** https://grafana.com/grafana/dashboards/1860-node-exporter-full/

To set up **Node Exporter** on an **EC2 instance** to monitor system metrics (CPU, memory, disk, network, etc.) and integrate it with **Prometheus**, follow the steps below.

### **Step 1: Launch an EC2 Instance**

1. **Login to AWS Console**:
   - Go to the [AWS Console](https://aws.amazon.com/console/).
   - Navigate to **EC2** and launch a new instance. You can use the **Amazon Linux 2** AMI for simplicity.

2. **Configure Security Group**:
   - When configuring the security group, ensure to allow inbound traffic on:
     - **Port 22** for SSH access (to configure Node Exporter).
     - **Port 9100** for Prometheus to scrape Node Exporter metrics (Node Exporter runs on this port by default).

3. **SSH into the EC2 Instance**:
   Once the EC2 instance is running, SSH into it using the public IP or DNS name:

   ```bash
   ssh -i /path/to/your-key.pem ec2-user@your-ec2-public-ip
   ```

---

### **Step 2: Install Node Exporter on the EC2 Instance**

1. **Download Node Exporter**:
   - Download the latest release of **Node Exporter** from the official Prometheus GitHub releases page.

   On the EC2 instance, use the following commands:

   ```bash
   # Change to /tmp directory to download the file
   cd /tmp

   # Download Node Exporter (check for the latest version on GitHub)
   wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

   # Extract the tarball
   tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
   ```

2. **Move Node Exporter Binary to /usr/local/bin**:

   ```bash
   sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
   ```

3. **Create a Systemd Service for Node Exporter**:
   To make Node Exporter run as a service, create a **systemd** service unit.

   ```bash
   sudo vi /etc/systemd/system/node_exporter.service
   ```

   Add the following content to the file:

   ```ini
   [Unit]
   Description=Node Exporter
   Documentation=https://github.com/prometheus/node_exporter
   After=network.target

   [Service]
   User=nobody
   ExecStart=/usr/local/bin/node_exporter

   [Install]
   WantedBy=multi-user.target
   ```

4. **Start Node Exporter**:
   Now that the systemd service is set up, enable and start the Node Exporter service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable node_exporter
   sudo systemctl start node_exporter
   ```

5. **Verify Node Exporter is Running**:
   You can check if Node Exporter is running by accessing its web interface at port **9100** (default port):

   ```bash
   curl http://localhost:9100/metrics
   ```

   This should return a long list of system metrics in a Prometheus-compatible format.

---

### **Step 3: Configure Prometheus to Scrape Node Exporter Metrics**

1. **Modify Prometheus Configuration**:
   In your Prometheus configuration file (`prometheus.yml`), add a job to scrape metrics from Node Exporter running on the EC2 instance.

   Edit the `prometheus.yml` file, which is usually located in the directory where Prometheus was installed.

   ```bash
   vi /path/to/prometheus.yml
   ```

   Add the following scrape configuration:

   ```yaml
   scrape_configs:
     - job_name: 'node'
       static_configs:
         - targets: ['your-ec2-public-ip:9100']  # Replace with your EC2 public IP
   ```

   This tells Prometheus to scrape metrics from Node Exporter running on port **9100** of your EC2 instance.

2. **Restart Prometheus**:
   Restart Prometheus to apply the changes.

   ```bash
   # If running Prometheus as a service:
   sudo systemctl restart prometheus
   ```

   If you're running Prometheus manually, stop the current process and start it again using the updated `prometheus.yml`.

3. **Verify Prometheus is Scraping Node Exporter Metrics**:
   - Open the Prometheus web UI by navigating to `http://your-prometheus-server-ip:9090`.
   - Go to **Status > Targets** and you should see the `node` job with the EC2 instance listed and marked as "up" (indicating that Prometheus is successfully scraping the Node Exporter metrics).

---

### **Step 4: Set Up Grafana to Visualize Node Exporter Metrics**

1. **Install Grafana**:
   Install Grafana on your EC2 instance (or on a separate server). If you haven't installed Grafana, you can do so with the following commands:

   ```bash
   sudo yum install -y https://dl.grafana.com/oss/release/grafana-8.6.2-1.x86_64.rpm
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server
   ```

2. **Access Grafana**:
   After Grafana is installed, you can access it by going to `http://your-ec2-public-ip:3000` in your browser. The default login credentials are:
   - **Username**: `admin`
   - **Password**: `admin`

3. **Add Prometheus as a Data Source in Grafana**:
   - Go to **Configuration > Data Sources**.
   - Click **Add data source** and select **Prometheus**.
   - Set the **URL** to `http://your-prometheus-server-ip:9090` (or `http://localhost:9090` if Grafana is running on the same machine as Prometheus).
   - Click **Save & Test** to ensure the connection is successful.

4. **Import a Node Exporter Dashboard**:
   - Grafana provides pre-built dashboards for monitoring **Node Exporter** metrics. Go to **Create > Import** and enter the dashboard ID **1860** (which is the official Node Exporter dashboard from Grafana).
   - Alternatively, you can create your own custom dashboard by selecting various metrics like CPU usage, memory usage, disk I/O, and network stats.

5. **Create Panels for Node Exporter Metrics**:
   You can create custom panels to visualize various system metrics like:
   - **CPU Usage**: `avg by (mode) (rate(node_cpu_seconds_total[5m]))`
   - **Memory Usage**: `node_memory_MemTotal_bytes - node_memory_MemFree_bytes`
   - **Disk I/O**: `rate(node_disk_io_time_seconds_total[5m])`
   - **Network Usage**: `rate(node_network_receive_bytes_total[5m])`

---

### **Step 5: Set Up Alerts for Node Exporter Metrics (Optional)**

You can configure alerts in **Prometheus** or **Grafana** to notify you when certain system metrics exceed thresholds, such as high CPU usage or low memory.

#### Example: Prometheus Alert for High CPU Usage

1. **Create Alert Rule in Prometheus**:
   Add an alert rule to **Prometheus** to notify you if the CPU usage exceeds a certain threshold for a specific duration. Create or modify an alert rules file (`alert_rules.yml`):

   ```yaml
   groups:
     - name: node_alerts
       rules:
         - alert: HighCpuUsage
           expr: avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) < 0.2
           for: 5m
           labels:
             severity: critical
           annotations:
             summary: "High CPU usage detected on instance {{ $labels.instance }}"
   ```

   This rule triggers the alert if CPU idle time is less than 20% for 5 minutes, meaning the CPU is more than 80% utilized.

2. **Configure Alertmanager (Optional)**:
   If you want to receive notifications (e.g., via email, Slack, or other systems), you'll need to set up **Alertmanager** and configure Prometheus to send alerts to it.

3. **Set Alerts in Grafana**:
   You can also configure alerts directly in Grafana by clicking on a panel and selecting the **Alert** tab. For example, you could set up an alert to notify you if CPU usage exceeds a certain threshold.

---
