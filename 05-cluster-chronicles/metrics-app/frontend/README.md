# Metrics Dashboard Frontend

This directory contains the frontend web application for the Metrics Dashboard. It is a React-based single-page application that fetches and displays real-time system metrics from the backend API in an intuitive, visual interface.

## Frontend Application

A responsive dashboard built with React and Chart.js that provides real-time visualization of Linux system metrics.

### Project Structure

```sh
.
├── Dockerfile           # Multi-stage Docker build
├── eslint.config.js     # ESLint configuration
├── index.html           # HTML entry point
├── nginx.conf           # Nginx server configuration
├── package.json         # npm dependencies and scripts
├── package-lock.json    # Locked dependency versions
├── README.md            # This document
├── src
│   ├── App.jsx          # Main application component
│   ├── index.css        # Global styles and Tailwind imports
│   └── main.jsx         # React entry point
└── vite.config.js       # Vite configuration
```

## Features

The dashboard displays the following information:
- OS Information: Hostname, kernel details, and operating system
- CPU Metrics: Vendor, model, cores, frequency, and real-time usage
- Memory Visualization:
    - Doughnut chart showing RAM usage percentage
    - Line chart tracking RAM usage over time
    - Bar chart with toggleable MB/GB units
    - Swap memory statistics
- System Uptime: Formatted uptime display and total seconds
- Disk Usage:
    - Usage statistics per mount point
    - Bar chart showing usage percentage
    - Doughnut chart for visual representation
- Network Traffic: Real-time incoming/outgoing bandwidth per interface

## Technology Stack

- React 18 - UI framework
- Vite - Build tool and dev server
- Chart.js - Data visualization library
- react-chartjs-2 - React wrapper for Chart.js
- Tailwind CSS - Utility-first CSS framework
- Nginx - Production web server

## Requirements

- Node.js 20 or higher
- npm or yarn

## Installation

```sh
npm install
```

### Development

```sh
npm run dev
```

The application will be available at `http://localhost:5173`

### Building for Production

```sh
npm run build
```

This creates an optimized production build in the `dist/` directory.

## Docker Deployment

The application is designed to be containerized and deployed to the frontend pod on local Kubernetes cluster

### Building the Docker Image

```sh
sudo docker build -t frontend-app:latest .
```

### Configuration

The Nginx configuration (`nginx.conf`) includes:
- Reverse proxy to backend API at `/api/metrics`
- Server info endpoint at `/server-info`
- Static file serving for the React application

Make sure to update the proxy settings in `nginx.conf` if your backend API is running on a different host or port.

### API Integration

The frontend expects the following endpoints:

- `GET /api/metrics` - Returns system metrics JSON (proxied to backend)
- `GET /server-info` - Returns the server hostname

Metrics are fetched every 1 second to provide real-time updates.

## Features in Detail

### Interactive Charts

- RAM Doughnut Chart: Visual breakdown of free vs used memory
- RAM Line Chart: Historical view of memory usage (last 10 data points)
- RAM Bar Chart: Click to toggle between MB and GB units
- Disk Bar Chart: Shows disk usage percentage
- Disk Doughnut Chart: Visual representation of disk space

### Responsive Design

- Grid layout adapts to screen size
- Mobile-friendly interface
- Scrollable sections for overflow content

### Real-time Updates

- Automatic refresh every second
- Maintains history of last 10 metric snapshots
- Loading state during initial fetch

## Notes

- Update Frequency: Dashboard refreshes every 1000ms (1 second)
- History Length: Charts display the last 10 data points
- Error Handling: Console logs errors if metrics fetch fails
- Browser Support: Modern browsers with ES6+ support required
- Docker Build: Uses Node 20 Alpine for building and Nginx 1.27 Alpine for serving