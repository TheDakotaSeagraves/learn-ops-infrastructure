# learn-ops-api: AI-Assisted Exploration

## 1. Top-level folders in `learn-ops-api`

| Directory | Purpose |
| --- | --- |
| `.github/` | GitHub Actions CI/CD workflows and PR/issue templates |
| `.vscode/` | Editor settings shared with the team (launch configs, recommended extensions) |
| `config/` | Deployment config — Docker Compose YAML and Nginx configs for both the API and client reverse proxy |
| `LearningAPI/` | The core Django app — all business logic, models, views, serializers, and tests live here |
| `LearningPlatform/` | The Django project root — `settings.py`, root `urls.py`, and `wsgi.py`; wires everything together |
| `LogViewer/` | A separate mini Django app that provides a UI for viewing server logs |
| `logs/` | Runtime log files written by the server (not committed, just present in dev/prod) |
| `static/` | Source static assets (CSS, JS, images) before collection |
| `staticfiles/` | Output of `collectstatic` — what Django serves in production |
| `templates/` | Project-wide Django HTML templates (email layouts, base pages, etc.) |

## 2. Folders inside `LearningAPI`

| Directory | Responsibility |
| --- | --- |
| `models/` | ORM model definitions, split into sub-packages: `coursework/` (books, courses, assessments, learning objectives), `people/` (users, cohorts, students), and `skill/` (core skills, records). `tag.py` sits at the top level here. |
| `views/` | DRF ViewSets and APIViews — one file per resource. Handles HTTP request/response logic, permissions, and querysets. Also contains sub-packages for `github/` and `oauth2/` authentication flows. |
| `serializers/` | DRF serializers that translate model instances to/from JSON. Focused on the main user/cohort/capstone shapes that need custom representation. |
| `tests/` | Automated test suite — endpoint integration tests (`test_cohort_endpoints.py`) and unit-level tests per domain (cohorts, courses, team maker). |
| `fixtures/` | JSON data dumps for every model, used to seed the database for local development and tests. One file per model table (e.g., `LearningAPI_cohort.json`). |
| `migrations/` | Auto-generated Django migration files tracking the full schema history. |

## 3. What is the Pipfile?

`Pipfile` is the dependency manifest for **Pipenv**, Python's higher-level package manager. It replaces the older `requirements.txt` pattern by separating concerns into two sections:

- **`[packages]`** — runtime dependencies needed in every environment (dev, staging, production).
- **`[dev-packages]`** — dependencies only needed locally (linters, test runners, debuggers). These are never installed in production, keeping the production image smaller and the attack surface smaller.
- **`[requires]`** — pins the exact Python version (`3.11.11`) so every developer and the CI environment use the same interpreter.
- **`[scripts]`** — convenience shortcuts (`pipenv run migrate` instead of `pipenv run python3 manage.py migrate`).

Pipenv also generates a `Pipfile.lock` alongside this file, which records the exact resolved versions of every transitive dependency — ensuring reproducible installs across machines.

## 4. Key packages

| Package | What functionality does it provide and why? |
|---------|---------------------------------------------|
| `django` | The web framework the entire project is built on. Provides the ORM, URL router, admin interface, migrations system, authentication backend, and request/response lifecycle. Without it there is no project — every other package is either a Django plugin or a tool that supports it. |
| `djangorestframework` | Extends Django to build JSON APIs. Provides the `APIView` and `ViewSet` base classes used throughout `LearningAPI/views/`, the serializer system in `LearningAPI/serializers/`, token/session authentication for API clients, and automatic permission enforcement. This project is a pure API backend (its frontend is a separate React app), so DRF is what makes that possible. |
| `django-allauth` | Handles social/OAuth authentication — specifically GitHub login (`LearningAPI/views/github_login.py`, `oauth2/`). Manages the OAuth 2.0 handshake, user account creation/linking when a GitHub identity arrives for the first time, and session management after login. Pinned to `0.54.0` because allauth has breaking API changes between minor versions and `dj-rest-auth` depends on a specific allauth interface. |

## 5. What does `decorators.py` do?

A decorator in Python is a function that wraps another function to add behavior before or after it runs — without changing the wrapped function itself. You apply one with `@decorator_name` above a function definition.

`LearningAPI/decorators.py` defines two role-based access guards:

- **`@is_instructor()`** — checks whether the authenticated user belongs to the `Instructors` group. If not, it short-circuits the request and returns a `401 Unauthorized` before the view logic even runs.
- **`@is_staff()`** — same pattern, but checks for the `Staff` group.

Both follow the same three-layer structure: an outer factory function (`is_instructor`) returns a `decorator` function which returns a `__wrapper` function that intercepts the actual request. You call them with parentheses (`@is_instructor()`) rather than just `@is_instructor` because the outer function needs to run first to produce the decorator.

## 6. What is a serializer, and how does it fit the request/response cycle?

A serializer is a translator that sits between Django's ORM models and the outside world (JSON over HTTP).

**On the way out (response):** A view fetches a queryset or model instance from the database. The serializer converts that Python object into a dict, which DRF then renders as JSON and sends back to the client.

**On the way in (request):** A client POSTs or PATCHes JSON. The serializer parses that raw data, validates it against the declared fields and any custom `validate_*` methods, and — if valid — either creates or updates a model instance.

Without serializers, you'd have to manually call `.values()` on querysets, hand-validate every field, and write your own JSON encoding. DRF's `ModelSerializer` (which all serializers here extend) introspects the model and generates the fields automatically; you only declare what to include or override.

`NssUserSerializer` is a minimal example — it exposes five fields from the `NssUser` model (`url`, `slack_handle`, `github_handle`, `mentor`, `user`) and nothing else. Any field not listed is invisible to API consumers.

## 7. One model and what it represents

**`StudentAssessment`** (`LearningAPI/models/people/student_assessment.py`) represents the real-world act of an instructor assigning a specific assessment to a specific student.

It is a join table with context: it links a `student` (an `NssUser`) to an `assessment` (an `Assessment`), but it also carries:

- **`status`** — where the student currently stands on this assessment (e.g., assigned, in progress, passed).
- **`instructor`** — which instructor owns this assignment (nullable, in case an instructor is later removed).
- **`url`** — the student's submission link.
- **`date_created`** — when the assignment was made.

The `unique_together` constraint on `(student, assessment)` enforces that the same student can only be assigned the same assessment once — a direct encoding of a real-world business rule. The API needs to track this data because the entire grading and progress-tracking workflow depends on knowing which students have which assessments and what state each one is in.


## 8. Views vs. viewsets

| Type | Example | When to use it |
|------|---------|----------------|
| Plain view | `github_callback` — `LearningAPI/views/github_login.py:31` | One-off endpoint with unique logic (an OAuth callback, a webhook receiver, a redirect). There's no resource to model, and a ViewSet's structure would just add noise. |
| ViewSet | `CohortViewSet` — `LearningAPI/views/cohort_view.py:26` | Standard CRUD on a resource (list, get, create, update, delete). The router handles URL wiring and you get consistent naming for free. Also the right choice when you need per-action permission logic — `view.action` gives you the action name, as seen in `CohortPermission` at `cohort_view.py:18`. |

A **plain view** handles exactly one URL and one HTTP operation. You write the logic directly, wire it to a URL pattern yourself, and it does what you tell it to do — nothing more.

A **ViewSet** groups related operations — `list`, `retrieve`, `create`, `update`, `destroy` — into a single class where each method name is the action name. DRF's `Router` automatically generates all the standard URLs (`/cohorts/`, `/cohorts/<id>/`) from one class registration. The `@action` decorator lets you bolt on non-standard endpoints (like `assign` or `migrate` in `CohortViewSet`) without leaving the class.

`github_callback` is the right call as a plain function: it does one thing (redirect back to the frontend after GitHub's OAuth handshake), has no resource to CRUD, and would be awkward to force into a ViewSet.

## 9. What replaces templates and why?

In Django's Model-Template-View pattern, the template's job is to render data into a format the client can consume — traditionally HTML. In a REST API that format is JSON, and the **serializer takes that role entirely**.

The cycle becomes:

```
Model  →  Serializer  →  JSON response
```

Instead of a template interpolating Python values into HTML tags, the serializer converts a model instance into a Python dict and DRF renders that dict to JSON. The client (a React frontend, a mobile app, another service) receives clean JSON and decides how to display it — the API has no opinion on presentation at all.

This separation makes sense here because the frontend is a completely separate React application. Coupling the API to HTML templates would mean the backend owns the UI, which prevents the frontend from being independently deployed, versioned, or replaced. Returning JSON keeps the API reusable: the same endpoints can serve the React app, a future mobile client, or automated scripts without any changes on the API side.
