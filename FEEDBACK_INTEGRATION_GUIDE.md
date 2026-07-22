## FEEDBACK\_INTEGRATION\_GUIDE.md Updates

The following content should be added to `FEEDBACK_INTEGRATION_GUIDE.md` under a new section titled **API Rate Limiting Configuration**.

***

### 🛡️ API Rate Limiting Configuration Parameters

To ensure the stability, reliability, and security of the GSSoC platform, all external interactions with the core APIs are protected by an integrated rate-limiting middleware. This mechanism prevents abuse, mitigates brute force attacks, and ensures fair usage across all integrations (including those consuming feedback data). Understanding the configuration parameters is crucial for developing robust client applications.

#### 💡 Overview of Rate Limiting

Rate limiting controls the number of requests a client can make within a defined time window. Instead of applying hard stops, the API adheres to standard best practices by returning HTTP 429 "Too Many Requests" errors when limits are exceeded, and providing detailed headers for client introspection.

#### ⚙️ Understanding the Rate Limiting Window Parameters

The rate limiting policy is defined by two primary parameters: **`Limit`** (the count) and **`Window Duration`** (the time period). These policies can be applied globally or specifically to named endpoints.

| Parameter | Description | Data Type | Required/Optional | Typical Value Range |
| :--- | :--- | :--- | :--- | :--- |
| `Limit` | The maximum number of requests permitted during the specified window duration. | Integer | Required | 5 - 1000 |
| `Window Duration` | The time span (in minutes or seconds) over which the request count is maintained. | Float/Integer | Required | 1 (minute) to 60 (minutes) |
| `Policy Identifier` | A unique name used to reference a specific rate-limiting policy group (useful for named middleware policies). | String | Optional | e.g., `user_feedback_write`, `global_read_api` |

#### 🧱 Implementation Context: Rate Limiting Policy Definition

The underlying architecture configures rate limiting using dedicated service policies. When designing a new integration flow that requires specific limits, the following considerations apply:

**1. Global Limits (Default Behavior):**
A set of general constraints applied across all endpoints unless overridden. These are designed to protect the overall platform stability from excessive background traffic.

*   **Example Use Case:** Preventing malicious flooding of the entire API surface area.
*   **Headers Used:** `X-RateLimit-Limit` (Total permitted), `X-RateLimit-Remaining` (Count remaining in the current window).

**2. Named/Scoped Policies (Recommended Practice):**
For critical or resource-intensive endpoints (e.g., submitting detailed feedback, fetching complex reports), specific policies must be applied using a unique identifier (`Policy Identifier`). This allows one component to exceed its limit without impacting another endpoint.

*   **Example Policy:** `[Endpoint: POST /api/feedback]` protected by policy `USER_SUBMISSION`.
*   If the service is configured to allow 10 submissions per minute for this specific policy, exceeding that count will result in a clear rate-limit violation for *only* this resource.

#### 📝 Troubleshooting and Client Handling

Developers must implement robust client-side logic to gracefully handle `429 Too Many Requests` responses:

1.  **Read Headers:** Upon receiving an error, inspect the following standard HTTP headers provided in the response body/headers:
    *   `X-RateLimit-Reset`: The time (often a Unix timestamp or minutes from now) when the rate limit counter resets and new requests are permitted.
    *   `Retry-After`: A human-readable integer indicating how many seconds the client must wait before attempting the request again.
2.  **Implement Backoff:** Use an Exponential Backoff strategy combined with a randomized jitter factor when retrying API calls to prevent contributing to further rate limiting issues (the "thundering herd" problem).

---

### Example: Configuring 10 requests every 60 seconds

To document this specific configuration for quick reference, the conceptual policy setup looks like this:

```markdown
# Policy Definition Example: High-Volume Feed Access

* **Policy Name:** `feed_consumer`
* **Target Endpoint(s):** `/api/v1/feeds/{id}`
* **Limit:** 10
* **Window Duration:** 60 seconds (1 minute)
* **Behavior Note:** Requests exceeding this limit will trigger a 429 error, and the client should wait for the `X-RateLimit-Reset` window to open.
```