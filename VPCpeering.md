# AWS VPC Peering Architecture

## Overview

This architecture connects two AWS Virtual Private Clouds (VPCs) in different AWS Regions using **VPC Peering**.

- **VPC 1 – N. Virginia (us-east-1):** Hosts the application backend and frontend.
- **VPC 2 – Mumbai (ap-south-1):** Hosts the database.
- **VPC Peering:** Provides private network connectivity between the two VPCs using their private IP addresses.

## Architecture

```text
                 AWS VPC Peering
        ┌───────────────────────────────┐
        │                               │
┌───────▼────────────────────┐   ┌──────▼─────────────────┐
│ VPC 1 – N. Virginia        │   │ VPC 2 – Mumbai         │
│                            │   │                        │
│  ┌──────────────────────┐  │   │  ┌───────────────────┐ │
│  │ Private Subnet       │  │   │  │ Private Subnet    │ │
│  │ AZ 1                 │  │   │  │ AZ 1              │ │
│  │ Backend              │  │   │  │ Database          │ │
│  └──────────────────────┘  │   │  └───────────────────┘ │
│                            │   │                        │
│  ┌──────────────────────┐  │   │                        │
│  │ Public Subnet        │  │   │                        │
│  │ AZ 2                 │  │   │                        │
│  │ Frontend             │  │   │                        │
│  └──────────────────────┘  │   │                        │
└────────────────────────────┘   └────────────────────────┘
```

## Components

| Component | Region | Subnet | Purpose |
|---|---|---|---|
| Backend | N. Virginia | Private subnet, AZ 1 | Runs application/backend services |
| Frontend | N. Virginia | Public subnet, AZ 2 | Hosts the user-facing application |
| Database | Mumbai | Private subnet, AZ 1 | Stores application data |
| VPC Peering | N. Virginia ↔ Mumbai | Private network connection | Enables communication between the VPCs |

## How It Works

1. Users access the frontend hosted in the N. Virginia VPC.
2. The frontend communicates with the backend over the VPC's private network.
3. When the backend needs data, it sends a request to the database in the Mumbai VPC.
4. VPC Peering carries the traffic between the VPCs using private IP addresses.
5. The database returns the response to the backend through the peering connection.

## Configuration Requirements

1. Create the two VPCs in the required AWS Regions.
2. Use **non-overlapping CIDR blocks** for the VPCs.
3. Create the public and private subnets and launch the required resources.
4. Create a VPC peering connection between the two VPCs and accept the request.
5. Add routes to the route tables:
   - In the N. Virginia VPC, route the Mumbai VPC CIDR through the peering connection.
   - In the Mumbai VPC, route the N. Virginia VPC CIDR through the peering connection.
6. Update security groups and network ACLs to allow only the required traffic and ports.
7. Keep the database in a private subnet and allow database access only from the required application resources.

## Important Notes

- VPC Peering supports private communication; it does not automatically provide internet access.
- VPC peering is **not transitive**. A third VPC cannot use one peering connection to reach another VPC.
- The route tables on both sides must contain the appropriate routes.
- Security groups and network ACLs must permit the traffic.
- Cross-Region data transfer charges may apply.
- A public subnet does not automatically make an instance publicly reachable; the instance also needs an appropriate public IP and internet routing.

## Expected Result

The frontend and backend in the N. Virginia VPC can communicate with the database in the Mumbai VPC over private IP addresses through the VPC peering connection, without exposing the database directly to the public internet.