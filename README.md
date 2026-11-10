# Privacy-Preserving AI System Using Federated Learning

A full-stack Django platform that lets multiple organizations (hospitals, schools,
manufacturers, software companies, transport authorities, government departments)
collaboratively train an AI model **without sharing raw data** with a central server —
using a real, working implementation of Federated Averaging (FedAvg).

> **Honesty note:** Federated Learning reduces the need to centralize raw data, but it does
> not by itself guarantee complete privacy. Model updates can still leak information in some
> situations. Techniques such as differential privacy and secure aggregation (see
> `federated/services/fedavg.py` docstring) can add further protection and are noted as
> future enhancements in this codebase.

## Features

- Role-based access control: **Super Admin**, **Organization Admin**, **Staff**, **Analyst**
- Auto-generated unique 6-digit Organization & User IDs, checked for DB collisions
- Server-side organization data isolation — every query is scoped to `request.user.profile.organization`, never trusted from the frontend
- No public "forgot password" flow — only a Super Admin can reissue credentials, and every reissue is audit-logged
- Dataset & PDF/document upload with validation and (optional) text extraction
- **Real local model training** (scikit-learn: logistic regression, linear regression, small neural network) on an organization's own uploaded CSV
- **Real FedAvg aggregation** — a genuine weighted average of model weight arrays across organizations, not a simulated/fake computation
- Global model versioning with rollback
- AI-assisted prediction module, with private per-organization prediction history
- Internal + cross-organization chat (polling-based today, structured for a drop-in upgrade to Django Channels / WebSockets)
- Project status workflow (Not Started → Developing → Completed) with full timeline history
- Centralized, reusable audit logging across every sensitive action
- Chart.js-powered monitoring dashboard backed entirely by real database data
- CSV report exports
- REST API layer (Django REST Framework) with the same organization-scoping rules as the web UI

## Architecture

```
Organization 1 ──┐
                  ├── Local Dataset (private, never leaves the org)
                  ├── Local Training (training/services/local_training.py)
                  └── Model Update (weights + metrics only) ──┐
Organization 2 ───────────────────────────────────────────────┼──► Federated Server
                                                                │    (federated/services/orchestration.py)
Organization N ─────────────────────────────────────────────┘         │
                                                                        ▼
                                                          FedAvg (federated/services/fedavg.py)
                                                                        │
                                                                        ▼
                                                              New Global Model Version
                                                                        │
                                                                        ▼
                                                        Redistributed for the next round
```

Layers:
- **Organization layer** (`organizations`, `accounts`) — tenants, roles, isolation
- **Data ingestion layer** (`datasets`, `documents`) — upload + validation, org-private
- **Local training layer** (`training`) — real scikit-learn training on org data
- **Federated aggregation layer** (`federated`) — FedAvg + round orchestration + global model versioning
- **Prediction layer** (`predictions`) — inference from the aggregated global model
- **Supporting layers** — `audit` (centralized logging), `chat`, `monitoring`, `reports`

## Project Structure

```
privacy_ai_project/
├── manage.py
├── privacy_ai_project/        # settings, urls, wsgi, asgi
├── accounts/                  # auth, roles, permissions, user management
├── organizations/             # Organization, ProjectStatus, dashboards, id_generator
├── audit/                     # centralized AuditLog + log_action() service
├── datasets/                  # dataset upload + validation
├── documents/                 # document/PDF upload + text extraction
├── training/                  # LocalTrainingJob + real local training service
├── federated/                 # FederatedRound, GlobalModel, FedAvg, orchestration
├── predictions/                # Prediction model + inference service
├── chat/                       # ChatRoom/ChatMessage, polling-based chat
├── monitoring/                 # Chart.js dashboard views
├── reports/                    # report views + CSV export
├── templates/                  # all HTML templates (Django template inheritance)
├── static/{css,js}/            # styles + chat polling script
├── media/{datasets,documents,models}/
├── requirements.txt
└── .env.example
```

## Installation

```bash
git clone <your-repo-url>
cd privacy_ai_project
```

Create and activate a virtual environment:

```bash
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Configure environment variables:

```bash
cp .env.example .env
# Edit .env — the default (DB_ENGINE=sqlite) works with zero extra configuration.
# Set DB_ENGINE=mysql and fill in DB_NAME/DB_USER/DB_PASSWORD/DB_HOST/DB_PORT for production.
```

Run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

Create a superuser (or use the seed command below, which creates one automatically):

```bash
python manage.py createsuperuser
```

**Optional but recommended** — seed realistic demo data (organizations, users, datasets,
a real trained local model per organization, an aggregated federated round, chat messages):

```bash
python manage.py seed_demo_data
```

This prints the demo login credentials, e.g.:
- Super Admin: `superadmin` / `AdminPass123!`
- Organization Admin: `admin_1` / `DemoPass123!` (repeats for `admin_2`, `admin_3`, ...)

Run the project:

```bash
python manage.py runserver
```

Visit `http://127.0.0.1:8000/`.

## Running Tests

```bash
python manage.py test
```

Covers (Section 30): authentication, organization data isolation, dataset access control,
FedAvg aggregation correctness, audit log creation, and role-based permission enforcement.

## Upgrading Chat to Real-Time WebSockets

The chat system ships as polling (`chat/views.py: poll_messages`) so the project runs with
zero extra infrastructure. To upgrade:

1. `pip install channels channels-redis daphne`
2. Add `'daphne'` and `'channels'` to `INSTALLED_APPS`
3. Uncomment the `ProtocolTypeRouter` block in `privacy_ai_project/asgi.py`
4. Fill in `chat/routing.py` and add a `chat/consumers.py`

## Security Notes

- Passwords are always hashed via Django's built-in password hashers — never stored in plain text.
- CSRF protection is enabled project-wide via Django's default middleware.
- Every sensitive queryset is filtered server-side by `request.user.profile.organization` —
  organization identity is never trusted from the frontend (see `accounts/permissions.py`).
- File uploads are restricted by extension (`ALLOWED_DATASET_EXTENSIONS`,
  `ALLOWED_DOCUMENT_EXTENSIONS` in `settings.py`).
- Raw datasets are never transmitted to the federated aggregation service — only
  `weights_json` (model weights) and scalar metrics (accuracy/loss/sample count) are shared,
  as enforced by `federated/services/fedavg.py` and `federated/services/orchestration.py`.

## Known Limitations / Future Enhancements

- This demo runs federated learning as an **in-process, multi-organization simulation**
  (`FEDERATED_SIMULATION_MODE = True` in `settings.py`) rather than a distributed multi-machine
  deployment. The FedAvg math itself is real and correct; only the network transport is simulated.
- Aggregation requires participating organizations to train the *same model architecture* on
  *schema-compatible* data (same number/order of features). This mirrors a real constraint of FedAvg.
- OCR for scanned PDFs is not included; `documents/text_extraction.py` is structured so an OCR
  engine (e.g. pytesseract) can be added as a drop-in enhancement.
- Differential privacy / secure aggregation are documented as recommended additions, not yet implemented.
