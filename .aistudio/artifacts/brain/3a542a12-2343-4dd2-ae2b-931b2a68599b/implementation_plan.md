# GitHub Pages Readiness & Client-Side Persistence Plan

**Direct Answer: No — the repository is not ready for GitHub Pages as-is, and GitHub Pages cannot host your Express/PostgreSQL backend.** However, because the React frontend is already architected with an internal state engine, a few targeted updates will make the frontend 100% ready for GitHub Pages with full `localStorage` persistence so all hospital workflows survive page reloads without needing a live backend server.

---

## User Review & Critical Decisions

> [!IMPORTANT]
> **1. Is it ready for GitHub Pages right now? → No (requires 2 small adjustments)**
> - **Asset Base Path**: Vite currently builds with absolute root paths (`/assets/...`, `/favicon.svg`). On a GitHub Pages project URL (`https://soumyadip1993.github.io/GHMS/`), absolute paths break with `404 Not Found`. Configuring `base: './'` (relative paths) makes the build work on both GitHub Pages subpaths and root URLs.
> - **Build Artifact Output**: GitHub Pages needs either a GitHub Actions workflow or static build output (`frontend/dist`) configured for deployment.
>
> **2. What about the Backend? → GitHub Pages cannot run Node.js / Express / PostgreSQL**
> - **How GitHub Pages Works**: GitHub Pages is a **static file host only** (HTML, CSS, JS, images). It cannot execute `node backend/src/server.js` or host a PostgreSQL database.
> - **Why Your App Still Works**: Your React frontend (`HospitalContext`) runs completely in the browser using `mockDatabase` and does not depend on the Express backend to render or execute workflows.
> - **Chosen Persistence Strategy**: Per your selection, we will upgrade `HospitalContext` with **automatic `localStorage` persistence** (plus a "Reset Demo Data" option) so patient registrations, queue updates, prescriptions, lab reports, ward admissions, vitals, and staff changes persist across browser refreshes on GitHub Pages.
> - *(Optional Future Path)*: If you ever want the Express + Prisma backend live in production later, you would host the backend separately on Render, Railway, or Google Cloud Run + Cloud SQL/Supabase, while keeping the frontend on GitHub Pages.

---

## 1. Overview & Core Concept

- **What It Delivers**:
  1. **Universal Static Asset Resolution**: Updates the frontend Vite configuration and HTML asset references to use relative paths (`./`), ensuring zero broken assets whether hosted at `/GHMS/`, a custom domain, or locally.
  2. **Browser `localStorage` Persistence Engine**: Enhances the hospital state store so every clinical, administrative, pharmacy, lab, and ward action is automatically saved in the user's browser and restored on reload.
  3. **Automated GitHub Pages Workflow File**: Adds a ready-to-use `.github/workflows/deploy-pages.yml` file so whenever you are ready to enable GitHub Pages in your repo settings, it deploys `frontend/dist` automatically.
- **Target Audience / Persona**: Hospital staff roles (Doctor, Receptionist, Nurse, Lab Technician, Pharmacist, Administrator) and Patients evaluating or using the Jeevanta Hospital Management System on GitHub Pages.
- **Key Value**: Turns the static GitHub Pages deployment into a stateful, reload-safe web application without requiring paid backend hosting.

---

## 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Seamless Reload Persistence**: A user logs into a workspace (e.g., Receptionist or Doctor), registers a patient, advances a queue token, or prescribes medication, and refreshes the browser tab — their active session and all newly created records remain intact.
  2. **One-Click State Reset**: Users can reset the local browser database back to the clean factory demo dataset at any time from the Settings / Header controls if they want a fresh state.
- **Visual Identity & Theme**:
  - Preserves the existing Jeevanta Hospital clinical design system (`--color-primary: #163956` deep navy, slate surfaces, tabular numerals, and Plus Jakarta Sans typography) with zero visual regressions.
- **Interactive Feedback & Motion**:
  - Existing toast notifications (`animate-fade-in`) confirm state mutations and data reset actions.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Relative Base Path (`base: './'`) in Vite**
  - *Chosen Approach*: Configure `base: './'` in `frontend/vite.config.js` and `./favicon.svg` in `frontend/index.html`.
  - *Why*: Works simultaneously on `https://<username>.github.io/GHMS/`, custom domains, and the AI Studio preview server on port 3000 without breaking Express static serving.
  - *Alternatives Considered*: Hardcoding `base: '/GHMS/'`, which would break root-path preview environments unless conditionally toggled.
- **Decision 2: Client-Side `localStorage` Sync in `HospitalContext`**
  - *Chosen Approach*: Hydrate all entity lists (`patients`, `queue`, `clinicalTimeline`, `prescriptions`, `labReports`, `inventory`, `wards`, `vitalsLogs`, `staff`, `auditLogs`, and auth session) from `localStorage` on initial load with automatic synchronization on change.
  - *Why*: Provides database-like persistence across page reloads on static hosting (GitHub Pages) where Node.js/PostgreSQL cannot run.

---

## 4. Technical Architecture & Data Strategy *(Technical Reference)*

### Architecture & Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GitHub Pages (Static Web Host)                       │
│             Serves compiled frontend/dist (HTML, CSS, JS)               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   React Frontend (HospitalContext)                      │
│                                                                         │
│  ┌──────────────────────┐       Hydrate / Sync      ┌────────────────┐  │
│  │ Initial Seed Data    │ ────────────────────────► │  Browser       │  │
│  │ (mockDatabase.js)    │                           │  localStorage  │  │
│  └──────────────────────┘                           └───────▲────────┘  │
│                                                             │           │
│  ┌──────────────────────────────────────────────────────────┴────────┐  │
│  │                  Role-Based Workspaces & Views                    │  │
│  │  • Reception / Queue  • OPD Consultation  • Pharmacy & Inventory  │  │
│  │  • Lab Reports        • Inpatient Wards   • Admin & Staff RBAC    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Model & State Persistence

- **Storage Keys (`jeevanta_hms_*`)**:
  - Session state: `isAuthenticated`, `currentRole`, `currentUser`, `activeTab`
  - Clinical & operational collections: `patients`, `queue`, `clinicalTimeline`, `prescriptions`, `labReports`, `inventory`, `wards`, `vitalsLogs`, `staff`, `auditLogs`
- **Interactive Component & State Mapping**:
  - **Initial Load**: Reads each key from `localStorage` inside a safe `try/catch` parser; falls back to `initialData` from `mockDatabase.js` when no saved state exists.
  - **State Mutations**: All 10 operational handlers (`registerPatient`, `advanceQueueToken`, `saveConsultation`, `createPrescription`, `dispensePrescription`, `uploadLabReport`, `admitPatientToBed`, `dischargePatientFromBed`, `recordVitals`, `addStaffMember`, `toggleStaffStatus`, `registerAndLoginPatient`) automatically persist updated collections to `localStorage`.
  - **Reset Handler (`resetHospitalData`)**: Clears stored keys, restores `initialData` defaults, and displays a confirmation toast.
