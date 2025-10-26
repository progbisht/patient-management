## Patient Management — README

Overview
--------
This repository contains a small patient-management system composed of two microservices:

- `patientService` — a Spring Boot REST service that manages patient records and orchestrates creation of billing accounts via gRPC.
- `billing-service` — a Spring Boot gRPC service that provides billing account creation.
- Postgres data volume and compose configuration for local development are included in `docker-compose.yaml` and `db-data/`.

Architecture
------------
- patientService (REST) -> BillingService (gRPC)
- patientService exposes a CRUD HTTP API for Patients and delegates billing account creation to billing-service using a generated gRPC client.
- billing-service exposes a gRPC server (port 9001 by default) and returns a BillingResponse when an account is created.

Services and responsibilities
-----------------------------

1) patientService
  - Responsibility: Store and manage patient records (CRUD). When a patient is created, it calls the billing service to create a billing account.
  - API: HTTP REST endpoints documented below.
  - Persistence: Uses a relational database (SQL scripts under `src/main/resources/data.sql` set up sample data).
  - Important files:
    - `src/main/java/.../controller/PatientController.java` — REST endpoints for patients.
    - `src/main/java/.../service/PatientService.java` — business logic, calls `BillingServiceGrpcClient`.
    - `src/main/java/.../grpc/BillingServiceGrpcClient.java` — gRPC client used to call billing-service.
  - Default ports: `server.port=4000` (see `src/main/resources/application.properties`).

2) billing-service
  - Responsibility: Provide billing account creation via gRPC. Currently returns a stubbed response (accountId "1234", status "ACTIVE").
  - Interface: gRPC service defined in `src/main/proto/billing_service.proto`.
  - Implementation file: `src/main/java/.../grpc/BillingGrpcService.java` (annotated with `@GrpcService`).
  - Default ports: `server.port=4001` (HTTP) and `grpc.server.port=9001` (gRPC) — see `src/main/resources/application.properties`.

API Reference
-------------

Patient REST API (patientService)

- Base path: `/patients`

1) GET /patients
   - Description: Retrieve a list of patients.
   - Response: 200 OK — JSON array of PatientResponseDTO.

2) POST /patients
   - Description: Create a new patient and create a billing account for them.
   - Request body (PatientRequestDTO):
     {
       "name": "string (required)",
       "email": "string (required, email)",
       "address": "string (required)",
       "dateOfBirth": "YYYY-MM-DD (string, required)",
       "registeredDate": "YYYY-MM-DD (string, required on create)"
     }
   - Response: 200 OK — PatientResponseDTO
     {
       "id": "uuid string",
       "name": "string",
       "email": "string",
       "address": "string",
       "dateOfBirth": "YYYY-MM-DD"
     }
   - Notes: On successful create, `PatientService` calls the billing-service gRPC `CreateBillingAccount` method.

3) PUT /patients/{id}
   - Description: Update an existing patient.
   - Path param: `id` (UUID)
   - Request body: same shape as `PatientRequestDTO` (registeredDate not required for update).
   - Response: 200 OK — updated PatientResponseDTO

4) DELETE /patients/{id}
   - Description: Delete a patient by id.
   - Path param: `id` (UUID)
   - Response: 204 No Content

gRPC API (billing-service)

- Proto file: `billing-service/src/main/proto/billing_service.proto` (also mirrored under `patientService/src/main/proto`).
- Service: `BillingService`
- RPCs:
  - CreateBillingAccount(BillingRequest) returns (BillingResponse)

BillingRequest
{
  string patientId = 1;
  string name = 2;
  string email = 3;
}

BillingResponse
{
  string accountId = 1;
  string status = 2; // e.g., ACTIVE
}

Notes on current behavior
-------------------------
- The billing-service `BillingGrpcService` currently returns a static accountId (`"1234"`) and status (`"ACTIVE"`). Replace this implementation to integrate with a real billing data store or external billing API.
- The patient creation flow in `PatientService` calls the gRPC client with the created patient's id, name and email:
  `billingServiceGrpcClient.createBillingAccount(newPatient.getId().toString(), newPatient.getName(), newPatient.getEmail());`

Environment & configuration
---------------------------
- patientService
  - `server.port` (default 4000) — `src/main/resources/application.properties`.
  - Billing gRPC client uses properties `billing.service.address` (default `localhost`) and `billing.service.grpc.port` (default `9001`).

- billing-service
  - `server.port` (HTTP default 4001) and `grpc.server.port` (default 9001) — configured in `src/main/resources/application.properties`.

Local development
-----------------
Option A: Run services with Maven wrapper

1. Start Postgres (if you use the included docker-compose):

```powershell
docker-compose up -d postgres
```

2. Run services from repo root in separate terminals:

```powershell
cd .\patientService
.\mvnw spring-boot:run

cd ..\billing-service
.\mvnw spring-boot:run
```

Option B: Run everything with Docker Compose (if compose file includes services):

```powershell
docker-compose up --build
```

Observability & errors
----------------------
- Both services log to console (SLF4J). Add metrics (Prometheus) and structured logs as needed.
- Alerts and SLAs are project-specific; add owners and thresholds when known.

Contracts & versioning
----------------------
- The `billing_service.proto` file is the canonical contract for the billing-service gRPC API. Keep proto under version control and follow semantic versioning for breaking changes.

Owners & contacts
------------------
- Service owners can be added here. For now, the repository owner is the main contact.

Next steps & improvements
-------------------------
- Implement persistent billing account storage in `billing-service`.
- Add integration tests that exercise patient creation and confirm gRPC calls.
- Add example curl/httpie requests and a Postman/Insomnia collection.

---
