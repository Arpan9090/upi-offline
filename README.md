# ⚡ UPI Offline Mesh — Demo

<p align="center">
  <em>A resilient Spring Boot backend demonstrating offline UPI payments routed through a secure Bluetooth-style mesh network.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen.svg" alt="Spring Boot">
  <img src="https://img.shields.io/badge/Java-17%2B-orange.svg" alt="Java Version">
  <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License">
</p>

---

## 🧭 Overview

Imagine you're trapped in a basement with **zero cellular or Wi-Fi connectivity**, and you need to send ₹500 to a friend. 

This system solves that problem:
1. Your phone encrypts the payment locally.
2. It broadcasts the packet via a Bluetooth-style mesh to nearby devices.
3. The packet hops device-to-device until *some* node walks upstairs, connects to 4G, and silently uploads it to this backend.
4. The backend securely decrypts, deduplicates, and settles the transaction.

This repository serves as the **server-side orchestration engine** alongside a built-in software simulator, letting you demo the entire peer-to-peer flow locally on a single machine without real BLE hardware.

---

## ✨ What This Demo Proves

* **🔒 End-to-End Security:** Payments travel through untrusted intermediate nodes safely using hybrid RSA + AES-GCM encryption.
* **⚡ Bulletproof Idempotency:** Even if multiple bridge nodes upload the exact same packet simultaneously, it settles **exactly once** using atomic compare-and-set operations.
* **🛡️ Tamper & Replay Defense:** Altered or replayed packets are instantly rejected before ever touching the financial ledger.

---

## 🚀 Quick Start

### Prerequisites
* **JDK 17 or newer** installed and added to your `PATH` (verify via `java -version`).
* No external databases or Redis required — everything runs out-of-the-box using an in-memory database and JVM storage.

### Running the Application

* **Windows:**
  ```cmd
  mvnw.cmd spring-boot:run
