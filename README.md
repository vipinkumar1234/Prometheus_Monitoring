# Prometheus Monitoring

A practical Prometheus and Grafana monitoring setup for Linux servers and AWS EC2 instances. This repository includes Docker Compose files, systemd installation scripts, Prometheus configuration examples, and a Node Exporter service.

> This repository is maintained by **Vipin Kumar** for the **World of AWS YouTube channel**.

## What is included

- `docker/`: Runs Prometheus and Grafana with Docker Compose.
- `prometheus.yml`: Basic Prometheus configuration that scrapes Prometheus itself.
- `prometheus_ec2.yml`: Discovers EC2 instances in a region and scrapes Node Exporter.
- `prometheus_serviceDiscovery.yml`: EC2 service discovery example.
- `prometheus_relabeeling.yml`: EC2 discovery with relabeling examples.
- `install-prometheus.sh`: Installs Prometheus as a systemd service on Linux.
- `install-grafana.sh`: Installs Grafana on Debian or Ubuntu.
- `install-node-exporter.sh`: Installs Node Exporter as a systemd service.
- `node-exporter-init.dservice/`: An alternative Node Exporter init.d setup.

## Prerequisites

### Docker setup

- Docker Engine
- Docker Compose v2 (`docker compose`)
- A Linux, macOS, or Windows environment capable of running Docker

### Host installation

- A Debian or Ubuntu-based Linux server
- `sudo` access
- An `x86_64`/`amd64` system with `wget`
- Network access to download Prometheus, Grafana, and Node Exporter

The installation scripts contain pinned software versions. Review and update those versions before using this repository in production.

## Option 1: Run with Docker Compose

From the repository root, start both services:

```bash
cd docker
docker compose up -d
```

Check the containers:

```bash
docker compose ps
docker compose logs -f prometheus
```

Open the web interfaces:

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

Grafana data is stored in the named Docker volume `grafana-storage`. Stop the services without deleting the data with:

```bash
docker compose down
```

To stop the services and delete the Grafana volume:

```bash
docker compose down -v
```

### Add targets to the Docker setup

Edit `docker/prometheus/prometheus.yml` and add a scrape job. For example:

```yaml
scrape_configs:
	- job_name: "node_exporter"
		static_configs:
			- targets: ["SERVER_PRIVATE_OR_PUBLIC_IP:9100"]
```

Apply configuration changes by restarting Prometheus:

```bash
docker compose restart prometheus
```

Ensure TCP port `9100` is reachable from the Prometheus host and restricted with security groups or a firewall.

## Option 2: Install services directly on Linux

Run the scripts from the repository root on the server where each service should be installed:

```bash
chmod +x install-prometheus.sh install-grafana.sh install-node-exporter.sh
./install-prometheus.sh
./install-grafana.sh
./install-node-exporter.sh
```

The scripts install services with these default endpoints:

- Prometheus: http://SERVER_IP:9090
- Grafana: http://SERVER_IP:3000
- Node Exporter metrics: http://SERVER_IP:9100/metrics

Verify service status:

```bash
sudo systemctl status prometheus
sudo systemctl status grafana-server
sudo systemctl status node-exporter
```

View logs when a service does not start:

```bash
sudo journalctl -u prometheus -e
sudo journalctl -u grafana-server -e
sudo journalctl -u node-exporter -e
```

After installation, Prometheus reads `/etc/prometheus/prometheus.yml`. Copy an appropriate configuration there, validate it, and restart Prometheus:

```bash
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

## Configure targets

### Static targets

Use `prometheus.yml` as a starting point for monitoring Prometheus itself. To monitor Node Exporter on another server, add its address under `static_configs`:

```yaml
scrape_configs:
	- job_name: "node_exporter"
		static_configs:
			- targets: ["SERVER_IP:9100"]
```

Copy the finished file to the Prometheus server and restart Prometheus:

```bash
sudo cp prometheus.yml /etc/prometheus/prometheus.yml
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

### AWS EC2 service discovery

Use `prometheus_ec2.yml`, `prometheus_serviceDiscovery.yml`, or `prometheus_relabeeling.yml` when Prometheus should discover EC2 instances automatically. Before using them:

1. Set the correct AWS region.
2. Make sure Node Exporter is running on each target instance.
3. Allow Prometheus to reach TCP port `9100` in the target security group.
4. Give Prometheus an IAM role with the minimum required EC2 read permissions, such as `ec2:DescribeInstances`.

Do not commit AWS access keys or secret keys to a configuration file. Prefer an IAM role, environment variables, or another secret-management solution. The `access_key` and `secret_key` values in the examples are placeholders and must be replaced or removed.

## Grafana setup

1. Open Grafana at http://localhost:3000 or `http://SERVER_IP:3000`.
2. Sign in and change the default password when prompted.
3. Add Prometheus as a data source using:
	 - Docker Compose: `http://prometheus:9090`
	 - Separate hosts: `http://PROMETHEUS_SERVER_IP:9090`
4. Import or create dashboards for Prometheus and Node Exporter metrics.

## Troubleshooting

- **No data in Prometheus:** Open `Status > Target health` and check the target error message.
- **Node Exporter is down:** Check `sudo systemctl status node-exporter` and confirm port `9100` is open.
- **EC2 targets are missing:** Check the AWS region, IAM permissions, and the Prometheus logs.
- **Grafana cannot connect to Prometheus:** Use `http://prometheus:9090` from the Docker network, not `localhost:9090`.
- **Configuration errors:** Run `promtool check config /etc/prometheus/prometheus.yml` before restarting Prometheus.

## Security notes

- Restrict ports `3000`, `9090`, and `9100` to trusted networks.
- Do not expose Prometheus or Node Exporter directly to the public internet without access controls.
- Do not store AWS credentials in Git.
- Change default Grafana credentials and keep packages updated.

## License

No license file is currently included. Add a license before distributing or reusing this repository publicly.
