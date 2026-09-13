Healthcare Booking System

A Spring Boot REST API for booking doctor consultations and lab tests. It supports searching for doctors, labs, and facilities near a location, creating time slots, booking appointments against those slots, checking appointment status, and cancelling bookings.

Tech Stack
Java / Spring Boot — REST controllers, service layer, validation
Spring Data JPA — repositories backed by a relational database (H2, MySQL, or Postgres, depending on your application.properties)
Lombok — @Data, @Builder, @RequiredArgsConstructor to cut boilerplate on entities, DTOs, and services
Jakarta Bean Validation (@Valid, @NotBlank, @NotNull, @Pattern) — request validation at the controller layer
Project Structure
com.healthcare.healthcare_booking_system
├── Controller/     REST endpoints (thin — delegate to Services)
├── Services/       Business logic (booking rules, search, formatting)
├── Entitys/        JPA entities mapped to database tables
├── DTOs/           Request/response shapes exposed over the API
├── Repository/     Spring Data JPA repositories
└── Exceptions/     Custom exceptions + a global exception handler
Domain Model
Facility — a physical location (clinic) with a name, address, coordinates, and rating. Doctors and lab tests belong to a facility.
Doctor — belongs to a Facility; has a specialization, consultation fee, and rating. Names are auto-normalized to always start with "Dr. " (via formatDoctorName), regardless of how the name was entered.
LabTest — belongs to a Facility; has a price, report turnaround time, and whether home sample collection is offered.
Slot — a bookable time window for either a Doctor or a LabTest, identified by providerType + providerId. Slots have a status: AVAILABLE, BOOKED, or BLOCKED.
Appointment — created when a patient books a Slot. Tracks patient info, price charged, a generated confirmation code (AYU-XXXXXXXX), estimated wait time, and a status: PENDING_CONFIRMATION, CONFIRMED, CANCELLED, COMPLETED, or NO_SHOW.

ProviderType (DOCTOR, CLINIC, LAB) is the common thread that ties Slots, Appointments, and search results back to either a Doctor or a LabTest without needing separate booking flows for each.

How Booking Works
A Facility must exist first (doctors and lab tests both reference one).
A Doctor or LabTest is created under that facility.
A Slot is created for that doctor/lab test, marked AVAILABLE.
A patient books the slot (POST /api/appointments/booking):
The slot is validated (must exist, must be AVAILABLE, must belong to the given provider).
The price is looked up from the provider (doctor's consultation fee or lab test's price).
The slot is flipped to BOOKED.
An estimated wait time is calculated from how many other CONFIRMED appointments exist for that provider on the same day (15 minutes × count).
An Appointment is saved with status CONFIRMED and a generated confirmation code.
The patient can later check status (GET /api/appointments/{id}/status) or cancel (POST /api/appointments/{id}/cancel), which also frees the slot back to AVAILABLE.
API Endpoints
Method	Path	Purpose
POST	/api/facilities/add	Create a facility
DELETE	/api/facilities/{id}	Delete a facility
GET	/api/facilities/nearby	Search facilities by location/radius/subtype
POST	/api/doctors/addDoctor	Create a doctor under a facility
DELETE	/api/doctors/{id}	Delete a doctor
GET	/api/doctors/search	Search doctors by specialization/location/date
POST	/api/slots/add	Create a slot for a doctor or lab test
DELETE	/api/slots/{id}	Delete a slot
GET	/api/labs/search	Search lab tests by name/location/date
POST	/api/appointments/booking	Book an appointment against a slot
GET	/api/appointments/{id}/status	Get an appointment's current status
POST	/api/appointments/{id}/cancel	Cancel an appointment and free its slot
Error Handling

All errors return a consistent JSON shape via a global exception handler:

json
{
  "timestamp": "2026-09-13T14:30:55.123",
  "status": 404,
  "error": "NOT_FOUND",
  "message": "Slot not found with id: 1",
  "path": "/api/appointments/booking"
}
Exception	HTTP Status	When it's thrown
ResourceNotFoundException	404	Facility/Doctor/LabTest/Slot/Appointment id doesn't exist
SlotUnavailableException	409	Slot exists but is already BOOKED or BLOCKED
InvalidBookingException	400	Slot doesn't belong to the given provider, bad time range, etc.
MethodArgumentNotValidException	400	@Valid request body fails field validation
Running Locally
Configure your datasource in src/main/resources/application.properties.
mvn spring-boot:run (or run the main class from your IDE).
The API is available at http://localhost:8080 by default.
Testing with Postman

Import Healthcare-Booking-System.postman_collection.json into Postman. It's organized into folders that mirror the dependency order of the domain model, so running folders top to bottom — or the whole collection via the Collection Runner — sets up real data and chains IDs automatically, with no manual copy-pasting:

Facilities — creates a facility, saves its id into the facilityId collection variable. Also includes a validation-error case (missing fields) and a not-found case (deleting a nonexistent facility).
Doctors — creates a doctor under {{facilityId}}, saves the returned id into doctorId. Includes error cases: missing required fields (400), a nonexistent facilityId (404), searching with no matches (200 + empty array), and deleting a nonexistent doctor (404).
Slots — creates a slot for {{doctorId}}, saves the returned id into slotId. Includes error cases: nonexistent provider (404), end time before start time (400), and deleting a nonexistent slot (404).
Labs — search-only folder (no lab test creation endpoint yet); includes a malformed-date error case (400).
Appointments — books an appointment against {{doctorId}} + {{slotId}}, saves the returned appointmentId. Includes error cases: invalid phone number (400), nonexistent slot (404), double-booking an already-booked slot (400/409), checking/cancelling a nonexistent appointment (404 each), and cancelling an already-cancelled appointment (400).

Each request has built-in test scripts (visible in the Tests/Scripts tab) that assert the expected status code and, where relevant, the shape of the response — so a full collection run gives you a pass/fail readout for the whole API in one click, without writing any tests by hand.

Collection Variables
Variable	Set by	Used by
baseUrl	You (defaults to http://localhost:8080)	Every request
facilityId	Create Facility - Success	Add Doctor
doctorId	Add Doctor - Success	Create Slot, Book Appointment
slotId	Create Slot - Success	Book Appointment
appointmentId	Book Appointment - Success	Get Status, Cancel
Known Gaps / Possible Next Steps
No LabTest creation endpoint yet — lab tests currently need to be seeded directly in the database before the Labs search or a lab booking will return anything.
No authentication/authorization layer — all endpoints are currently open.
estimatedWaitMinutes is a simple count-based heuristic, not based on actual doctor schedules or slot duration.
