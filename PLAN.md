2026-04-18 Saturday

Plan from gemma-4-26b-a4b on LM Studio

This high-level design focuses on a "Security through Obscurity" model for guests (using cryptographically secure tokens) and a traditional "Authentication" model for the administrators.

Since you are using **Rust (Actix-web)**, we will leverage its type safety and performance to ensure the backend is robust.

---

### 1. System Architecture
The system will follow a Client-Server architecture:
*   **Frontend:** A lightweight Single Page Application (SPA) built with HTML5, Tailwind CSS (for styling), and Vanilla JavaScript (to keep it fast and dependency-free).
*   **Backend:** An Actix-web REST API handling two distinct logic flows: the **Public Guest Flow** and the **Private Admin Flow**.
*   **Database:** A relational database (SQLite for simplicity/portability or PostgreSQL for robustness) to store guest lists, responses, and admin credentials.

---

### 2. Data Model
We need three primary tables in our database:

#### `guests` Table
This table is pre-populated with your 300 guests before the event.
*   `id`: Primary Key.
*   `name`: Full name of the guest.
*   `access_token`: A long, cryptographically secure UUID (e.g., `550e8400-e29b...`). This is what the QR code contains.
*   `is_invited`: Boolean.

#### `responses` Table
This stores the actual RSVP data once submitted.
*   `guest_id`: Foreign Key to `guests`.
*   `attendee_count`: Integer (how many people are coming from that household).
*   `note`: Text (the message for the bride/groom).
*   `submitted_at`: Timestamp.

#### `admins` Table
*   `username`: String.
*   `password_hash`: Argon2 or Bcrypt hashed password.

---

### 3. The API Design

#### **A. Public Guest Endpoints (No Login Required)**
These endpoints are accessible via the URL: `https://lkc.xfDC2.us/rsvp/{access_token}`
*   `GET /api/rsvp/{token}`: 
    *   **Logic:** Looks up the guest by token.
    *   **Returns:** Guest name (if found) and their current RSVP status. If token is invalid, returns 404.
*   `POST /api/rsvp/{token}`:
    *   **Logic:** Accepts a JSON body containing `attendee_count` and `note`. Updates the `responses` table for that specific guest ID.

#### **B. Private Admin Endpoints (Requires Authentication)**
These endpoints require a JWT (JSON Web Token) or Session Cookie.
*   `POST /api/auth/login`: Verifies admin credentials and issues a token.
*   `GET /api/admin/summary`: 
    *   **Returns:** Total guests invited, total number of responders, and the sum of all `attendee_count`.
*   `GET /api/admin/responses`: 
    *   **Returns:** A complete list of all guests who have responded, including their names, counts, and notes.

---

### 4. Frontend Strategy

#### **The Guest View (Mobile-First)**
When a guest scans the QR code, they land on a mobile-optimized page.
*   **Logic:** The JS reads the `access_token` from the URL. It calls the `GET` API to greet them by name ("Hi John!").
*   **Form:** A simple form with an input for "Number of Guests" and a textarea for "Note."
*   **Feedback:** Upon clicking "Submit," show a success animation and a thank you message.

#### **The Admin Dashboard (Desktop-Optimized)**
A protected route (`/admin`) that requires login.
*   **Dashboard Widgets:** 3 large cards showing: "Total Invited," "Total Responded," and "Total Expected Guests."
*   **Data Table:** A searchable, sortable list of all responses. Clicking a row expands it to show the full note left by the guest.

---

### 5. Security Plan

1.  **Guest Privacy:** Since there is no login for guests, we use **UUIDv4 tokens**. A guest cannot "guess" another guest's URL because the entropy in a UUIDv4 is too high.
2.  **Admin Security:** 
    *   Use **Argon2id** for password hashing.
    *   Implement **JWT (JSON Web Tokens)** for API authorization. The frontend stores this token in `HttpOnly` cookies to prevent XSS attacks.
3.  **Input Validation:** Use Rust's strong typing and the `validator` crate to ensure `attendee_count` is never negative and `note` does not contain malicious scripts (preventing XSS).
4.  **CORS Policy:** Configure Actix-web to only allow requests from `lkc.xf2.us`.

---

### 6. Deployment Workflow

1.  **Domain Setup:** Point `lkc.xf2.us` to your server IP via DNS.
2.  **Web Server:** Use **Nginx** as a reverse proxy. Nginx will handle SSL (via Let's Encrypt) and pass requests to the Actix-web service running on a local port.
3.  **Containerization:** Wrap the Rust binary, the SQLite database, and the Frontend assets in a **Docker container**. This makes deployment to any cloud provider (DigitalOcean, AWS, etc.) seamless.
4.  **QR Generation Script:** A small pre-deployment Python or Rust script that iterates through your guest list and generates 300 PNG files, each containing the unique URL `https://lkc.xf2.us/rsvp/{token}`.

### Summary of Tech Stack
*   **Language:** Rust
*   **Web Framework:** Actix-web
*   **Database Driver:** SQLx (Asynchronous)
*   **Database:** SQLite (Single file, easy backup)
*   **Frontend:** HTML5 / Tailwind CSS / JavaScript
*   **Auth:** JWT + Argon2id
*   **Infrastructure:** Docker + Nginx + Let's Encrypt (SSL)
