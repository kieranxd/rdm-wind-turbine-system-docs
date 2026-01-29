# E-Ink Display IoT System Documentation

Welcome to the technical documentation for the E-Ink Display IoT System at the RDM campus wind turbine. This guide covers the complete setup, configuration, and maintenance of the digital infrastructure that powers the turbine monitoring and visualization system.

!!! info "About This Documentation"
    This manual focuses exclusively on the **installation, configuration, and maintenance of the technical systems** that collect, store, and display wind turbine data. It is intended for system administrators and technical staff responsible for keeping the IoT infrastructure operational.

## Project Context

The broader project aims to increase visibility of the wind turbine's IoT capabilities through both digital and physical means:

**Physical Installation:**
- A sustainable display installation at the turbine location featuring the e-ink screen
- Integration of displays showing turbine data in an attractive and understandable format
- Different kind of art designs to replace the current fence at the turbine's location

**Digital Solution:**
- A simple way of providing access to the same data and visualizations online
- Docker-based infrastructure for scalability and management
- Integration between online and physical presentations

**This Documentation Covers:**

This manual documents the **digital infrastructure** that makes the project possible - the IoT stack, databases, visualization tools, networking, and automation that collect and display the turbine data. It provides step-by-step instructions for installing, configuring, and maintaining these systems.

## System Overview

The system uses a layered architecture:

- **Virtualization Layer**: Proxmox host running a Linux VM
- **Application Layer**: Docker containers (Node-RED, InfluxDB, Grafana, Image Renderer)
- **Bridge Layer**: Raspberry Pi 4 in USB gadget mode
- **Display Layer**: Samsung EM32DX e-ink display 
- **Networking**: Tailscale VPN connecting VM and Raspberry Pi

For detailed architecture diagrams and data flow, see the [Architecture](architecture/overview.md) section.

### Key Components

- **Proxmox Host**: Virtualization platform hosting the IoT stack
- **IOTstack**: Docker-based IoT services (InfluxDB, Grafana, Node-RED, Portainer)
- **Grafana Image Renderer**: Custom service for rendering dashboard snapshots
- **Raspberry Pi**: USB gadget mode mass storage device
- **Samsung EM32DX**: E-ink display for data display

### System Capabilities

- Real-time wind turbine sensor monitoring (via eGauge)
- Weather data integration (nearby weather station API)
- Data storage in InfluxDB time-series database
- Visual dashboard in Grafana
- Automated image rendering and delivery to e-ink display
- USB mass storage display method via Raspberry Pi

## Quick Start

1. **New Installation**: Start with [Proxmox Host Setup](infrastructure/proxmox.md)
2. **Maintenance**: See [Regular Tasks](maintenance/regular-tasks.md)
3. **Troubleshooting**: Check [Troubleshooting Guide](maintenance/troubleshooting.md)

## Documentation Structure

This manual is organized into logical sections:

- **Architecture**: Understand how the system works
- **Infrastructure Setup**: Set up the virtualization and IoT stack
- **Docker Services**: Configure individual services
- **Raspberry Pi**: Set up the USB gadget mode bridge
- **Hardware**: Physical setup and connections
- **Maintenance**: Keep the system running smoothly

## Support & Contact

For questions or issues, refer to the troubleshooting section or contact the system administrator.

---

*Documentation created: January 2026*
