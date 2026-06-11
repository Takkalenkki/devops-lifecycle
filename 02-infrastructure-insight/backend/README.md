# Metrics App

This directory contains backend API written in Go. It is designed to fetch information from the App server VM, and exposes the `/metrics` endpoint for the Web server (frontend) VMs to fetch for displaying various system metrics.

## Backend API

A lightweight HTTP server that collects and exposes real-time Linux system metrics in JSON format.

### Project Structure

```sh
.
├── Dockerfile      # Dockerfile for containerization
├── go.mod
├── go.sum
├── info
│   └── info.go     # System metrics collection
├── main.go         # Main program
├── README.md       # This document
└── structs
    └── structs.go  # Metrics structs
```

## Requirements

* Go 1.24.6 or higher.
* VM environment set up using the [previous task](https://gitea.kood.tech/akiheiskanen/server-sorcery-101).

## Features

The API collects the following system information:

- Hostname: System hostname from environment
- Kernel Information: Kernel name, release, version, and operating system
- CPU Metrics:
    - Model name and vendor
    - Clock speed (MHz)
    - Number of cores and siblings
    - Current CPU usage and free percentage
- Memory (RAM):
    - Total, free, in-use, and available memory (MB/GB)
    - Swap memory statistics
- System Uptime: Hours, minutes, seconds, and formatted display
- Disk Usage: Total, free, used space and usage percentage for root filesystem
- Network Statistics: Receive/transmit bytes per second for each network interface

## Installation

```sh
go mod download
go build -o metrics-app
```

### Running the server

```sh
./metrics-app
```

## Docker Deployment

The application is designed to be containerized and deployed to the backend VM (App server)

### Building the Docker Image

```sh
sudo docker build -t backend-app:latest .
```

### Deploying to App Server

Refer to [scripts/README](../scripts/README.md) for more instructions on deploying.

## API Endpoint

### Get `/metrics`

Returns a JSON object containing the system metrics.

### Example response:

```json
{
  "hostname": "",
  "kernel_info": {
    "kernel_name": "Linux",
    "kernel_release": "6.17.8-arch1-1",
    "kernel_version": "#1 SMP PREEMPT_DYNAMIC Fri, 14 Nov 2025 06:54:20 +0000",
    "operating_system": "GNU/Linux"
  },
  "cpu_info": {
    "model_name": "AMD Ryzen 7 3700X 8-Core Processor",
    "vendor_id": "AuthenticAMD",
    "mhz": "3974.841",
    "cores": 16,
    "siblings": 256,
    "usage_pct": "7.49",
    "free_pct": "92.51"
  },
  "ram": {
    "total_mb": 32768,
    "total_gb": 32,
    "free_mb": 571,
    "free_gb": 0.571,
    "in_use_mb": 32197,
    "in_use_gb": 32.197,
    "available_mb": 17661,
    "available_gb": 17.661,
    "swap_total_mb": 524,
    "swap_total_gb": 0.524,
    "swap_free_mb": 524,
    "swap_free_gb": 0.524,
    "swap_used_mb": 0,
    "swap_used_gb": 0
  },
  "uptime": {
    "hours": 5,
    "minutes": 14,
    "seconds": 44,
    "formatted": "5h 14m 44s",
    "total_seconds": 18884
  },
  "disk": [
    {
      "path": "/",
      "total_mb": 222241,
      "free_mb": 59371,
      "used_mb": 162870,
      "used_pct": 73.28530739152541
    }
  ],
  "net_info": [
    {
      "name": "lo",
      "rx_bps": 323127,
      "tx_bps": 323127
    },
    {
      "name": "enp6s0",
      "rx_bps": 466745814,
      "tx_bps": 21274580
    },
    {
      "name": "virbr0",
      "rx_bps": 7882385,
      "tx_bps": 109697827
    },
    {
      "name": "docker0",
      "rx_bps": 84,
      "tx_bps": 260
    }
  ]
}
```

## Notes

* **As the app is designed to be containerized and deployed to the App server VM, not all info might be accurate if running the app locally.**
* CPU Usage: Calculated as delta between consecutive calls. First call returns 0% as baseline is established.
* Network Speed: Measured in bytes per second as delta between calls. First call returns null while baseline is established.
* Error Handling: Returns HTTP 500 with error message if metrics collection fails.
* HOSTNAME Variable: Must be set in the environment for hostname to appear in metrics.