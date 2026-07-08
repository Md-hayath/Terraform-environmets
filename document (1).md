Below is a comprehensive reference of the **most commonly used Zoom REST APIs** grouped by category. I've also included the **HTTP method**, **endpoint**, **purpose**, **required scope (typical)**, and whether a **Basic (Free)** account can generally use it or if a **paid plan/feature** is usually required. (Some APIs also require admin privileges.)

> **Base URL**
>
> ```text
> https://api.zoom.us/v2
> ```

---

# 1. User APIs

| API Name             | Method | Endpoint                      | Purpose                        | Scope                        | Basic      |
| -------------------- | ------ | ----------------------------- | ------------------------------ | ---------------------------- | ---------- |
| Get Current User     | GET    | `/users/me`                   | Current logged-in user details | `user:read:user:admin`       | ✅          |
| List Users           | GET    | `/users`                      | List all users                 | `user:read:list_users:admin` | ✅          |
| Get User             | GET    | `/users/{userId}`             | Get user details               | `user:read:user:admin`       | ✅          |
| Create User          | POST   | `/users`                      | Create user                    | `user:write:user:admin`      | Paid/Admin |
| Update User          | PATCH  | `/users/{userId}`             | Update user                    | `user:write:user:admin`      | Paid/Admin |
| Delete User          | DELETE | `/users/{userId}`             | Delete user                    | `user:write:user:admin`      | Paid/Admin |
| Get User Settings    | GET    | `/users/{userId}/settings`    | User settings                  | `user:read:user:admin`       | ✅          |
| Update User Settings | PATCH  | `/users/{userId}/settings`    | Update settings                | `user:write:user:admin`      | Paid/Admin |
| User Permissions     | GET    | `/users/{userId}/permissions` | User permissions               | Admin                        | Paid       |

---

# 2. Meeting APIs

| API Name            | Method | Endpoint                                  |
| ------------------- | ------ | ----------------------------------------- |
| List Meetings       | GET    | `/users/{userId}/meetings`                |
| Create Meeting      | POST   | `/users/{userId}/meetings`                |
| Get Meeting         | GET    | `/meetings/{meetingId}`                   |
| Update Meeting      | PATCH  | `/meetings/{meetingId}`                   |
| Delete Meeting      | DELETE | `/meetings/{meetingId}`                   |
| Meeting Registrants | GET    | `/meetings/{meetingId}/registrants`       |
| Add Registrant      | POST   | `/meetings/{meetingId}/registrants`       |
| Meeting Polls       | GET    | `/meetings/{meetingId}/polls`             |
| Create Poll         | POST   | `/meetings/{meetingId}/polls`             |
| Batch Registrants   | POST   | `/meetings/{meetingId}/batch_registrants` |

Typical scope:

```text
meeting:read:admin
meeting:write:admin
```

---

# 3. Webinar APIs

| API                 | Endpoint                            |
| ------------------- | ----------------------------------- |
| List Webinars       | `/users/{userId}/webinars`          |
| Create Webinar      | `/users/{userId}/webinars`          |
| Get Webinar         | `/webinars/{webinarId}`             |
| Delete Webinar      | `/webinars/{webinarId}`             |
| Webinar Registrants | `/webinars/{webinarId}/registrants` |
| Webinar Polls       | `/webinars/{webinarId}/polls`       |

Requires Webinar license.

---

# 4. Recording APIs

| API                | Endpoint                                  |
| ------------------ | ----------------------------------------- |
| User Recordings    | `/users/{userId}/recordings`              |
| Meeting Recordings | `/meetings/{meetingId}/recordings`        |
| Delete Recording   | `/meetings/{meetingId}/recordings`        |
| Recover Recording  | `/meetings/{meetingId}/recordings/status` |

Scope:

```text
recording:read:admin
```

---

# 5. Reports APIs

Requires Pro/Business with Reports access.

| API                  | Endpoint                          |
| -------------------- | --------------------------------- |
| User Meeting Reports | `/report/users/{userId}/meetings` |
| Meeting Report       | `/report/meetings/{meetingId}`    |
| Daily Usage          | `/report/daily`                   |
| Billing Report       | `/report/billing`                 |
| Telephone Report     | `/report/telephone`               |
| User Activity        | `/report/activities`              |

Scopes:

```text
report:read:meeting:admin
report:read:billing:admin
report:read:daily_usage:admin
report:read:telephone:admin
report:read:user_activities:admin
```

---

# 6. Dashboard APIs

Business/Enterprise required.

| API                  | Endpoint             |
| -------------------- | -------------------- |
| Dashboard Users      | `/metrics/users`     |
| Dashboard Meetings   | `/metrics/meetings`  |
| Dashboard Webinars   | `/metrics/webinars`  |
| Dashboard Zoom Rooms | `/metrics/zoomrooms` |
| Dashboard CRC        | `/metrics/crc`       |

Scope:

```text
dashboard:read:admin
```

---

# 7. Billing APIs

| API              | Endpoint                      | Availability         |
| ---------------- | ----------------------------- | -------------------- |
| Billing Report   | `/report/billing`             | Paid accounts        |
| Account Plans*   | `/accounts/{accountId}/plans` | Limited availability |
| Account Settings | `/accounts/me/settings`       | Depends on account   |

> *The Plans API is available only for certain account types (for example, partner/master account scenarios), not all Zoom accounts.

---

# 8. Account APIs

| API              | Endpoint                                |
| ---------------- | --------------------------------------- |
| Account Settings | `/accounts/me/settings`                 |
| Account Info     | `/accounts/{accountId}`                 |
| Managed Domains  | `/accounts/{accountId}/managed_domains` |

---

# 9. Group APIs

| API          | Endpoint            |
| ------------ | ------------------- |
| List Groups  | `/groups`           |
| Get Group    | `/groups/{groupId}` |
| Create Group | `/groups`           |
| Update Group | `/groups/{groupId}` |
| Delete Group | `/groups/{groupId}` |

---

# 10. Role APIs

| API         | Endpoint          |
| ----------- | ----------------- |
| List Roles  | `/roles`          |
| Get Role    | `/roles/{roleId}` |
| Create Role | `/roles`          |
| Update Role | `/roles/{roleId}` |

---

# 11. Chat APIs

| API               | Endpoint                        |
| ----------------- | ------------------------------- |
| Send Chat Message | `/chat/users/{userId}/messages` |
| List Messages     | `/chat/users/{userId}/messages` |

---

# 12. Phone APIs

Requires Zoom Phone.

| API           | Endpoint            |
| ------------- | ------------------- |
| Phone Users   | `/phone/users`      |
| Call Logs     | `/phone/call_logs`  |
| Phone Reports | `/report/telephone` |

---

# 13. Team Chat APIs

| API      | Endpoint                        |
| -------- | ------------------------------- |
| Channels | `/chat/channels`                |
| Messages | `/chat/users/{userId}/messages` |

---

# 14. Whiteboard APIs

| API         | Endpoint       |
| ----------- | -------------- |
| Whiteboards | `/whiteboards` |

---

# 15. Contacts APIs

| API      | Endpoint    |
| -------- | ----------- |
| Contacts | `/contacts` |

---

# 16. Device APIs

| API     | Endpoint   |
| ------- | ---------- |
| Devices | `/devices` |

---

# 17. Zoom Rooms APIs

| API          | Endpoint                  |
| ------------ | ------------------------- |
| Zoom Rooms   | `/rooms`                  |
| Room Devices | `/rooms/{roomId}/devices` |

---

# 18. SIP/H.323 (CRC) APIs

| API       | Endpoint       |
| --------- | -------------- |
| CRC Ports | `/metrics/crc` |

---

# 19. Cloud Recording Analytics

| API                | Endpoint                           |
| ------------------ | ---------------------------------- |
| User Recordings    | `/users/{userId}/recordings`       |
| Meeting Recordings | `/meetings/{meetingId}/recordings` |

---

# 20. Commonly Tested APIs

These are the APIs most developers test first after implementing OAuth:

| Endpoint                              | Purpose                                      |
| ------------------------------------- | -------------------------------------------- |
| `GET /users/me`                       | Verify OAuth token and retrieve current user |
| `GET /users`                          | List all users                               |
| `GET /users/me/settings`              | View user settings                           |
| `GET /users/me/meetings`              | List meetings                                |
| `POST /users/me/meetings`             | Create a meeting                             |
| `GET /meetings/{meetingId}`           | Meeting details                              |
| `GET /users/me/recordings`            | Cloud recordings                             |
| `GET /report/users/{userId}/meetings` | Meeting reports (paid plans)                 |
| `GET /report/billing`                 | Billing report (paid plans)                  |
| `GET /metrics/users`                  | Dashboard users (Business/Enterprise)        |

---

## APIs that Zoom does **not** provide

These are common requests that are **not available** through the public Zoom REST API:

* ❌ Per-user license cost (e.g., "User A costs $15/month")
* ❌ License price lookup
* ❌ Subscription pricing
* ❌ Invoice PDF download via a standard REST endpoint
* ❌ Payment history
* ❌ Tax details
* ❌ Discount information
* ❌ Renewal date
* ❌ Direct "inactive license" API
* ❌ "Most expensive user" API

To build cost optimization features, you typically combine usage APIs (users, meetings, recordings, reports) with your organization's own subscription pricing to estimate savings.
