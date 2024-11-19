## Monitoring Architecture

- (A) ec2 w/ express (server.js) + promtail + node_exporter
- (D) ec2 w/ loki + prometheus 
- (G) ec2 w/ grafana + data connector to (D) -> (dashboard import)
