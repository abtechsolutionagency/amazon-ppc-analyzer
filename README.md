## Amazon PPC Analyzer – Technical Documentation

### 1. Project Overview

Amazon PPC Analyzer (also branded as **Angora Tools**) is a **browser-based single-page web application** for managing and analyzing Amazon PPC workflows.  
The app focuses on:

- **Centralizing PPC operations** such as audits, schedules, and product libraries.
- **Providing a modern dashboard UI** that runs entirely on the client side.
- **Storing state and configuration** securely in a managed Postgres database via Supabase.

The application is delivered as static files (HTML, CSS, JS) and does not require a traditional server-side framework.

---

### 2. Technology Stack

#### 2.1 Frontend

- **Language**: HTML5, modern JavaScript (ES6+), and CSS.
- **Architecture**: Single-page style interface built in `index.html` with custom UI components and layout.
- **JavaScript modules**: Feature-specific scripts live under the `js/` directory, including `js/auth.js` for authentication and user session handling.
- **Styling**: Custom CSS embedded in `index.html` with design tokens (CSS variables) for colors, typography, and dark/light themes.
- **Third-party browser libraries (CDN)**:
  - **`xlsx`**: Used for Excel import/export operations (e.g. reading PPC data from spreadsheets).
  - **`html2pdf`**: Used to convert in-browser reports and views into downloadable PDF files.

This design keeps the frontend lightweight, dependency-free (no React/Vue), and easy to host on any static hosting provider.

#### 2.2 Runtime / Hosting

- **Application type**: Static site.
- **Typical hosting**: Netlify or any static hosting (configured via `netlify.toml`).
- **Execution environment**: All business logic executes in the browser; communication with the backend happens via Supabase’s JavaScript SDK loaded from a CDN.

---

### 3. Backend & Database

#### 3.1 Backend-as-a-Service

- **Provider**: [Supabase](https://supabase.com/).
- **Services used**:
  - **Supabase Auth** for user registration, login, password recovery, and session management.
  - **Supabase PostgreSQL** for persistent storage of user data, configuration, and PPC-related structures.
- **Client SDK**: `@supabase/supabase-js` v2, initialized in `js/auth.js` with project URL and public (anon) key.

`js/auth.js` handles:

- Sign-in, sign-up, password reset, and recovery flows.
- Synchronization of basic profile information into a dedicated profile table.
- UI gating based on whether the user is authenticated and approved.

#### 3.2 Email Delivery (Mailgun)

- **Email provider**: [Mailgun](https://www.mailgun.com/).
- **Usage**: All transactional emails (such as password reset links, account confirmation, and admin notifications) are sent via Mailgun.
- **Integration**:
  - The frontend triggers email-related actions (for example, password recovery) via Supabase/Auth endpoints.
  - Behind the scenes, the configured Supabase project and/or supporting backend services use **Mailgun** to actually deliver the email to the user’s inbox.
- **Configuration**:
  - Mailgun API keys, domains, and sender addresses are stored securely in environment variables (not in this repository).
  - Different environments (development, staging, production) can point to different Mailgun domains or API keys.

#### 3.3 Database Schema (PostgreSQL)

The SQL file `supabase_angora_storage.sql` contains the core schema and security model. Key tables:

- **`angora_storage`**  
  Per-user key/value storage used by the frontend to persist arbitrary configuration and tool state.

- **`angora_user_profiles`**  
  Stores a single row per authenticated user:
  - Basic identity (`email`, `first_name`, `last_name`, `full_name`).
  - **Role** (`admin`, `super_admin`).
  - **Approval status** (`pending`, `active`, `rejected`) with optional rejection reason.
  - Audit fields (`reviewed_at`, `reviewed_by_user_id`, timestamps).

- **`angora_product_library`**  
  Holds a JSON document per user describing the product library (brands and products) used in audits and reports.

- **`angora_audit_schedule`**  
  Stores recurring audit schedules and completion tracking data for each user.

- **`angora_ops_audits`**  
  Represents guided OPS audit workflow state per user, audit card, and week (plus optional day/date/brand).

All of these tables are defined with:

- **Row Level Security (RLS)** policies to restrict access to the owning user or admins.
- **Triggers** that keep `updated_at` timestamps in sync automatically.

---

### 4. Authorization & Roles

Authorization is a combination of **Postgres security functions** and **frontend checks**:

- Helper functions such as:
  - `public.angora_is_super_admin()`
  - `public.angora_is_admin()`
  - `public.angora_is_active_user()`
  - `public.angora_user_role()` and `public.angora_user_status()`
- These functions:
  - Read current user information from the Supabase JWT.
  - Determine whether the user is authenticated and active.
  - Determine whether the user has `admin` or `super_admin` privileges.
- RLS policies on the tables above use these helper functions to allow or deny **select/insert/update/delete**.

On the frontend side:

- `js/auth.js`:
  - Manages sign-in/sign-up and password reset flows.
  - Syncs Supabase user metadata into `angora_user_profiles`.
  - Locks or unlocks the main UI based on:
    - Whether the user is logged in.
    - Whether the user has an `active` approval status.
    - Whether the user is a `super_admin` for admin-only actions.

This ensures that sensitive operations (such as managing admin accounts or audit data) are protected both in the database and in the UI.

---

### 5. Local Development & Setup

#### 5.1 Prerequisites

- A Supabase project (with URL and public anon key).
- Access to the Supabase SQL Editor for running the schema script.
- Any simple static file server (Node, Python, or built-in tooling from your IDE).

#### 5.2 Steps

1. **Clone or download the project code.**
2. **Configure Supabase**:
   - Open `js/auth.js`.
   - Set `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY` to match your Supabase project.
3. **Provision the database schema**:
   - Open the Supabase dashboard.
   - Go to **SQL Editor**.
   - Run the contents of `supabase_angora_storage.sql` once to create tables, functions, triggers, and RLS policies.
4. **Serve the app locally**:
   - From the project root, start a local static server, for example:
     - `npx serve .`
     - or any other static hosting tool.
   - Open the served URL (e.g. `http://localhost:3000`) in your browser.
5. **Create the initial super admin**:
   - Ensure the email configured in the SQL script (e.g. `proabdulbasit.me@gmail.com`) exists as a user in Supabase.
   - The final section of `supabase_angora_storage.sql` bootstraps this user with `super_admin` role and `active` status.

---

### 6. Summary

In summary, this project uses:

- A **static, JS-based SPA frontend** for all user interactions.
- **Supabase Auth + PostgreSQL** as the backend for authentication and data persistence.
- **Postgres RLS and helper functions** to implement a robust role/approval model for admins and super admins.
- **Browser libraries** such as `xlsx` and `html2pdf` for working with Excel data and generating PDF reports directly from the client.

This documentation provides a high-level yet technical view of what stack, database, and security model the Amazon PPC Analyzer uses.

