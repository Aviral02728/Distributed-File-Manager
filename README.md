# 📁 Distributed File Storage System

An educational distributed file storage system built for the **Computer Networks Lab (24B15CS313)** at Jaypee Institute of Information Technology, Noida.

The project demonstrates how a coordinator can split files into chunks, distribute those chunks across independent storage nodes, maintain replicas, and reconstruct the original file during download.

## ✨ Features

- 🌐 Client-server communication over HTTP/TCP
- 🏗️ Coordinator and storage-node architecture
- 🧩 Fixed-size file chunking using 1 MiB chunks
- 🔄 Round-robin placement across available storage nodes
- 📚 Replication factor of 2
- 🔐 SHA-256 integrity checks for chunks and complete files
- 💓 Storage-node registration and periodic heartbeats
- 🛟 Replica fallback when a primary node is unavailable
- 🗄️ SQLite metadata persistence
- 🖥️ React dashboard for upload, download, deletion, node status, and metrics
- 🐳 Docker Compose deployment and local process-based deployment

## 🏛️ Architecture

```text
                         HTTP
  React browser  -------------------------->  Coordinator
   :3000                         upload/download/list     :8000
                                               |
                         chunk transfer       |
                         HTTP                 |
                 +---------------------------+--------------------------+
                 |                           |                          |
                 v                           v                          v
          Storage Node A              Storage Node B              Storage Node C
             :8001                       :8002                       :8003
          chunks + health             chunks + health             chunks + health
```

| Component | Responsibility |
|---|---|
| React frontend | User interface and browser-side file operations |
| Coordinator | File metadata, chunking, placement, replication, reconstruction, and health tracking |
| Storage nodes | Persisting and serving raw chunk bytes |
| SQLite | Storing file, chunk, checksum, and chunk-location metadata |

The browser communicates with the coordinator. Storage nodes communicate with the coordinator and provide chunk-level operations.

## 🔁 Workflow

### ⬆️ Upload

1. The browser sends the file to `POST /upload`.
2. The coordinator splits the file into 1 MiB chunks.
3. Each chunk receives an identifier and SHA-256 checksum.
4. Chunks are assigned to available nodes using round-robin placement.
5. Each chunk is written to two different nodes when enough nodes are online.
6. File and chunk metadata is committed to SQLite.
7. The coordinator returns the `file_id`, checksum, chunk count, and replication factor.

### ⬇️ Download

1. The browser requests `GET /download/{file_id}`.
2. The coordinator loads the ordered chunk list from SQLite.
3. It tries the recorded nodes for each chunk, preferring the primary location.
4. Each downloaded chunk is checked against its stored checksum.
5. Valid chunks are assembled in index order.
6. The complete file checksum is verified before the response is returned.

If one node is unavailable, the coordinator can use another replica. If every replica for a required chunk is unavailable or corrupted, the download fails.

## 🛠️ Technology stack

| Area | Tools and technologies |
|---|---|
| Backend language | Python 3.9+; Docker images use Python 3.11 |
| API framework | FastAPI 0.115.5 |
| ASGI server | Uvicorn 0.32.1 |
| HTTP client | HTTPX 0.28.0 |
| Validation | Pydantic 2.10.1 |
| Multipart uploads | python-multipart |
| Database | SQLite |
| Integrity | Python `hashlib` with SHA-256 |
| Frontend | React 18.3.1 and React Scripts 5.0.1 |
| Styling | Vanilla CSS |
| Containers | Docker and Docker Compose |
| Network analysis | Wireshark and NPcap loopback capture |

## 🗂️ Project structure

```text
distributed-file-storage/
├── coordinator/
│   ├── main.py              # Coordinator API and metadata server
│   ├── requirements.txt
│   └── Dockerfile
├── storage_node/
│   ├── main.py              # Reusable storage-node API
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.js           # Dashboard
│   │   ├── api.js           # Coordinator API client
│   │   └── index.css        # UI styles
│   ├── package.json
│   └── Dockerfile
├── scripts/
│   ├── run_local.py         # Starts the backend services locally
│   └── test_system.py       # End-to-end test suite
├── docker-compose.yml       # Full multi-container deployment
├── start.bat                # Windows launcher
└── README.md
```

## 📋 Requirements

Choose one deployment method:

- **Docker:** Docker Desktop with Docker Compose support
- **Local:** Python 3.9+, Node.js 18+, and npm
- **Windows shortcut:** Python, Node.js/npm, and the included `start.bat`

## 🚀 Quick start with Docker

From the project directory:

```bash
docker compose up --build
```

Open the dashboard at <http://localhost:3000>.

| Service | URL |
|---|---|
| Frontend | <http://localhost:3000> |
| Coordinator API | <http://localhost:8000> |
| Interactive API docs | <http://localhost:8000/docs> |
| Coordinator health | <http://localhost:8000/health> |
| Storage node A | <http://localhost:8001> |
| Storage node B | <http://localhost:8002> |
| Storage node C | <http://localhost:8003> |

Stop the services with:

```bash
docker compose down
```

Docker uses named volumes for coordinator metadata and node data. To also remove persisted development data:

```bash
docker compose down -v
```

## 💻 Local development without Docker

Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install backend dependencies:

```powershell
python -m pip install -r coordinator\requirements.txt
python -m pip install -r storage_node\requirements.txt
```

Start the coordinator and all three storage nodes:

```powershell
python scripts\run_local.py
```

In another terminal, start the React frontend:

```powershell
cd frontend
npm install
npm start
```

The frontend is available at <http://localhost:3000>.

### 🪟 Windows launcher

After dependencies are installed, the included launcher can start all services in separate command windows:

```powershell
.\start.bat
```

The launcher is intended for Windows development and clears processes listening on ports 3000 and 8000-8003 before starting.

## 🔌 API overview

The coordinator exposes interactive documentation at `/docs`.

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Coordinator health |
| `GET` | `/stats` | Storage and system metrics |
| `GET` | `/nodes` | Registered node status |
| `POST` | `/nodes/register` | Register a storage node |
| `POST` | `/nodes/heartbeat` | Update node liveness and free space |
| `POST` | `/upload` | Upload and distribute a file |
| `GET` | `/files` | List stored files |
| `GET` | `/files/{file_id}` | Inspect file and chunk metadata |
| `GET` | `/download/{file_id}` | Reconstruct and download a file |
| `DELETE` | `/files/{file_id}` | Delete metadata and replicas |

Each storage node exposes:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Node status, free disk space, and chunk count |
| `POST` | `/chunks/{chunk_id}` | Store a chunk |
| `GET` | `/chunks/{chunk_id}` | Return chunk bytes |
| `DELETE` | `/chunks/{chunk_id}` | Delete a chunk |
| `GET` | `/chunks` | List chunks on the node |

## ⚙️ Configuration

Coordinator settings are defined in `coordinator/main.py`:

| Setting | Default | Meaning |
|---|---:|---|
| `CHUNK_SIZE_BYTES` | `1 MiB` | Maximum size of each chunk |
| `REPLICATION_FACTOR` | `2` | Number of copies requested per chunk |
| `HEARTBEAT_INTERVAL` | `10 seconds` | Coordinator health-check interval |
| `NODE_TIMEOUT` | `30 seconds` | Time before a node is considered offline |

Storage nodes use these environment variables:

| Variable | Example | Meaning |
|---|---|---|
| `NODE_ID` | `node_a` | Unique node name |
| `NODE_PORT` | `8001` | Listening port |
| `NODE_PUBLIC_URL` | `http://localhost:8001` | URL advertised to the coordinator |
| `COORDINATOR_URL` | `http://localhost:8000` | Coordinator address |
| `STORAGE_DIR` | `data/node_a` | Directory containing chunk files |
| `HEARTBEAT_INTERVAL` | `10` | Heartbeat period in seconds |

## 🧪 Testing

Start the full system first, then run:

```powershell
python scripts\test_system.py
```

The end-to-end suite covers:

- Coordinator and storage-node health
- Small single-chunk upload
- Large multi-chunk upload
- File listing
- Downloaded-byte and SHA-256 integrity
- Distribution across multiple nodes
- Replica-recovery guidance
- Checksum verification behavior
- File deletion and `404` confirmation

The automated failure test does not kill a running process. For a live Docker demonstration:

```bash
docker stop dfs-node-a
python scripts/test_system.py
docker start dfs-node-a
```
---

## 🔬 Wireshark Analysis

To capture traffic between coordinator and storage nodes:

```bash
# Capture on loopback (all DFS ports)
# Windows: capture on "Loopback Adapter" or use NPcap
wireshark -i lo -f "tcp port 8000 or tcp port 8001 or tcp port 8002 or tcp port 8003"

# Useful display filters in Wireshark:
# http                          → all HTTP frames
# tcp.port == 8001              → Node A traffic
# http.request.method == "POST" → chunk uploads
# http.request.method == "GET"  → chunk downloads
# http && ip.addr == 127.0.0.1  → all local DFS traffic
```

**What to observe:**
- TCP 3-way handshake before each chunk transfer
- HTTP POST to `/chunks/{chunk_id}` during upload
- HTTP GET to `/chunks/{chunk_id}` during download
- Heartbeat POSTs every 10 seconds to `/nodes/heartbeat`
- TCP FIN/RST when a node goes offline

---

## 📊 Performance Metrics

| Metric | Description |
|--------|-------------|
| Upload throughput | MB/s — displayed in UI after each upload |
| Download throughput | MB/s — measured during reconstruction |
| Chunk count | Files ≥ 1 MiB split into multiple chunks |
| Replication overhead | 2× storage per chunk (replication factor = 2) |
| Heartbeat interval | 10 seconds |
| Node failure detection | ≤ 30 seconds (configurable via NODE_TIMEOUT) |
| Checksum pass rate | 100% under normal conditions |

---

## 🎓 CN Concepts Demonstrated

| Concept | Implementation |
|---------|---------------|
| **TCP/IP** | HTTP over TCP — all chunk transfers use TCP |
| **Socket Programming** | FastAPI/Uvicorn uses Python sockets internally |
| **Client-Server Model** | Coordinator = server, Nodes & Browser = clients |
| **Data Transfer** | Large files transferred as ordered chunks |
| **Replication** | Each chunk stored on 2 different nodes |
| **Error Detection** | SHA-256 checksums catch corruption/incomplete transfers |
| **Heartbeat** | Nodes send `/nodes/heartbeat` every 10s |
| **Failure Recovery** | Download falls back to replica if primary offline |
| **IP Addressing** | Each node has unique host:port (IP equivalent) |
| **Distributed Systems** | Coordinator tracks global state across 3 nodes |

---
