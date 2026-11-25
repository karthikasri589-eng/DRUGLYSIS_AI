# DRUGLYSIS AI hospital workspace prototype

## Run locally

Use Python 3.11+ and install the dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
uvicorn backend.app:app --reload
```

On Windows, double-click `run.bat` to start the application. It creates `.venv` if needed, installs the requirements only when FastAPI or Uvicorn is missing from that environment, and starts the single FastAPI/Uvicorn server on port 8000. Stop any server already using port 8000 first; the launcher checks the port before indexing the catalogue. Open http://127.0.0.1:8000/.

For other computers on the same LAN, keep this server running and open `http://<server-lan-ip>:8000/` from each client (find the server address with `ipconfig`). Allow inbound TCP port 8000 on the server's private-network firewall if needed. The UI uses same-origin `/api` requests, so all clients use this server and its SQLite database; do not share the database file itself.

Alternatively, start one FastAPI server from the project root with `python -m uvicorn backend.app:app --reload --host 127.0.0.1 --port 8000`. The home URL shows the animated DRUGLYSIS AI welcome screen; after about four seconds it transitions to role selection. Routes `/entry`, `/home`, `/roles`, `/common-login`, `/doctor-login`, `/login`, `/individual-login` and the role-specific dashboard URLs use the existing consolidated `UI/index.html` for the entry, role selection, authentication, and workspaces. The UI uses the project's existing HTML/CSS/JavaScript setup; there is no React or Tailwind build configuration. All pages use the same FastAPI backend and SQLite database. Use the FastAPI server for application pages; VS Code Live Server cannot serve `/api` endpoints. The application stores data in `backend/druglysis.sqlite3` by default. Set `DRUGLYSIS_DB` to use another database location. Set `DRUGLYSIS_HTTPS=1` when serving behind HTTPS so the session cookie is Secure.

Use synthetic data only. This is a development prototype, not a deployment-ready clinical system. Before real patient information is used, configure managed secrets and HTTPS, backups, access monitoring, operational controls and applicable privacy/security review. Reception/Nurse registrations create a server-generated unique Patient ID and save the patient, contact and clinical history fields, optional blood group, nurse notes, assigned doctor's database ID, and registration timestamp in the shared SQLite database. Assigned doctors see registrations in a dashboard section that polls the backend every 10 seconds. Doctors can search and filter assigned patients, and submit separate visit/treatment records with observations, doctor-entered assessments, medicine details, recommended tests, notes, and status. Older visit records are retained; status and history are stored in the database. Test requests appear in the lab queue. Lab staff can search pending requests by Patient ID or name and submit measured result values, units, reference ranges, notes, or PDF/TXT/CSV/XML report files (maximum 5 MB). Uploaded files are stored in SQLite and served only to the doctor assigned to that patient. Nurse patient listings are scoped to records they registered, doctors only see assigned patients, and lab staff can submit results only against an authorized pending request. Instrument imports and automated clinical interpretation are not connected. Files are not parsed or clinically interpreted. Enter only results from validated methods; no demo patient results are seeded.

## Main API workflow

- `POST /api/login`, `POST /api/logout`, and `GET /api/me` manage the role-scoped server session.
- `POST /api/patients` registers a patient; `PUT /api/patients/{patient_id}` updates only nurse-owned registrations; `GET /api/patients?q=<name-or-id>&status=<pending|completed|follow_up>` lists only patients permitted for the signed-in nurse or assigned doctor.
- `POST /api/patients/{patient_id}/treatments` adds a new doctor visit. `GET /api/patients/{patient_id}/treatments` returns that assigned doctor's chronological treatment history.
- `GET /api/lab/requests` lists requested tests; `GET /api/lab/patients?q=<name-or-id>` searches patients with pending lab requests; `POST /api/lab/results` submits results against a pending request; `GET /api/lab/results` returns authorized laboratory result history.

The SQLite database keeps accounts, sessions, patients, consultations/current status, immutable treatment history, test requests, laboratory results/files, and audit entries. Schema changes are applied idempotently at application startup, including additive columns for existing SQLite databases. To test the full flow, create separate Doctor, Reception/Nurse, and Lab Technician accounts from their role login pages, register a synthetic patient assigned to that doctor with a lab request, submit a lab result from the lab dashboard, then review and submit two doctor visits. Confirm the results and both visits appear in the assigned doctor's patient profile, status filters return the expected patients, and logout invalidates the session.

## Prescriptions and pharmacy

Doctors can open **Prescription / Pharmacy** to find only their assigned patients, review medical history and lab reports, create a draft, manually enter medicines, optionally use catalog text matches, set follow-up details, review a print preview, and explicitly confirm finalization. Catalog results are informational matches only; they do not establish clinical suitability, and the application never calculates an individualized dose, frequency, duration, or quantity. The doctor must enter and confirm all prescription directions. Drafts are not sent to pharmacy. Finalized prescriptions are read-only and available in the authenticated Pharmacy queue, where staff can record partial or full dispensing without changing the doctor's original order.

The medicine catalogue source can be configured with `DRUGLYSIS_MEDICINE_CSV`. `run.bat` checks `backend\medicine.csv` and then `%USERPROFILE%\Downloads\druglysis-ai\backend 2\medicine.csv` when the variable is not set. A changed source file is indexed into the existing SQLite database at startup; unchanged sources are skipped. The supplied 195,605-row CSV expands the SQLite database to about 1.1 GB in the tested environment, so keep at least 1.2 GB free for the initial import. To index manually, run `python -m backend.import_medicine_catalog` from the project root. If no catalogue is present, manual prescription entry still works.

Pharmacy accounts cannot self-register. On the application server, run `python -m backend.create_pharmacy_user` from the project root and follow the prompts to provision each pharmacy account. Password input is hidden and stored using the application's password hash. The Pharmacy role can access only finalized prescriptions and dispensing actions; account provisioning requires local access to the project and database.

Prescription APIs include `GET/POST /api/prescriptions`, `GET/PUT /api/prescriptions/{prescription_id}`, `GET /api/patients/{patient_id}/prescriptions`, `GET /api/medicine-catalog/search`, `GET /api/pharmacy/prescriptions`, `PUT /api/pharmacy/prescriptions/{prescription_id}/status`, and `POST /api/pharmacy/dispense`. Doctors are scoped to their authenticated account and assigned patients. Pharmacy staff cannot read drafts or alter prescription items. Dispensing records include the authenticated staff account, date, quantity, status, and notes. Patient dashboard responses include the latest finalized next-visit date.

## Roles and data permissions

- Doctor accounts can be registered and can view only patients assigned to their account.
- Reception/Nurse accounts can register patients and assign registered doctors.
- Lab Technician accounts can search patient IDs and submit reports only for existing patients with a doctor assignment.
- Pharmacy accounts are provisioned locally by an operator; pharmacy staff can process finalized prescriptions but cannot create or edit doctor prescriptions.
- Database routes enforce these roles. Browser pages require a valid server session; frontend visibility is not the security boundary.
- Passwords use salted PBKDF2-HMAC-SHA256 hashes. Sessions are random opaque tokens stored hashed in SQLite and issued as HttpOnly, SameSite=Lax cookies.
- Important account, patient and lab result changes create audit entries.

Doctor, Reception/Nurse, and Lab Technician accounts can self-register through their role login pages. Pharmacy accounts require the local operator provisioning command above. For a real hospital deployment, all role enrollment should be restricted to a trusted administrator/invitation process.
