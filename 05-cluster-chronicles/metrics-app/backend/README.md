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

The application is designed to be containerized and deployed to the Kubernetes cluster.

### Building the Docker Image

```sh
sudo docker build -t backend-app:latest .
```

## API Endpoint

### Get `/metrics`

Returns a JSON object containing the system metrics.

### Example response:

```json
{
  "hostname": "backend-5c8cd8bf85-42lg5",
  "kernel_info": {
    "kernel_name": "Linux",
    "kernel_release": "6.6.95",
    "kernel_version": "#1 SMP PREEMPT_DYNAMIC Tue Jan 27 22:55:23 UTC 2026",
    "operating_system": "Linux"
  },
  "cpu_info": {
    "model_name": "AMD Ryzen 7 3700X 8-Core Processor",
    "vendor_id": "AuthenticAMD",
    "mhz": "3599.998",
    "cores": 1,
    "threads": 4,
    "usage_pct": "0.00",
    "free_pct": "100.00"
  },
  "ram": {
    "total_mb": 12240,
    "total_gb": 12,
    "free_mb": 1575,
    "free_gb": 1.575,
    "in_use_mb": 10665,
    "in_use_gb": 10.665,
    "available_mb": 6336,
    "available_gb": 6.336,
    "swap_total_mb": 0,
    "swap_total_gb": 0,
    "swap_free_mb": 0,
    "swap_free_gb": 0,
    "swap_used_mb": 0,
    "swap_used_gb": 0
  },
  "uptime": {
    "hours": 6,
    "minutes": 19,
    "seconds": 35,
    "formatted": "6h 19m 35s",
    "total_seconds": 22775
  },
  "disk": [
    {
      "path": "/",
      "total_mb": 17318,
      "free_mb": 6385,
      "used_mb": 10933,
      "used_pct": 63.13084651807368
    }
  ],
  "net_info": null,
  "app_version": "",
  "app_commit_msg": ""
}
```

## Notes

* **As the app is designed to be containerized and deployed to local Kubernetes cluster, not all info might be accurate if running the app locally.**
* CPU Usage: Calculated as delta between consecutive calls. First call returns 0% as baseline is established.
* Network Speed: Measured in bytes per second as delta between calls. First call returns null while baseline is established.
* Error Handling: Returns HTTP 500 with error message if metrics collection fails.
* HOSTNAME Variable: Must be set in the environment for hostname to appear in metrics.