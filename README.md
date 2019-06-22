# nginx-loadbalancer
A simple NGINX as HTTP Load Balancer

## Prerequisites

You need docker installed

## Usage

Swarm:

```bash
docker stack deploy -c docker-compose.yml lb
```

Compose:

```bash
docker-compose up -d
```

### Expose ports

Prometheus: 9090 \
Grafana: 3000 \
Nginx Load Balancer: 80

### Grafana login

user: admin \
password: nginxlb