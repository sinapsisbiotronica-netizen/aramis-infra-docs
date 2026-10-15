# Aramis

Aramis is a self hosted server built for student and faculty projects at Instituto Tecnológico de Hermosillo. It runs workflow automation, a public status page, and a local AI model. It was built on a reused workstation, with security in mind from the start.

## Goals

The project has four goals. It gives students a real platform where they can build and show their automation and AI projects. It keeps the server hidden from the public internet while its services remain reachable. It runs on modest, reused hardware with no licensing cost. And it documents everything so the setup can be maintained and handed over to someone else.

## Hardware

The server is a Dell Precision Tower 5810 with an Intel Xeon E5 1620 v4 processor (4 cores and 8 threads), 16 GB of ECC DDR4 memory, and a 232 GB NVMe drive. It runs Ubuntu 26.04.1 LTS. The installed graphics card is an NVIDIA Quadro K420, which is too old for AI workloads, so the AI model runs on the processor instead.

## Deployment

The server sits on the campus network with a fixed address assigned by the IT department. The network settings live in a configuration file tied to the built in network port, so the address stays the same after every restart.

When the server first arrived on campus, the network sent all traffic through a proxy and blocked direct name lookups. Most things worked through the proxy, but the Cloudflare tunnel did not, because it needs a type of name lookup that a proxy cannot carry. The network administrator was asked for help, and the server was given direct internet access. The old proxy settings are still kept in the configuration files, turned off, so they can be restored quickly if the network policy changes again.

The main lesson from this is simple. On a restricted network, test name resolution on its own, not just web access. A proxy can let web pages through and still leave a service broken.

## Architecture

```mermaid
flowchart LR
    U[Users] --> CF[Cloudflare edge<br/>DDoS protection, WAF, TLS]
    CF --> T[Cloudflare Tunnel<br/>outbound only]

    subgraph A[Aramis<br/>campus network, no inbound ports]
        T --> N[n8n<br/>workflow automation]
        T --> K[Uptime Kuma<br/>public status page]
        N --> DB[(PostgreSQL<br/>internal only)]
        N --> O[Ollama on host<br/>Llama 3.2 3B / 1B, CPU only]
        SP[Docker socket proxy] -.->|guards Docker API| N
    end

    ADM[Administrators] -->|Tailscale mesh VPN<br/>SSH bound to this interface| A
    A -->|Encrypted, deduplicated| BK[(Offsite cloud backup)]
```

### How traffic reaches the server

Public visitors never connect to the server directly. They reach Cloudflare first, and Cloudflare passes their requests through a tunnel that the server opens from the inside. Because of this, the server's address is never exposed and no inbound ports are open on the network.

Administrators connect through a private Tailscale VPN using key based login. Remote login is only allowed on that VPN connection, so it cannot be reached from the campus network, not even from machines in the same room.

## Services

n8n runs the workflow automation that students use for their projects, and it is publicly reachable through Cloudflare. Uptime Kuma provides a public status page, also through Cloudflare. PostgreSQL stores the data for n8n and can only be reached from inside the server. Ollama runs the local AI model, so no data ever leaves the server. A Docker socket proxy limits what the containers are allowed to do with Docker itself.

Every service except Ollama runs in its own Docker container. Each group of services has its own isolated network, every container has limits on how much processor time and memory it can use, and all of them are managed from a single configuration file, which keeps updates predictable. All file paths in that configuration are written out in full, so the services behave the same no matter which user starts them.

### Local AI

Ollama runs directly on the server rather than in a container, and n8n talks to it over an internal connection. Students use it through a chat workflow inside n8n, so the AI service itself is never exposed to the network.

Two versions of the Llama 3.2 model are available. The larger 3B model gives better answers, and the smaller 1B model responds about three times faster. Since everything runs on the processor, answers are slow and requests are handled one at a time. This works well for learning and demonstrations, but it will not keep up with a full class using it at once, and anyone using it should know that.

## Security

Several layers protect the server. At the edge, Cloudflare handles attack mitigation, bot protection, and a web application firewall. The Cloudflare tunnel hides the server and removes the need for any open inbound ports. Cloudflare also enforces strict encryption, with HSTS, a minimum of TLS 1.2, and support for HTTP/3.

On the server itself, the UFW firewall blocks all inbound traffic by default and only allows remote login over the VPN. Login works with keys only, and both passwords and direct root login are turned off. Fail2ban blocks addresses that fail to log in repeatedly, and CrowdSec adds shared community block lists and behavior detection. Security updates install automatically.

The containers are kept apart from each other, and the socket proxy limits the damage a compromised container could do. Every month, Lynis audits how well the system is hardened and Trivy scans the container images for known vulnerabilities.

### Known risks

n8n is the most exposed part of the system, since it is public and it runs code written by its users. To reduce that risk, only trusted users are allowed to use its code features. Vulnerabilities in the container images are reviewed every month and fixed as soon as updates are available.

## Backups

Backups are made with Restic and sent to offsite cloud storage through Rclone. They are encrypted and deduplicated, so they stay small over time.

The backups cover service settings, application data, system configuration, and the credentials needed to run the backups themselves. Before each backup, the n8n database is exported to a file so its data can be restored cleanly. Container images are not included because they can always be downloaded again.

A backup runs every night, followed by a cleanup that keeps seven daily, four weekly, and three monthly copies. With these backups alone, the services can be rebuilt on new hardware.

## Operations

Once a month, the server goes through a maintenance routine. Every container image is scanned for vulnerabilities, images are updated and old ones are removed, the hardening audit is run and compared against a target score, the threat detection numbers are reviewed, and the backups are checked.

A set of health check scripts reports on the containers, system services, public access, the backup connection, and available disk space and memory. A separate connectivity script checks every outside service the server depends on, so any change in the network policy shows up right away instead of breaking something quietly.
