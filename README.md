# ⚡ UPI Offline Mesh — Resilient Store-and-Forward Payment Engine

<p align="center">
  <em>A production-pattern Spring Boot backend demonstrating offline UPI payments routed through a Bluetooth-style peer-to-peer mesh network with zero-connectivity origination, hybrid cryptography, and atomic exact-once settlement.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Spring%20Boot-3.3.5-brightgreen.svg" alt="Spring Boot">
  <img src="https://img.shields.io/badge/Java-17-orange.svg" alt="Java Version">
  <img src="https://img.shields.io/badge/Security-RSA--OAEP%20%2B%20AES--256--GCM-blue.svg" alt="Crypto">
  <img src="https://img.shields.io/badge/Database-H2%20In--Memory-yellow.svg" alt="H2 Database">
  <img src="https://img.shields.io/badge/Build-Maven%20Wrapper-red.svg" alt="Maven">
</p>

---

## 📑 Table of Contents

1. [System Overview & Real-World Motivation](#-system-overview--real-world-motivation)
2. [End-to-End Architectural Flow](#-end-to-end-architectural-flow)
3. [Deep-Dive: The Core Engineering Challenges Solved](#-deep-dive-the-core-engineering-challenges-solved)
   - [1. Untrusted Intermediaries (Hybrid Cryptography)](#1-untrusted-intermediaries-hybrid-cryptography)
   - [2. The Concurrent Duplicate Storm (Atomic Idempotency)](#2-the-concurrent-duplicate-storm-atomic-idempotency)
   - [3. Replay Protection & Freshness Gates](#3-replay-protection--freshness-gates)
   - [4. Double-Spend & Balance Isolation (Optimistic Locking)](#4-double-spend--balance-isolation-optimistic-locking)
4. [Wire Formats & Payload Schemas](#-wire-formats--payload-schemas)
5. [Interactive Demo Dashboard Walkthrough](#-interactive-demo-dashboard-walkthrough)
6. [Complete REST API Reference](#-complete-rest-api-reference)
7. [Local Setup & How to Run on Your PC](#-local-setup--how-to-run-on-your-pc)
8. [Automated Testing & Concurrency Verification](#-automated-testing--concurrency-verification)
9. [Project Directory Structure](#-project-directory-structure)
10. [Production Roadmap vs. Demo Implementation](#-production-roadmap-vs-demo-implementation)
11. [Honest Engineering Limitations](#-honest-engineering-limitations)

---

## 🧭 System Overview & Real-World Motivation

Standard UPI architectures require real-time, bidirectional internet connectivity from both the sender and the receiver to the central NPCI/bank switch. When users are in **underground parking basements, subway tunnels, remote trails, or disaster-affected zones**, transactions instantly fail.

Existing alternatives have limitations:
- **USSD (`*99#`)**: Requires a live 2G GSM cellular voice/command channel (unavailable in zero-signal zones).
- **Standard UPI Lite**: Still requires one party or terminal to connect to verify ledger balance.

### The Solution: Mesh-Routed Deferred Settlement
This project implements **Delay-Tolerant Networking (DTN)** combined with **Asymmetric/Symmetric Hybrid Cryptography**:
1. **Origination (100% Offline):** The sender inputs their payment. The sender's phone encrypts the payload using the bank's public key.
2. **Hop-by-Hop Gossip (Offline Mesh):** The encrypted packet is broadcasted peer-to-peer across nearby devices over Bluetooth Low Energy (BLE) using a hop-decrementing Time-To-Live (TTL) counter.
3. **Bridge Ingestion (Online Transit):** As soon as *any* phone carrying the packet steps outside and obtains 4G/5G or Wi-Fi, it acts as a **bridge node** and automatically pushes the packet to the server.
4. **Backend Settlement (Exact-Once Guarantee):** The Spring Boot backend enforces an atomic idempotency gate, decrypts and validates the authentication tag, checks timestamp freshness, and commits the transaction to the database.

---

## 🏛️ End-to-End Architectural Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                        OFFLINE SENDER DEVICE                           │
│  1. Compose: PaymentInstruction { sender, receiver, amount, nonce }   │
│  2. Hybrid Encrypt with Bank Public Key:                               │
│     - Generate ephemeral AES-256 key                                   │
│     - AES-256-GCM encrypt payload (Authenticated)                     │
│     - RSA-OAEP-2048 encrypt AES key                                   │
│  3. Wrap: MeshPacket { packetId, ttl=5, createdAt, ciphertext }        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                       BLE Broadcast│ (Zero Internet)
                                    ▼
┌──────────────┐     Hop 1    ┌──────────────┐     Hop 2    ┌────────────────────────┐
│  Stranger 1  │ ───────────► │  Stranger 2  │ ───────────► │      Bridge Node       │
│  (Offline)   │              │  (Offline)   │              │ (Steps into 4G signal) │
└──────────────┘              └──────────────┘              └───────────┬────────────┘
                                                                        │
                                                HTTPS POST /bridge/ingest│
                                                                        ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                      SPRING BOOT BACKEND ORCHESTRATOR                          │
│                                                                                │
│  1. Ingestion Gate: Compute SHA-256(ciphertext)                                │
│                                                                                │
│  2. Atomic Idempotency Check:                                                  │
│     IdempotencyService.claim(hash) ──► putIfAbsent (CAS atomic operation)      │
│        ├── [Already Claimed] ────────► 200 OK (DUPLICATE_DROPPED) ──► Drop     │
│        └── [First Time Claimed] ─────► Continue Pipeline                       │
│                                                                                │
│  3. Hybrid Decryption & Tamper Check:                                          │
│     HybridCryptoService.decrypt(ciphertext)                                    │
│        - Unwrap AES key with RSA-2048 Private Key                              │
│        - Decrypt payload with AES-256-GCM                                      │
│        - Verify 128-bit Auth Tag ────► [Tampered/Altered] ──► Reject (INVALID) │
│                                                                                │
│  4. Freshness Gate:                                                            │
│     Verify |now - signedAt| <= 24 hours ──► [Expired] ──────► Reject (INVALID) │
│                                                                                │
│  5. Settlement Engine (@Transactional):                                         │
│     SettlementService.settle()                                                 │
│        - Debit Sender, Credit Receiver                                         │
│        - JPA @Version Optimistic Lock Verification                             │
│        - Insert Ledger Row with UNIQUE constraint on packet_hash               │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 Deep-Dive: The Core Engineering Challenges Solved

### 1. Untrusted Intermediaries (Hybrid Cryptography)
* **The Problem:** Strangers' devices carry the transaction packet. How do you prevent them from inspecting the amount or modifying the receiver VPA to their own account?
* **Why not plain RSA?** RSA-2048 using standard OAEP padding has a maximum data capacity of 214 bytes. Our payment JSON containing nonces, VPAs, and timestamps easily exceeds this.
* **The Hybrid Solution (`HybridCryptoService.java`):**
  1. Generate an ephemeral, single-use **AES-256** symmetric key via `SecureRandom`.
  2. Encrypt the raw payment JSON with **AES-GCM (Galois/Counter Mode)**.
  3. Encrypt the 32-byte AES key using the bank's **RSA-2048 Public Key** with `OAEPWithSHA-256AndMGF1Padding`.
  4. Pack together: `[ 256 bytes RSA Key ] + [ 12 bytes GCM IV ] + [ AES Ciphertext + 16-byte Auth Tag ]`.
* **Tamper Proofing:** AES-GCM is an AEAD (Authenticated Encryption with Associated Data) cipher. If an intermediary flips even a single bit in the ciphertext, decryption throws an `AEADBadTagException` and the backend immediately rejects the packet.

### 2. The Concurrent Duplicate Storm (Atomic Idempotency)
* **The Problem:** In a dense mesh, 3 different phones hold copies of the same packet. When all 3 walk outside at the exact same millisecond, they all POST the packet to the backend concurrently. Without synchronization, the sender would be debited 3 times.
* **Why Deduplicate on Ciphertext Hash (Not Packet ID or Plaintext)?**
  - **Packet ID can be forged:** A rogue intermediate could modify the outer `packetId` UUID and forward it.
  - **Plaintext wastes CPU:** Decrypting RSA-2048 before deduplicating opens the backend to CPU-exhaustion Denial-of-Service attacks.
  - **Ciphertext Hash is Secure & Fast:** Two deliveries of the same payment have byte-identical ciphertexts. Hashing takes $O(1)$ and happens before any expensive crypto operations.
* **The Atomic Gate (`IdempotencyService.java`):**
  ```java
  public boolean claim(String packetHash) {
      Instant now = Instant.now();
      Instant prev = seen.putIfAbsent(packetHash, now);
      return prev == null; // true = Winner / Settles; false = Duplicate / Dropped
  }
  ```
  `ConcurrentHashMap.putIfAbsent()` utilizes low-level CPU Compare-And-Swap (CAS) primitives. Even under extreme thread contention, exactly one caller succeeds.

### 3. Replay Protection & Freshness Gates
* **The Problem:** An attacker captures a valid encrypted packet out of the airwaves and retransmits it two weeks later.
* **The Defense:**
  - Inside the encrypted payload, the sender stamps `signedAt` (epoch milliseconds) and a unique UUID `nonce`.
  - The server verifies: `(currentTime - signedAt) <= 86400 seconds (24h)`.
  - Since `signedAt` is inside the AES-GCM envelope, an attacker cannot manipulate the timestamp without invalidating the authentication tag.

### 4. Double-Spend & Balance Isolation (Optimistic Locking)
* **The Problem:** Even with idempotency, simultaneous transfers affecting the same sender account must not cause lost updates.
* **The Defense:** The `Account` entity leverages JPA `@Version`:
  ```java
  @Version
  private Long version;
  ```
  If two settlement threads attempt to update the same sender concurrently, Hibernate checks the version column upon commit. The slower transaction aborts with an `OptimisticLockException` rather than corrupting the account balance.

---

## 📦 Wire Formats & Payload Schemas

### 1. Over-the-Air Wire Format: `MeshPacket.java`
This payload is transmitted across peer-to-peer devices. The outer fields are unencrypted so intermediaries can route and decrement TTL.

```json
{
  "packetId": "d3b07384-d113-494a-9c87-89b1e985b8c9",
  "ttl": 5,
  "createdAt": 1730000000000,
  "ciphertext": "pWz5Q...[Base64 encoded blob: 256B RSA key + 12B IV + Ciphertext + Tag]...=="
}
```

### 2. Sealed Payload: `PaymentInstruction.java`
Only readable by the Spring Boot backend after RSA-OAEP + AES-GCM decryption:

```json
{
  "senderVpa": "alice@demo",
  "receiverVpa": "bob@demo",
  "amount": 500.00,
  "pinHash": "8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918",
  "nonce": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "signedAt": 1730000000000
}
```

---

## 🖥️ Interactive Demo Dashboard Walkthrough

When you start the application and visit **`http://localhost:8080`**, an interactive, dark-mode visualizer lets you observe the mesh:

1. **Step 1 — Inject Packet:** Select Sender (`alice@demo`), Receiver (`bob@demo`), Amount (`100.00`), and click **"Inject into Mesh"**. Alice's virtual device now holds 1 encrypted packet.
2. **Step 2 — Gossip Rounds:** Click **"Run Gossip Round"**. Each round simulates local BLE discovery. Packets hop from `phone-alice` → `phone-stranger1` → `phone-stranger2` → `phone-bridge`, with the TTL decrementing each hop.
3. **Step 3 — Bridge Upload:** Click **"Flush Bridges to Backend"**. The bridge node (`phone-bridge`, the only node with internet) posts the packet to `/api/bridge/ingest`.
4. **Step 4 — Verify Ledger:** Observe the **Account Balances** update immediately in real time, and view the immutable transaction recorded in the **Transaction Ledger**.
5. **Step 5 — Duplicate Drop Verification:** Click **"Flush Bridges"** a second time. The dashboard will report: `DUPLICATE_DROPPED`, showing that the idempotency gate prevented a second debit.

---

## 📡 Complete REST API Reference

### 1. Server Public Key
Fetch the active public key to encrypt offline transactions.
* **Endpoint:** `GET /api/server-key`
* **Response:**
  ```json
  {
    "algorithm": "RSA-2048 / OAEP-SHA256",
    "publicKey": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...",
    "hybridScheme": "RSA-OAEP encrypts an AES-256-GCM session key"
  }
  ```

---

### 2. The Core Production Endpoint: Bridge Ingestion
Used by any online bridge device to upload mesh packets.
* **Endpoint:** `POST /api/bridge/ingest`
* **Headers:**
  - `Content-Type: application/json`
  - `X-Bridge-Node-Id: phone-bridge-01` *(Optional metadata)*
  - `X-Hop-Count: 3` *(Optional metadata)*
* **Request Body:**
  ```json
  {
    "packetId": "b1b72e90-c2cb-4fa1-841f-d7d6f5194f4a",
    "ttl": 2,
    "createdAt": 1730000000000,
    "ciphertext": "..."
  }
  ```
* **Possible Responses:**
  - **Settled (200 OK):**
    ```json
    {
      "outcome": "SETTLED",
      "packetHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "reason": null,
      "transactionId": 1
    }
    ```
  - **Duplicate Dropped (200 OK):**
    ```json
    {
      "outcome": "DUPLICATE_DROPPED",
      "packetHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "reason": null,
      "transactionId": null
    }
    ```
  - **Tampered or Stale (200 OK):**
    ```json
    {
      "outcome": "INVALID",
      "packetHash": "a4f1...",
      "reason": "decryption_failed",
      "transactionId": null
    }
    ```

---

### 3. Simulation & Mesh Management Endpoints

| Method | Endpoint | Description | Sample Request / Payload |
|---|---|---|---|
| `POST` | `/api/demo/send` | Simulates an offline sender generating and encrypting a packet. | `{"senderVpa":"alice@demo","receiverVpa":"bob@demo","amount":150.00,"pin":"1234","ttl":5}` |
| `GET` | `/api/mesh/state` | Returns the packet inventory of all 5 simulated devices. | *None* |
| `POST` | `/api/mesh/gossip` | Triggers one round of peer-to-peer gossip across nearby devices. | *None* |
| `POST` | `/api/mesh/flush` | Simulates all bridge nodes uploading held packets concurrently. | *None* |
| `POST` | `/api/mesh/reset` | Resets all device packet caches and clears idempotency memory. | *None* |
| `GET` | `/api/accounts` | Lists all accounts, balances, and optimistic lock versions. | *None* |
| `GET` | `/api/transactions`| Lists the 20 most recent transactions from the ledger. | *None* |

#### Quick `cURL` Test:
```bash
# Inject a payment of 250 into the mesh
curl -X POST http://localhost:8080/api/demo/send \
  -H "Content-Type: application/json" \
  -d "{\"senderVpa\":\"alice@demo\",\"receiverVpa\":\"bob@demo\",\"amount\":250.00,\"pin\":\"1234\",\"ttl\":5}"

# Run one gossip round
curl -X POST http://localhost:8080/api/mesh/gossip

# Trigger bridge upload
curl -X POST http://localhost:8080/api/mesh/flush
```

---

## 💻 Local Setup & How to Run on Your PC

### Prerequisites
* **Java Development Kit (JDK) 17 or higher** installed. Verify in your terminal:
  ```bash
  java -version
  ```
* **Git** installed.
* *Note:* You do **not** need Maven installed globally; the repository includes the Maven Wrapper (`mvnw` / `mvnw.cmd`).

### 1. Clone the Repository
```bash
git clone https://github.com/<your-username>/UPI_Without_Internet.git
cd UPI_Without_Internet
```

### 2. Run the Application

#### On Windows (Command Prompt / PowerShell):
```cmd
.\mvnw.cmd spring-boot:run
```

#### On macOS / Linux:
```bash
chmod +x ./mvnw
./mvnw spring-boot:run
```

### 3. Accessing the Services
* **Interactive Web Dashboard:** Open [http://localhost:8080](http://localhost:8080) in your browser.
* **H2 Database Web Console:** Open [http://localhost:8080/h2-console](http://localhost:8080/h2-console)
  - **JDBC URL:** `jdbc:h2:mem:upimesh`
  - **User Name:** `sa`
  - **Password:** *(Leave blank)*

---

## 🧪 Automated Testing & Concurrency Verification

The test suite validates cryptographic integrity and multithreaded race conditions.

Execute all tests via:
```bash
# Windows
.\mvnw.cmd test

# Mac / Linux
./mvnw test
```

### The Multithreaded Concurrency Test (`IdempotencyConcurrencyTest.java`)
This test simulates the hardest scenario in distributed payments: **3 bridge devices uploading the same transaction at the exact same instant**.

```java
@Test
void singlePacketDeliveredByThreeBridgesSettlesExactlyOnce() throws Exception {
    // 1. Create a single valid encrypted packet
    MeshPacket packet = demoService.createPacket("alice@demo", "bob@demo", new BigDecimal("100.00"), "1234", 5);

    // 2. Set up 3 concurrent threads blocked behind a CountDownLatch
    ExecutorService pool = Executors.newFixedThreadPool(3);
    CountDownLatch startGate = new CountDownLatch(1);
    AtomicInteger settled = new AtomicInteger();
    AtomicInteger duplicates = new AtomicInteger();

    for (int i = 0; i < 3; i++) {
        pool.submit(() -> {
            startGate.await(); // Synchronize release
            BridgeIngestionService.IngestResult r = bridge.ingest(packet, "bridge-node", 3);
            if ("SETTLED".equals(r.outcome())) settled.incrementAndGet();
            else if ("DUPLICATE_DROPPED".equals(r.outcome())) duplicates.incrementAndGet();
        });
    }

    startGate.countDown(); // Release all 3 threads simultaneously
    pool.shutdown();
    pool.awaitTermination(5, TimeUnit.SECONDS);

    // Assert: EXACTLY one settled, two dropped, balance debited only once
    assertEquals(1, settled.get());
    assertEquals(2, duplicates.get());
}
```

---

## 📂 Project Directory Structure

```
UPI_Without_Internet-main/
├── pom.xml                                 # Build dependencies (Spring Boot 3.3.5, Java 17)
├── mvnw / mvnw.cmd                         # Cross-platform Maven wrappers
├── README.md                               # Comprehensive system documentation
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties      # DB configuration, idempotency TTL, freshness windows
    │   │   └── templates/
    │   │       └── dashboard.html          # Interactive Thymeleaf frontend UI
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java     # Application entrypoint
    │       ├── config/
    │       │   └── AppConfig.java          # @EnableScheduling for cache TTL eviction
    │       ├── controller/
    │       │   ├── ApiController.java      # Public REST surface (/api/bridge/ingest, /api/mesh/*)
    │       │   └── DashboardController.java# Serves Thymeleaf UI at '/'
    │       ├── crypto/
    │       │   ├── ServerKeyHolder.java    # Manages RSA-2048 keypair lifecycle
    │       │   └── HybridCryptoService.java# RSA-OAEP + AES-256-GCM cipher pipeline
    │       ├── model/
    │       │   ├── Account.java            # Account entity with @Version optimistic lock
    │       │   ├── AccountRepository.java  # Spring Data JPA repository
    │       │   ├── Transaction.java        # Immutable ledger with unique index on packetHash
    │       │   ├── TransactionRepository.java
    │       │   ├── MeshPacket.java         # Over-the-air wire format (outer metadata + ciphertext)
    │       │   └── PaymentInstruction.java # Decrypted payload (sender, receiver, nonce, amount)
    │       └── service/
    │           ├── BridgeIngestionService.java # Orchestrates Hash -> Claim -> Decrypt -> Settle
    │           ├── DemoService.java        # Seeds mock accounts & simulates sender encryption
    │           ├── IdempotencyService.java # Atomic Compare-And-Set cache (in-memory Redis SETNX)
    │           ├── SettlementService.java  # @Transactional balance ledger debit/credit
    │           ├── VirtualDevice.java      # Simulated mesh participant phone
    │           └── MeshSimulatorService.java# Peer-to-peer BLE gossip routing engine
    └── test/
        └── java/com/demo/upimesh/
            └── IdempotencyConcurrencyTest.java # 3-thread concurrent upload & tamper test
```

---

## 🚀 Production Roadmap vs. Demo Implementation

| Architectural Component | Current Prototype / Demo | Enterprise Production Deployment |
|---|---|---|
| **Persistence Layer** | In-Memory H2 Database | **PostgreSQL / CockroachDB** with read replicas & CDC |
| **Idempotency Cache** | In-Memory `ConcurrentHashMap` | **Distributed Redis Cluster** (`SET key hash NX EX 86400`) |
| **Key Management** | Local JVM startup generation | **Hardware Security Module (HSM)** / AWS CloudHSM / Vault |
| **Mesh Communication** | Software Simulation (`MeshSimulatorService`) | Native Android **Nearby Connections API / BLE GATT** |
| **Bridge Ingestion Security** | Open HTTP endpoint | **Mutual TLS (mTLS)** with signed bridge client certificates |
| **Core Banking Integration** | Internal `Account` entity | **ISO 8601 / NPCI UPI Switch** asynchronous messaging (Kafka) |
| **Rate Limiting** | None | **Token Bucket / Redis RateLimiter** per bridge node and IP |

---

## ⚖️ Honest Engineering Limitations

When discussing or evaluating this architecture, consider the following real-world trade-offs:

1. **Deferred Settlement vs. Guaranteed Cash (The Double-Spend Challenge):**
   - Because the sender is completely offline, their current balance cannot be verified at the moment of packet creation.
   - If Alice has ₹500 and sends ₹500 to Bob in one room, and then sends another ₹500 to Carol in another room, whichever packet reaches the backend first will settle; the second will be `REJECTED` for insufficient funds.
   - *Production Solution:* Similar to **UPI Lite**, real-world implementations require pre-funding an **on-device Secure Enclave hardware wallet**, freezing those funds from the primary account before going offline.
2. **Mobile Operating System Bluetooth Restrictions:**
   - Modern mobile operating systems (iOS and Android 8+) aggressively throttle background Bluetooth Low Energy discovery to conserve battery life. Achieving seamless background mesh propagation without user intervention requires platform-level OS permissions or specialized frameworks.
3. **Delayed Receiver Confirmation:**
   - The merchant/receiver does not receive instant confirmation until an intermediary bridge syncs with the server. In our model, the receiver accepts a cryptographically signed deferred promise (IOU) until settlement.

---

## 📄 License
This project is open-source under the MIT License. Feel free to use, modify, and integrate it into your own research and portfolio projects.
