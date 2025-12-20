# Reliability: Oracle Architecure & Scalability

## 1. Problem: The "Cron-Loop" Bottleneck
The current Rodurite MVP utilizes a naive "infinite loop" architecture for its Oracle service.
1. Wake up.
2. Iterate through check **every active escrow** sequentially.
3. Sleep for 20 seconds.
4. Repeat.

### 1.1 Risks
*   **Linear Latency**: As the number of active trades grows ($N$), the time to process a loop increases ($O(N)$). If checking one trade takes 1s and we have 100 active trades, the loop takes 100s. Users waiting for their funds to release will experience increasing delays.
*   **Single Point of Failure**: The Oracle runs inside the main API process. If the API crashes (e.g., memory leak, bad request), the Oracle stops. Timers freeze, and refunds are missed.
*   **API Rate Limits**: Indiscriminately checking stale trades every 20 seconds wastes limited Roblox API quotas/threatens IP bans.

## 2. Design Goals
*   **G1: Instant Responsiveness**: User actions (e.g., clicking "I've Delivered") should trigger immediate checks.
*   **G2: Robustness**: The safety mechanism (refunds) must survive API crashes.
*   **G3: Efficiency**: Polling frequency should match the urgency of the trade.

## 3. Solution: Priority Worker Architecture
We propose shifting from a monotonic loop to a **Hybrid Event/Polling System**.

### 3.1 Architecture Overview
The system is split into three distinct components:

1.  **The API (Trigger)**:
    *   Exposes a `POST /check-status` endpoint.
    *   When a user manually claims delivery, this endpoint performs an **immediate, out-of-band check** for that specific trade.
    *   *Result*: 90% of successful trades clear instantly.

2.  **The "Hot" Sweeper (Frequent Polling)**:
    *   **Scope**: Escrows created or "Armed" in the last 15 minutes, or within 15 minutes of expiring.
    *   **Frequency**: Every 30 seconds.
    *   **Purpose**: Catch users who complete trades but forget to click the button, or ensure timely refunds for expiring trades.

3.  **The "Cold" Sweeper (Background Safety)**:
    *   **Scope**: All other active escrows (stale).
    *   **Frequency**: Every 10 minutes.
    *   **Purpose**: Ultimate safety net to ensure no trade is ever "lost" in specific edge cases.

### 3.2 Infrastructure Separation
*   **Service A (Web API)**: Handles user requests, dashboard, and manual triggers. Scaled horizontally.
*   **Service B (Oracle Worker)**: A dedicated, lightweight process responsible solely for the Sweepers and timers.
    *   *Reliability*: Being decoupled from the user-facing API, it is less likely to crash due to traffic spikes or bad input.

### 3.3 (Future) Task Queue Integration
For enterprise scale, the "Sweepers" will be replaced by a persistent Task Queue (e.g., Redis/Celery):
*   Instead of "checking if time is up", we schedule a job: `refund_job(listing_id, scheduled_for=now + 48h)`.
*   The worker sleeps until exactly the right second, reducing idle CPU and API usage to near zero.

## 4. Conclusion
Migrating to this Priority Worker model reduces average settlement time from ~40s (loop dependent) to <2s (event driven), while reducing Roblox API load by estimated 80% through adaptive polling of stale trades.
