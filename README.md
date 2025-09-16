# Jaeger Installation Guide

End-to-end distributed tracing system for monitoring and troubleshooting complex microservices architectures. Essential tool for understanding request flows and performance bottlenecks.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Linux system (any modern distribution)
- Root or sudo access
- 4GB RAM minimum, 8GB+ recommended for production
- Storage backend (Elasticsearch, Cassandra, or Kafka)
- Network connectivity for trace collection


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### All-in-One Deployment (Development)
```bash
# Download Jaeger all-in-one binary
JAEGER_VERSION="1.51.0"
cd /tmp
wget https://github.com/jaegertracing/jaeger/releases/download/v${JAEGER_VERSION}/jaeger-${JAEGER_VERSION}-linux-amd64.tar.gz
tar -xzf jaeger-${JAEGER_VERSION}-linux-amd64.tar.gz

# Install Jaeger
sudo cp jaeger-${JAEGER_VERSION}-linux-amd64/jaeger-all-in-one /usr/local/bin/
sudo chmod +x /usr/local/bin/jaeger-all-in-one

# Create jaeger user
sudo useradd --system --shell /bin/false jaeger
sudo mkdir -p /var/lib/jaeger /var/log/jaeger
sudo chown -R jaeger:jaeger /var/lib/jaeger /var/log/jaeger

# Create systemd service
sudo tee /etc/systemd/system/jaeger.service > /dev/null <<EOF
[Unit]
Description=Jaeger Distributed Tracing
After=network.target

[Service]
Type=simple
User=jaeger
Group=jaeger
ExecStart=/usr/local/bin/jaeger-all-in-one \
  --memory.max-traces=10000 \
  --log-level=info
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now jaeger

# Access UI at http://localhost:16686
```

### Docker Installation (Production)
```bash
# Create Jaeger stack with Elasticsearch
mkdir -p ~/jaeger

cat > ~/jaeger/docker-compose.yml <<EOF
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:7.17.15
    container_name: jaeger-elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "127.0.0.1:9200:9200"
    networks:
      - jaeger

  jaeger-collector:
    image: jaegertracing/jaeger-collector:latest
    container_name: jaeger-collector
    restart: unless-stopped
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_NUM_SHARDS=1
      - ES_NUM_REPLICAS=0
    ports:
      - "14268:14268"
      - "14250:14250"
    depends_on:
      - elasticsearch
    networks:
      - jaeger

  jaeger-agent:
    image: jaegertracing/jaeger-agent:latest
    container_name: jaeger-agent
    restart: unless-stopped
    environment:
      - REPORTER_GRPC_HOST_PORT=jaeger-collector:14250
    ports:
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
    depends_on:
      - jaeger-collector
    networks:
      - jaeger

  jaeger-query:
    image: jaegertracing/jaeger-query:latest
    container_name: jaeger-query
    restart: unless-stopped
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
    ports:
      - "127.0.0.1:16686:16686"
    depends_on:
      - elasticsearch
    networks:
      - jaeger

networks:
  jaeger:
    driver: bridge

volumes:
  elasticsearch_data:
EOF

docker-compose up -d
```

### Kubernetes Installation
```bash
# Install Jaeger Operator
kubectl create namespace observability
kubectl create -n observability -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.50.0/jaeger-operator.yaml

# Deploy Jaeger instance
cat > jaeger-instance.yaml <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        num-shards: 1
        num-replicas: 0
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - jaeger.example.com
EOF

kubectl apply -f jaeger-instance.yaml
```

## 4. Configuration and Integration

### Application Instrumentation
```bash
# Example: Instrument Node.js application
npm install --save jaeger-client opentracing

# Create tracing configuration
cat > tracing.js <<EOF
const opentracing = require('opentracing');
const jaeger = require('jaeger-client');

// Jaeger configuration
const config = {
  serviceName: 'my-nodejs-app',
  reporter: {
    agentHost: 'localhost',
    agentPort: 6832,
  },
  sampler: {
    type: 'probabilistic',
    param: 1.0, // Sample 100% of traces
  },
};

const options = {
  logger: {
    info: msg => console.log('JAEGER INFO:', msg),
    error: msg => console.error('JAEGER ERROR:', msg),
  },
};

const tracer = jaeger.initTracer(config, options);
opentracing.setGlobalTracer(tracer);

module.exports = tracer;
EOF
```

## Additional Resources

- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Distributed Tracing Guide](https://opentracing.io/guides/)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.