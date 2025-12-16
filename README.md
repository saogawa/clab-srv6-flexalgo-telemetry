# SRv6 FlexAlgo Telemetry Lab (Nokia SR OS) — *Derivative / Personal Edition*

[![Discord][discord-svg]][discord-url] [![DevPod][devpod-svg]][devpod-url] [![Codespaces][codespaces-svg]][codespaces-url]  
![w212][w212][Learn more](https://containerlab.dev/macos/#devpod) ![w90][w90][Learn more](https://containerlab.dev/manual/codespaces)

> **Respect & Credits**  
> This repository is a **derivative work** based on the excellent original lab created and maintained by **sros-labs**:  
> - Original: https://github.com/sros-labs/srv6-flexalgo-telemetry-lab  
>  
> I cloned the original repository and applied modifications for my own use (lab operation, config tweaks, and extensions).  
> All credit for the original concept, structure, and implementation belongs to the original authors.

---

## Objective

Creation of a traffic-engineered path based on SRv6 transport between 2 endpoints (R1 and R5) using **delay as a metric** to provide lowest latency connectivity between 2 clients over a L3VPN.

- Transport: Base [SRv6](https://www.nokia.com/networks/ip-networks/segment-routing/) (end-dt46) and FlexAlgo 128 (with STAMP dynamic delay measurement)
- Service: [EVPN](https://www.nokia.com/networks/ethernet-vpn/) IFL (Interface-less)

The purpose of this pre-configured lab is to demonstrate the use of an end-to-end SRv6 transport on Nokia SR OS routers spanning from Access/Aggregation ([7250 IXR](https://www.nokia.com/networks/ip-networks/7250-interconnect-router/) Gen2/2c) to Edge/Core ([7750 SR](https://www.nokia.com/networks/ip-networks/7750-service-router/), [FP4/FP5-based](https://www.nokia.com/networks/technologies/fp-network-processor-technology/)):

![wan_nodes drawio](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/943a1061-fb6c-4263-9717-9e602507dc20)

This relies on usage of a Flex-Algorithm (Algo 128) with delay used as metric to achieve the lowest latency path.  
The Flex-Algorithm for SRv6-based VPRNs feature allows the computation of constraint-based paths across an SRv6-enabled network, based on metrics other than the default IGP metrics. This allows carrying data traffic over an end-to-end path that is optimized using the best suited metric (IGP, delay, or TE).

---

## Telemetry / Observability Stack

An open source telemetry stack is used to collect and report objects of interest via Telemetry/gRPC:

- [gnmic](https://gnmic.openconfig.net/)
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)

![Screenshot 2024-03-04 at 12 53 12 PM](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/cafa2ed8-b933-4e48-9b67-b8001b72ae17)

gnmic is collecting streaming telemetry data (push-based) from all routers with a 5s sampling interval via subscription to paths of interest:

- `/state/router[router-name=Base]/interface[interface-name=*]/statistics`
- `/state/router[router-name=*]/interface[interface-name=*]/oper-state`
- `/state/service/vprn[service-name=50]/interface[interface-name=*]/oper-state`
- `/state/service/vprn[service-name=50]/interface[interface-name=*]/statistics`
- `/state/test-oam/link-measurement/router[router-instance=Base]/interface[interface-name=*]/last-reported-delay`
- `/state/system/cpu[sample-period=60]/summary/usage/`
- `/state/system/memory-pools/summary/`
- `/state/router[router-name=Base]/route-table/unicast/ipv6`
- `/state/router[router-name=Base]/bgp/statistics/peers`
- `/state/router[router-name=Base]/bgp/statistics/routes-per-family/vpn-ipv4/remote-active-routes`
- `/state/router[router-name=Base]/bgp/statistics/routes-per-family/vpn-ipv6/remote-active-routes`

> Note: All Nokia SR OS YANG models are publicly available at:  
> https://github.com/nokia/7x50_YangModels

Grafana dashboards are provided to check:
- Interfaces state per node
- Link latency in near-real time (STAMP/TWAMP-light)
- BGP peers/routes per node
- CPU/memory per node

---

## Network Topology

![Screenshot 2024-03-04 at 12 52 16 PM](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/b76b684c-4b13-41a7-bfb9-e61d17e214cd)

All routers are pre-configured — startup configuration can be found under `configs/Rx/Rx.cfg`.

Each router has 2 locators:
- Locator `c000:db8:aaa:10n::/64` in ISIS Algo 0
- Locator `c128:db8:aaa:10n::/64` in ISIS Algo 128 (used by VPRN 50) where *n* is Node-ID (1 is R1, 5 is R5)

R1 and R5 are ready to send/receive customer traffic through VPRN 50 (locator `c128:db8:aaa:10n::/64`).

The protocol used to dynamically measure link delay is TWAMP-Light (RFC 5357). Example on interface `to-R3` (R1):

```conf
interface "to-R3" {
-- Snip --
  if-attribute {
    delay {
      delay-selection dynamic
      dynamic {
        measurement-template "standard-direct"
        twamp-light {
          ipv4 {
            admin-state enable
          }
        }
      }
    }
  }
}

Using Grafana dashboard, it is possible to correlate the sum of TWAMP delay measurements and the IPv6 route table:


⸻

Requirements

To deploy this lab you need:
	1.	A server with Docker and containerlab￼
	2.	nokia_sros SR-SIM￼ 25.7.R1+ image (25.7.R1 introduced SR-SIM), and a valid license file.

⸻

Clone (This Repository)

Replace <your-account> with your GitHub account name.

git clone https://github.com/<your-account>/clab-srv6-flexalgo-telemetry.git
cd clab-srv6-flexalgo-telemetry

If you want the upstream original instead:
git clone https://github.com/sros-labs/srv6-flexalgo-telemetry-lab.git

⸻

Deploying the Lab

The lab is deployed with containerlab￼.
The topology is described in srv6-flexalgo.clab.yml￼.

clab deploy --reconfigure

To remove the lab:

clab destroy --cleanup


⸻

Accessing Network Elements and Telemetry Stack

Once deployed, SR OS nodes can be accessed via SSH using their management IPs (shown by clab deploy) or via hostname defined in the topology.

# SR OS routers
ssh admin@R1
ssh admin@R5-a

# Linux client containers
docker exec -it client1 bash

If accessing from a remote host, replace localhost with the CLAB server IP:
	•	Grafana: http://localhost:3000 (default credentials: admin/admin)
	•	Prometheus: http://localhost:9090/graph

⸻

Launching Traffic and Modifying Delay on Links

Client1 sends unidirectional traffic to Client2 through a L3VPN (EVPN IFL).
	•	Start: start_traffic.sh￼
	•	Stop:  stop_traffic.sh￼

A fine-grained control on link delay can be achieved via NetEm (host-side) or containerlab tools (since 0.44￼):

# Add 100ms latency on eth2 for node R1
containerlab tools netem set -n R1 -i eth2 --delay 100ms


⸻

License

This repository includes license-25.txt.
Please follow the original project’s license terms and any third-party component licenses.

⸻

Upstream References
	•	Upstream/original lab: https://github.com/sros-labs/srv6-flexalgo-telemetry-lab
	•	containerlab: https://containerlab.dev/
	•	SR OS YANG models: https://github.com/nokia/7x50_YangModels

⸻

