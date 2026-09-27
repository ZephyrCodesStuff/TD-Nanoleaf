# TD-Nanoleaf

A modular TouchDesigner component (`.tox`) that streams real-time generative visuals, video textures, and color animations directly to Nanoleaf panels via low-latency local UDP (`extControl`).

It automatically queries your Nanoleaf controller's physical panel layout via the local REST API, normalizes the geometric coordinates into UV texture space, and samples any arbitrary input TOP at a smooth, rate-limited cadence without requiring external Python libraries or cloud connections.

## Supported Hardware

| Controller / Panel Lineup | Protocol Version | Status | Notes |
| :--- | :--- | :--- | :--- |
| **Nanoleaf Aurora (Light Panels)** | `v1` | Supported and Tested | Uses 1-byte panel IDs and frame transitions |
| **Nanoleaf Canvas** | `v2` | Supported | Uses 2-byte Big-Endian panel IDs |
| **Nanoleaf Shapes (Hexagons, Triangles)** | `v2` | Supported | Requires `extControlVersion: "v2"` |
| **Nanoleaf Elements** | `v2` | Supported | Maps RGB samples to dual-white CCT |

---

## 🛠️ Component Overview: `nanoleaf.tox`

* **Arbitrary TOP Input:** Accepts any texture, movie, generative TOP, or render network via its input connector.
* **Layout-Aware Sampling:** Automatically fetches physical panel positions over the local REST API and samples pixel colors at true $(x, y)$ panel centers with edge inset protection.
* **Native Protocol Switching (`v1` / `v2`):** Supports legacy Aurora 1-byte UDP framing and modern 2-byte Big-Endian framing for newer panel shapes.
* **Rate-Limited Stream Pipeline:** Throttles output to a clean 10 Hz cadence with synchronized 100ms fade transitions (`trans_time = 1`), matching Nanoleaf's hardware interpolation engine for jitter-free fades without network flooding.
* **Zero Dependencies:** Uses standard Python sockets and `urllib`—no `requests` or `pip` packages required.

---

## ⚡ Quick Start

### 1. Find Your Nanoleaf's Local IP
The Nanoleaf OpenAPI server always operates on port **`16021`**.

1. Open Wireshark (or your router's client list / an mDNS discovery tool like Discovery/Bonjour Browser).
2. Look for hostnames matching `Light-Panels-<device_name>.local` or `Nanoleaf-<model>.local`.
3. Ping or query the domain to resolve its local IPv4 address (e.g., `192.168.1.62`).

### 2. Generate an Auth Token
Nanoleaf uses token-based local authentication:

1. Press and hold the **Power Button** on the physical Nanoleaf controller for **5–7 seconds** until the LED indicator starts flashing continuously (pairing mode).
2. Immediately send a POST request from your terminal:

```bash
curl "http://<NANOLEAF_IP>:16021/api/v1/new" --json '{}'
```

3. The controller returns your persistent auth token:

```json
{"auth_token": "<some_random_string_here>"}
```

*(Keep this token safe; you only need to generate it once.)*

## 🔌 TouchDesigner Setup

1. Drop `nanoleaf.tox` into your project network.
2. In the component's parameters:
   * **IP Address**: Enter your panel IP (e.g., `192.168.1.62`).
   * **Auth Token**: Paste your generated authorization token.
   * **Protocol Version**: Select `v1` for original Aurora panels or `v2` for Canvas/Shapes/Elements.
3. Click **Fetch Layout**: This queries the layout API and populates the internal coordinate table.
4. Wire any **TOP** (Movie File In, Noise TOP, Render, etc.) into the input of `nanoleaf.tox`.

## 🩺 Troubleshooting

### Panels are unresponsive / colors not updating
The most common cause of dropped frames is the controller failing to transition into UDP streaming mode (`extControl`) during TouchDesigner's initial cook cycle.

If the component is running but the lights remain frozen or fail to listen on UDP, manually trigger the streaming mode from your terminal:

**For Aurora (Light Panels - v1):**
```bash
curl -X PUT "http://<NANOLEAF_IP>:16021/api/v1/<AUTH_TOKEN>/effects" \
  --json '{"write":{"command":"display","animType":"extControl"}}' -v
```

**For Canvas / Shapes / Newer Models (v2):**
```bash
curl -X PUT "http://<NANOLEAF_IP>:16021/api/v1/<AUTH_TOKEN>/effects" \
  --json '{"write":{"command":"display","animType":"extControl","extControlVersion":"v2"}}' -v
```

You should receive a `200 OK` response with the streaming socket details (the port might be 60222):

```json
{
  "streamControlIpAddr": "0.0.0.0",
  "streamControlPort": 60221,
  "streamControlProtocol": "udp"
}
```

Once this command succeeds, TouchDesigner will immediately resume streaming frames to the reported UDP port.

### HTTP 400 Bad Request on effect changes
* Ensure you are issuing a **`PUT`** request, not a `POST` request.
* If testing on Canvas or Shapes, omitting `"extControlVersion": "v2"` will return a `400` error.
* If testing on legacy Aurora firmware, including `"extControlVersion": "v2"` will return a `400` error. Set the component version parameter to `v1`.

---

## License
AGPL-3.0. See [LICENSE](LICENSE) for more details.