# Medixar HMS -- BDD Scenarios

## 1. Overview

This document contains Gherkin-style Behavior-Driven Development (BDD) scenarios for the critical flows in Medixar HMS Phase 1, 2, and 3. These scenarios serve as executable specifications and can be adapted for Playwright E2E tests or Jest integration tests.

---

## 2. Authentication

### Feature: User Login

```gherkin
Feature: User Login
  As a registered user
  I want to log in with my credentials
  So that I can access the hospital management system

  Background:
    Given a tenant "hospital-alpha" exists in the platform database
    And a user exists with email "doctor@hospital.com" and password "SecurePass123!"
    And the user has roles ["doctor"]
    And the user account is active

  Scenario: Successful login with valid credentials
    When I send a POST request to "/api/v1/auth/login" with:
      | email    | doctor@hospital.com |
      | password | SecurePass123!      |
    Then the response status should be 200
    And the response should contain "access_token"
    And the response should contain "refresh_token"
    And the response "user.email" should be "doctor@hospital.com"
    And the response "user.roles" should contain "doctor"
    And the response "user.permissions" should contain "patient:read"
    And the response "user.permissions" should contain "encounter:create"

  Scenario: Login with invalid password
    When I send a POST request to "/api/v1/auth/login" with:
      | email    | doctor@hospital.com |
      | password | WrongPassword123!   |
    Then the response status should be 401
    And the response "error.code" should be "UNAUTHORIZED"
    And the response "error.message" should be "Invalid credentials"

  Scenario: Login with non-existent email
    When I send a POST request to "/api/v1/auth/login" with:
      | email    | nobody@hospital.com |
      | password | SecurePass123!      |
    Then the response status should be 401
    And the response "error.message" should be "Invalid credentials"

  Scenario: Login with inactive account
    Given the user account is deactivated
    When I send a POST request to "/api/v1/auth/login" with:
      | email    | doctor@hospital.com |
      | password | SecurePass123!      |
    Then the response status should be 401
    And the response "error.message" should be "Invalid credentials"

  Scenario: Login with invalid tenant slug
    When I send a POST request to "/api/v1/auth/login" with:
      | email       | doctor@hospital.com |
      | password    | SecurePass123!      |
      | tenant_slug | non-existent-org    |
    Then the response status should be 404
    And the response "error.message" should be "Organization not found"

  Scenario: Login with missing email field
    When I send a POST request to "/api/v1/auth/login" with:
      | password | SecurePass123! |
    Then the response status should be 400
    And the response "error.code" should be "VALIDATION_ERROR"

  Scenario: Login with password shorter than 8 characters
    When I send a POST request to "/api/v1/auth/login" with:
      | email    | doctor@hospital.com |
      | password | short               |
    Then the response status should be 400
    And the response "error.code" should be "VALIDATION_ERROR"
```

### Feature: User Registration

```gherkin
Feature: User Registration
  As a new user
  I want to register for an account in my organization
  So that I can access the system

  Background:
    Given a tenant "hospital-alpha" exists in the platform database

  Scenario: Successful registration
    When I send a POST request to "/api/v1/auth/register" with:
      | email       | newuser@hospital.com |
      | password    | SecurePass123!       |
      | first_name  | Jane                 |
      | last_name   | Smith                |
      | tenant_slug | hospital-alpha       |
    Then the response status should be 201
    And the response should contain "id"
    And the response "email" should be "newuser@hospital.com"
    And the response "first_name" should be "Jane"

  Scenario: Registration with duplicate email
    Given a user exists with email "existing@hospital.com" in tenant "hospital-alpha"
    When I send a POST request to "/api/v1/auth/register" with:
      | email       | existing@hospital.com |
      | password    | SecurePass123!        |
      | first_name  | Jane                  |
      | last_name   | Smith                 |
      | tenant_slug | hospital-alpha        |
    Then the response status should be 409
    And the response "error.code" should be "CONFLICT"

  Scenario: Registration with non-existent tenant
    When I send a POST request to "/api/v1/auth/register" with:
      | email       | newuser@hospital.com |
      | password    | SecurePass123!       |
      | first_name  | Jane                 |
      | last_name   | Smith                |
      | tenant_slug | non-existent-org     |
    Then the response status should be 404

  Scenario: New user receives default receptionist role
    When I register and then log in as "newuser@hospital.com"
    Then the response "user.roles" should contain "receptionist"
    And the response "user.permissions" should contain "patient:create"
    And the response "user.permissions" should contain "appointment:create"
```

### Feature: Token Refresh

```gherkin
Feature: Token Refresh
  As an authenticated user
  I want my session to be automatically refreshed
  So that I do not have to log in again within 7 days

  Scenario: Successful token refresh
    Given I am logged in and have a valid refresh token
    When I send a POST request to "/api/v1/auth/refresh" with:
      | refresh_token | <valid_refresh_token> |
    Then the response status should be 200
    And the response should contain "access_token"
    And the response should contain "refresh_token"
    And the new refresh token should be different from the old one

  Scenario: Refresh with revoked token
    Given I am logged in and have a refresh token
    And the refresh token has been revoked (user logged out)
    When I send a POST request to "/api/v1/auth/refresh" with:
      | refresh_token | <revoked_token> |
    Then the response status should be 401
    And the response "error.message" should be "Invalid or expired refresh token"

  Scenario: Refresh with expired token
    Given I have a refresh token that expired 8 days ago
    When I send a POST request to "/api/v1/auth/refresh" with:
      | refresh_token | <expired_token> |
    Then the response status should be 401

  Scenario: Old refresh token is invalidated after rotation
    Given I am logged in and have a valid refresh token "TOKEN_A"
    When I refresh using "TOKEN_A" and receive "TOKEN_B"
    And I try to refresh again using "TOKEN_A"
    Then the response status should be 401
```

### Feature: Logout

```gherkin
Feature: Logout
  As an authenticated user
  I want to log out
  So that my session is securely terminated

  Scenario: Successful logout
    Given I am logged in with a valid JWT token
    When I send a POST request to "/api/v1/auth/logout"
    Then the response status should be 200
    And the response "message" should be "Logged out successfully"
    And all my refresh tokens should be revoked

  Scenario: Logout without authentication
    When I send a POST request to "/api/v1/auth/logout" without a Bearer token
    Then the response status should be 401
```

---

## 3. Patient CRUD

### Feature: Patient Registration

```gherkin
Feature: Patient Registration
  As a receptionist
  I want to register new patients
  So that they can receive medical care

  Background:
    Given I am logged in as a user with "patient:create" permission

  Scenario: Register a patient with required fields only
    When I send a POST request to "/api/v1/patients" with:
      | first_name    | John       |
      | last_name     | Doe        |
      | date_of_birth | 1990-01-15 |
      | gender        | Male       |
      | phone_primary | +1234567890 |
    Then the response status should be 201
    And the response should contain "id"
    And the response should contain "mrn"
    And the response "first_name" should be "John"
    And the response "status" should be "Active"
    And the response "created_by" should be the authenticated user's ID

  Scenario: Register a patient with all fields
    When I send a POST request to "/api/v1/patients" with full patient details
    Then the response status should be 201
    And the response should include all provided fields

  Scenario: Register a patient with invalid gender enum
    When I send a POST request to "/api/v1/patients" with:
      | first_name    | John         |
      | last_name     | Doe          |
      | date_of_birth | 1990-01-15   |
      | gender        | InvalidValue |
      | phone_primary | +1234567890  |
    Then the response status should be 400
    And the response "error.code" should be "VALIDATION_ERROR"

  Scenario: Register a patient without required phone
    When I send a POST request to "/api/v1/patients" without "phone_primary"
    Then the response status should be 400
```

### Feature: Patient Search and Pagination

```gherkin
Feature: Patient Search
  As a user with patient:read permission
  I want to search for patients
  So that I can quickly find patient records

  Background:
    Given I am logged in as a user with "patient:read" permission
    And the following patients exist:
      | first_name | last_name | mrn      |
      | John       | Doe       | MRN-0001 |
      | Jane       | Doe       | MRN-0002 |
      | Alice      | Smith     | MRN-0003 |

  Scenario: Search by last name
    When I send a GET request to "/api/v1/patients?search=Doe"
    Then the response should contain 2 patients
    And all results should have last_name "Doe"

  Scenario: Search by MRN
    When I send a GET request to "/api/v1/patients?search=MRN-0001"
    Then the response should contain 1 patient
    And the patient "mrn" should be "MRN-0001"

  Scenario: Paginated results with limit
    When I send a GET request to "/api/v1/patients?limit=2"
    Then the response should contain at most 2 patients
    And the response should contain a cursor for the next page

  Scenario: Search by phone number
    When I send a GET request to "/api/v1/patients?search=+1234567890"
    Then the response should contain patients with matching phone_primary or phone_secondary
    And the phone search should be case-insensitive

  Scenario: Filter by status
    Given patients exist with status "Active", "Inactive", and "Deceased"
    When I send a GET request to "/api/v1/patients?status=Active"
    Then the response should contain only patients with status "Active"

  Scenario: Combine search and status filter
    When I send a GET request to "/api/v1/patients?search=Doe&status=Active"
    Then the response should contain only active patients matching "Doe"

  Scenario: Cursor-based pagination
    Given I fetched the first page with limit=2 and received a cursor
    When I send a GET request to "/api/v1/patients?limit=2&cursor=<cursor>"
    Then the response should contain the next batch of patients
```

### Feature: Patient Update

```gherkin
Feature: Patient Update
  As a user with patient:update permission
  I want to update patient records
  So that patient information stays current

  Background:
    Given I am logged in as a user with "patient:update" permission
    And a patient "John Doe" exists with id "<patient_id>"

  Scenario: Update patient contact information
    When I send a PATCH request to "/api/v1/patients/<patient_id>" with:
      | phone_primary | +9876543210     |
      | email         | john@newmail.com |
    Then the response status should be 200
    And the response "phone_primary" should be "+9876543210"
    And the response "updated_by" should be the authenticated user's ID

  Scenario: Change patient status
    When I send a PATCH request to "/api/v1/patients/<patient_id>" with:
      | status | Inactive |
    Then the response status should be 200
    And the response "status" should be "Inactive"

  Scenario: Update patient with invalid status
    When I send a PATCH request to "/api/v1/patients/<patient_id>" with:
      | status | InvalidStatus |
    Then the response status should be 400
```

---

## 4. Appointment Booking

### Feature: Appointment Slot Management

```gherkin
Feature: Appointment Slot Management
  As an admin
  I want to create appointment slots for providers
  So that patients can book appointments

  Background:
    Given I am logged in as a user with "appointment:create" permission
    And a staff member "Dr. Smith" exists with id "<provider_id>"
    And a department "Cardiology" exists with id "<dept_id>"

  Scenario: Create a 30-minute consultation slot block
    When I send a POST request to "/api/v1/appointments/slots" with:
      | provider_id           | <provider_id>          |
      | department_id         | <dept_id>              |
      | date                  | 2026-03-15             |
      | start_time            | 2026-03-15T09:00:00Z   |
      | end_time              | 2026-03-15T17:00:00Z   |
      | slot_duration_minutes | 30                     |
      | appointment_type      | Consultation           |
    Then the response status should be 201
    And the slot "status" should be "Available"
    And the slot "booked_count" should be 0

  Scenario: Create slot with invalid duration (too short)
    When I send a POST request to "/api/v1/appointments/slots" with:
      | slot_duration_minutes | 3 |
    Then the response status should be 400
    And the response should indicate minimum duration is 5 minutes

  Scenario: Create slot with duration exceeding max
    When I send a POST request to "/api/v1/appointments/slots" with:
      | slot_duration_minutes | 500 |
    Then the response status should be 400
```

### Feature: Appointment Booking

```gherkin
Feature: Book an Appointment
  As a receptionist
  I want to book appointments for patients
  So that they can see a provider at a scheduled time

  Background:
    Given I am logged in as a user with "appointment:create" permission
    And a patient "John Doe" exists with id "<patient_id>"
    And a provider "Dr. Smith" exists with id "<provider_id>"
    And a department "Cardiology" exists with id "<dept_id>"
    And an available slot exists for 2026-03-15 09:00-09:30

  Scenario: Book a consultation appointment
    When I send a POST request to "/api/v1/appointments" with:
      | patient_id       | <patient_id>          |
      | provider_id      | <provider_id>         |
      | department_id    | <dept_id>             |
      | appointment_type | Consultation          |
      | scheduled_start  | 2026-03-15T09:00:00Z  |
      | scheduled_end    | 2026-03-15T09:30:00Z  |
      | reason           | Annual checkup        |
    Then the response status should be 201
    And the response "status" should be "Scheduled"
    And the response "appointment_type" should be "Consultation"
    And the response "created_by" should be the authenticated user's ID

  Scenario: Book an urgent appointment
    When I send a POST request to "/api/v1/appointments" with priority "Urgent"
    Then the response status should be 201
    And the response "priority" should be "Urgent"

  Scenario: Book a telehealth appointment
    When I send a POST request to "/api/v1/appointments" with:
      | is_telehealth | true |
    Then the response status should be 201
    And the response "is_telehealth" should be true

  Scenario: Book appointment without required reason
    When I send a POST request to "/api/v1/appointments" without "reason"
    Then the response status should be 400
    And the response "error.code" should be "VALIDATION_ERROR"
```

### Feature: Appointment Check-In

```gherkin
Feature: Patient Check-In
  As a receptionist
  I want to check in patients for their appointments
  So that providers know the patient has arrived

  Background:
    Given I am logged in as a user with "appointment:update" permission
    And an appointment exists with id "<appt_id>" and status "Scheduled"

  Scenario: Successful check-in
    When I send a POST request to "/api/v1/appointments/<appt_id>/check-in"
    Then the response status should be 200
    And the response "checked_in_at" should be set
    And the response "checked_in_by" should be the authenticated user's ID

  Scenario: Check-in updates appointment status
    When I send a POST request to "/api/v1/appointments/<appt_id>/check-in"
    Then the appointment status should be updated to "CheckedIn"
```

### Feature: Appointment Filtering

```gherkin
Feature: Appointment Filtering
  As a provider or receptionist
  I want to filter appointments by various criteria
  So that I can view relevant schedules

  Background:
    Given I am logged in as a user with "appointment:read" permission
    And multiple appointments exist for different dates, providers, and statuses

  Scenario: Filter by date
    When I send a GET request to "/api/v1/appointments?date=2026-03-15"
    Then all returned appointments should be scheduled for 2026-03-15

  Scenario: Filter by provider
    When I send a GET request to "/api/v1/appointments?provider_id=<provider_id>"
    Then all returned appointments should have provider_id "<provider_id>"

  Scenario: Filter by patient
    When I send a GET request to "/api/v1/appointments?patient_id=<patient_id>"
    Then all returned appointments should have patient_id "<patient_id>"

  Scenario: Filter by status
    When I send a GET request to "/api/v1/appointments?status=Scheduled"
    Then all returned appointments should have status "Scheduled"

  Scenario: Get available slots for a date
    When I send a GET request to "/api/v1/appointments/slots?date=2026-03-15&provider_id=<provider_id>"
    Then the response should contain available slots for that provider on that date
```

---

## 5. Staff Management

### Feature: Staff Registration

```gherkin
Feature: Staff Registration
  As a tenant admin
  I want to register new staff members
  So that they can access the system and be assigned to departments

  Background:
    Given I am logged in as a user with "staff:create" permission
    And a department "Cardiology" exists with id "<dept_id>"

  Scenario: Register a doctor with required fields
    When I send a POST request to "/api/v1/staff" with:
      | first_name      | Jane              |
      | last_name       | Smith             |
      | date_of_birth   | 1985-06-15        |
      | gender          | Female            |
      | email           | jane@hospital.com |
      | phone           | +1234567890       |
      | staff_type      | Doctor            |
      | designation     | Senior Cardiologist |
      | employment_type | FullTime          |
      | hire_date       | 2020-01-15        |
      | password        | SecurePass123!    |
    Then the response status should be 201
    And the response should contain "id"
    And the response should contain "employee_id"
    And a User record should be created with email "jane@hospital.com"

  Scenario: Register staff with department assignment
    When I send a POST request to "/api/v1/staff" with department_id "<dept_id>"
    Then the response status should be 201
    And the staff member should be linked to department "Cardiology"

  Scenario: Register staff with specializations
    When I send a POST request to "/api/v1/staff" with:
      | specializations | ["Cardiology", "Internal Medicine"] |
    Then the response status should be 201
    And the response "specializations" should contain "Cardiology"

  Scenario: Register staff with duplicate email
    Given a staff member exists with email "jane@hospital.com"
    When I send a POST request to "/api/v1/staff" with email "jane@hospital.com"
    Then the response status should be 409
```

### Feature: Staff Credential Management

```gherkin
Feature: Staff Credentials
  As an admin
  I want to manage staff credentials and certifications
  So that we maintain compliance with regulatory requirements

  Background:
    Given I am logged in as a user with "staff:update" permission
    And a staff member "Dr. Smith" exists with id "<staff_id>"

  Scenario: Add a medical license credential
    When I send a POST request to "/api/v1/staff/<staff_id>/credentials" with:
      | type               | License             |
      | name               | Medical License     |
      | issuing_authority  | State Medical Board |
      | credential_number  | LIC-123456          |
      | issue_date         | 2023-01-01          |
      | expiry_date        | 2028-01-01          |
      | auto_alert_days    | 90                  |
    Then the response status should be 201
    And the response "status" should be "Active"

  Scenario: View staff credentials
    Given the staff member has 2 credentials on file
    When I send a GET request to "/api/v1/staff/<staff_id>/credentials"
    Then the response should contain 2 credentials

  Scenario: Add credential with missing required fields
    When I send a POST request to "/api/v1/staff/<staff_id>/credentials" without "credential_number"
    Then the response status should be 400
```

### Feature: Staff Search and Filtering

```gherkin
Feature: Staff Search
  As a user with staff:read permission
  I want to search and filter staff members
  So that I can find the right person quickly

  Background:
    Given I am logged in as a user with "staff:read" permission
    And the following staff members exist:
      | first_name | last_name | staff_type | department   |
      | Jane       | Smith     | Doctor     | Cardiology   |
      | John       | Doe       | Nurse      | Emergency    |
      | Alice      | Johnson   | Doctor     | Cardiology   |

  Scenario: Filter by staff type
    When I send a GET request to "/api/v1/staff?staff_type=Doctor"
    Then the response should contain 2 staff members
    And all results should have staff_type "Doctor"

  Scenario: Filter by department
    When I send a GET request to "/api/v1/staff?department_id=<cardiology_id>"
    Then the response should contain 2 staff members

  Scenario: Search by name
    When I send a GET request to "/api/v1/staff?search=Smith"
    Then the response should contain 1 staff member
    And the result should be "Jane Smith"
```

---

## 6. RBAC Enforcement

### Feature: Role-Based Access Control

```gherkin
Feature: RBAC Enforcement
  As the system
  I want to enforce role-based permissions
  So that users can only access resources they are authorized for

  Scenario: Doctor can read patients
    Given I am logged in with role "doctor"
    When I send a GET request to "/api/v1/patients"
    Then the response status should be 200

  Scenario: Doctor cannot create staff
    Given I am logged in with role "doctor"
    When I send a POST request to "/api/v1/staff" with valid staff data
    Then the response status should be 403
    And the response "error.code" should be "FORBIDDEN"

  Scenario: Receptionist can create patients
    Given I am logged in with role "receptionist"
    When I send a POST request to "/api/v1/patients" with valid patient data
    Then the response status should be 201

  Scenario: Receptionist cannot delete departments
    Given I am logged in with role "receptionist"
    When I send a DELETE request to "/api/v1/departments/<dept_id>"
    Then the response status should be 403

  Scenario: Nurse can read patients but not create them
    Given I am logged in with role "nurse"
    When I send a GET request to "/api/v1/patients"
    Then the response status should be 200
    When I send a POST request to "/api/v1/patients" with valid patient data
    Then the response status should be 403

  Scenario: Tenant admin has full module access
    Given I am logged in with role "tenant_admin"
    When I send a POST request to "/api/v1/staff" with valid staff data
    Then the response status should be 201
    When I send a POST request to "/api/v1/departments" with valid department data
    Then the response status should be 201
    When I send a DELETE request to "/api/v1/departments/<dept_id>"
    Then the response status should be 200

  Scenario: Only super_admin can manage tenants
    Given I am logged in with role "tenant_admin"
    When I send a POST request to "/api/v1/tenants" with valid tenant data
    Then the response status should be 403

  Scenario: Super admin can manage tenants
    Given I am logged in with role "super_admin"
    When I send a POST request to "/api/v1/tenants" with valid tenant data
    Then the response status should be 201

  Scenario: Billing clerk can read patients and billing but not update patients
    Given I am logged in with role "billing_clerk"
    When I send a GET request to "/api/v1/patients"
    Then the response status should be 200
    When I send a PATCH request to "/api/v1/patients/<patient_id>"
    Then the response status should be 403

  Scenario: Unauthenticated request is rejected
    When I send a GET request to "/api/v1/patients" without a Bearer token
    Then the response status should be 401
    And the response "error.code" should be "UNAUTHORIZED"

  Scenario: Expired JWT is rejected
    Given I have an expired JWT access token
    When I send a GET request to "/api/v1/patients" with the expired token
    Then the response status should be 401

  Scenario: Public endpoints bypass auth
    When I send a GET request to "/api/v1/health" without a Bearer token
    Then the response status should be 200
    When I send a POST request to "/api/v1/auth/login" without a Bearer token
    Then the response should not be 401
```

---

## 7. Department Management

### Feature: Department CRUD

```gherkin
Feature: Department Management
  As a tenant admin
  I want to manage departments
  So that the hospital organizational structure is maintained

  Background:
    Given I am logged in as a user with "department:create" and "department:read" permissions

  Scenario: Create a clinical department
    When I send a POST request to "/api/v1/departments" with:
      | name | Cardiology |
      | code | CARD       |
      | type | Clinical   |
    Then the response status should be 201
    And the response "is_active" should be true

  Scenario: Create a department with hierarchy
    Given a department "Medicine" exists with id "<parent_id>"
    When I send a POST request to "/api/v1/departments" with:
      | name                 | Cardiology   |
      | code                 | CARD         |
      | type                 | Clinical     |
      | parent_department_id | <parent_id>  |
    Then the response status should be 201
    And the department should be a child of "Medicine"

  Scenario: Create department with duplicate code
    Given a department with code "CARD" already exists
    When I send a POST request to "/api/v1/departments" with code "CARD"
    Then the response status should be 409

  Scenario: List active departments
    When I send a GET request to "/api/v1/departments"
    Then the response should contain only active departments

  Scenario: List all departments including inactive
    When I send a GET request to "/api/v1/departments?include_inactive=true"
    Then the response should contain both active and inactive departments

  Scenario: Deactivate a department
    Given I have "department:delete" permission
    And a department "Radiology" exists with id "<dept_id>"
    When I send a DELETE request to "/api/v1/departments/<dept_id>"
    Then the department "is_active" should be false
    And the department should still exist in the database
```

---

## 8. Input Validation and Error Handling

### Feature: Input Validation

```gherkin
Feature: Input Validation
  As the system
  I want to validate all incoming requests
  So that malformed data is rejected before processing

  Scenario: Reject extra fields not in DTO (forbidNonWhitelisted)
    Given I am logged in as a user with "patient:create" permission
    When I send a POST request to "/api/v1/patients" with an extra field "hacker_field"
    Then the response status should be 400
    And the response "error.code" should be "VALIDATION_ERROR"

  Scenario: Reject invalid UUID format
    Given I am logged in as a user with "appointment:create" permission
    When I send a POST request to "/api/v1/appointments" with:
      | patient_id | not-a-valid-uuid |
    Then the response status should be 400

  Scenario: Reject invalid date format
    Given I am logged in as a user with "patient:create" permission
    When I send a POST request to "/api/v1/patients" with:
      | date_of_birth | 15-01-1990 |
    Then the response status should be 400

  Scenario: Reject invalid enum value
    Given I am logged in as a user with "staff:create" permission
    When I send a POST request to "/api/v1/staff" with:
      | staff_type | InvalidType |
    Then the response status should be 400

  Scenario: Error response includes request_id for tracing
    When any request results in an error
    Then the response should contain "meta.request_id"
    And the response should contain "meta.timestamp"
```

---

## 9. Roster Generation

### Feature: Auto-Generate Roster

```gherkin
Feature: Auto-Generate Roster
  As a department head
  I want to auto-generate rosters for my department
  So that staff shifts are assigned fairly and comply with labor rules

  Background:
    Given I am logged in as a user with "roster:create" permission
    And a department "Emergency" exists with id "<dept_id>"
    And the following active staff are assigned to "Emergency":
      | id          | first_name | last_name |
      | <staff_id1> | Alice      | Nurse     |
      | <staff_id2> | Bob        | Nurse     |
    And the following active shift templates exist:
      | id           | name        | start_time | end_time | is_overnight | break_duration_minutes |
      | <template1>  | Day Shift   | 07:00      | 15:00    | false        | 30                     |
      | <template2>  | Night Shift | 23:00      | 07:00    | true         | 30                     |

  Scenario: Successful roster generation
    When I send a POST request to "/api/v1/rostering/generate" with:
      | departmentId | <dept_id>  |
      | startDate    | 2026-03-09 |
      | endDate      | 2026-03-13 |
    Then the response status should be 200
    And the response "generated" should be a non-empty array
    And each generated schedule should have "status" equal to "Scheduled"
    And each generated schedule should have "type" equal to "Regular"
    And the response should contain "conflicts"

  Scenario: Overtime limit exceeded -- staff excluded from assignment
    Given staff "Alice Nurse" already has 38 scheduled hours for the week of 2026-03-09
    And the shift template "Day Shift" is 7.5 hours (after 30-min break)
    When I send a POST request to "/api/v1/rostering/generate" with:
      | departmentId | <dept_id>  |
      | startDate    | 2026-03-09 |
      | endDate      | 2026-03-09 |
    Then staff "Alice Nurse" should NOT be assigned to any shift for 2026-03-09
    And the 40-hour weekly maximum should not be exceeded for any staff member

  Scenario: Insufficient rest period -- staff excluded from assignment
    Given staff "Bob Nurse" has a shift ending at 2026-03-09T23:00:00
    And the "Day Shift" template starts at 07:00 on 2026-03-10 (only 8 hours gap)
    When I send a POST request to "/api/v1/rostering/generate" with:
      | departmentId | <dept_id>  |
      | startDate    | 2026-03-10 |
      | endDate      | 2026-03-10 |
    Then staff "Bob Nurse" should NOT be assigned to the "Day Shift" on 2026-03-10
    And the 11-hour minimum rest period should be respected
```

---

## 10. Shift Swap Workflow

### Feature: Shift Swap Request

```gherkin
Feature: Shift Swap Request
  As a staff member
  I want to request a shift swap with a colleague
  So that I can manage my schedule when personal conflicts arise

  Background:
    Given I am logged in as staff "Alice Nurse" with id "<alice_id>"
    And a schedule exists with id "<schedule_id>" assigned to "<alice_id>"
    And staff "Bob Nurse" exists with id "<bob_id>"
    And a user "Manager" exists with id "<manager_id>" and "roster:approve" permission

  Scenario: Full swap lifecycle -- request, accept, approve
    # Step 1: Alice requests the swap
    When I send a POST request to "/api/v1/rostering/swaps" with:
      | schedule_id | <schedule_id>               |
      | reason      | Personal appointment conflict |
    Then the response status should be 201
    And the response "status" should be "Pending"
    And the response "requester_id" should be "<alice_id>"
    And the response should contain "id" as "<swap_id>"

    # Step 2: Bob accepts the swap
    Given I am logged in as staff "Bob Nurse" with id "<bob_id>"
    When I send a POST request to "/api/v1/rostering/swaps/<swap_id>/accept" with:
      | accepter_response | Happy to cover for you |
    Then the response status should be 200
    And the response "status" should be "Accepted"
    And the response "accepter_id" should be "<bob_id>"
    And the response "accepted_at" should be set

    # Step 3: Manager approves the swap
    Given I am logged in as "Manager" with id "<manager_id>"
    When I send a POST request to "/api/v1/rostering/swaps/<swap_id>/approve"
    Then the response status should be 200
    And the response "status" should be "Approved"
    And the response "approver_id" should be "<manager_id>"
    And the response "approved_at" should be set
    And the schedule "<schedule_id>" should now have "staff_id" equal to "<bob_id>"
    And the schedule "<schedule_id>" should have "original_staff_id" equal to "<alice_id>"

  Scenario: Manager rejects a swap request
    Given a swap request "<swap_id>" exists with status "Accepted"
    And I am logged in as "Manager" with id "<manager_id>"
    When I send a POST request to "/api/v1/rostering/swaps/<swap_id>/reject" with:
      | rejection_reason | Insufficient coverage for that shift |
    Then the response status should be 200
    And the response "status" should be "Rejected"
    And the response "rejection_reason" should be "Insufficient coverage for that shift"
    And the schedule staff assignment should remain unchanged

  Scenario: Staff cannot accept their own swap request
    When I send a POST request to "/api/v1/rostering/swaps/<swap_id>/accept" with:
      | accepter_response | I will take it myself |
    Then the response status should be 400
    And the response "message" should be "You cannot accept your own swap request"

  Scenario: Cannot swap someone else's shift
    Given a schedule exists with id "<other_schedule>" assigned to "<bob_id>"
    When I send a POST request to "/api/v1/rostering/swaps" with:
      | schedule_id | <other_schedule>  |
      | reason      | Want to switch     |
    Then the response status should be 400
    And the response "message" should be "You can only request swaps for your own shifts"
```

---

## 11. Clock In/Out

### Feature: Staff Clock In and Clock Out

```gherkin
Feature: Staff Clock In and Clock Out
  As a staff member
  I want to clock in and out of my shifts
  So that my attendance and hours are accurately recorded

  Background:
    Given I am logged in as staff "Alice Nurse" with id "<alice_id>"
    And a schedule exists with id "<schedule_id>" assigned to "<alice_id>"
    And the schedule "start_time" is "2026-03-10T07:00:00Z"
    And the schedule "end_time" is "2026-03-10T15:00:00Z"

  Scenario: On-time clock-in
    Given the current time is "2026-03-10T06:55:00Z"
    When I send a POST request to "/api/v1/rostering/clock-in" with:
      | schedule_id | <schedule_id> |
      | notes       | Starting shift |
    Then the response status should be 201
    And the response "is_late" should be false
    And the response "late_minutes" should be 0
    And the response "clock_in" should be set
    And the response should contain "id" as "<clock_record_id>"

  Scenario: Late clock-in with late_minutes calculated
    Given the current time is "2026-03-10T07:22:00Z"
    When I send a POST request to "/api/v1/rostering/clock-in" with:
      | schedule_id | <schedule_id> |
    Then the response status should be 201
    And the response "is_late" should be true
    And the response "late_minutes" should be 22

  Scenario: Clock-out with overtime calculation
    Given a clock record "<clock_record_id>" exists with clock_in at "2026-03-10T07:00:00Z"
    And the current time is "2026-03-10T17:30:00Z" (10.5 hours after clock-in)
    When I send a POST request to "/api/v1/rostering/clock-out/<clock_record_id>" with:
      | notes | Stayed for handover |
    Then the response status should be 200
    And the response "clock_out" should be set
    And the response "actual_hours" should be 10.5
    And the response "overtime_hours" should be 2.5

  Scenario: Cannot clock in twice for the same schedule
    Given I have already clocked in for schedule "<schedule_id>"
    When I send a POST request to "/api/v1/rostering/clock-in" with:
      | schedule_id | <schedule_id> |
    Then the response status should be 400
    And the response "message" should be "Already clocked in for this schedule"

  Scenario: Cannot clock out another staff member's record
    Given a clock record "<other_record>" belongs to staff "<bob_id>"
    When I send a POST request to "/api/v1/rostering/clock-out/<other_record>"
    Then the response status should be 400
    And the response "message" should be "You can only clock out your own record"
```

---

## 12. Patient Admission (Bed Management)

### Feature: Patient Admission

```gherkin
Feature: Patient Admission
  As a nurse or doctor
  I want to admit a patient to a bed
  So that inpatient care can begin

  Background:
    Given I am logged in as a user with "bed:update" permission
    And a patient "John Doe" exists with id "<patient_id>"
    And a ward "Medical Ward" exists with a room "Room 101"
    And a bed "101-A" exists with id "<bed_id>" and status "Available"

  Scenario: Successful patient admission
    When I send a POST request to "/api/v1/bed-management/admissions" with:
      | patient_id             | <patient_id>     |
      | bed_id                 | <bed_id>         |
      | admission_type         | Emergency        |
      | attending_provider_id  | <provider_id>    |
      | diet_order             | Regular          |
      | notes                  | Fall from height |
    Then the response status should be 201
    And the response "status" should be "Admitted"
    And the response should contain "admission_number"
    And the response "patient.id" should be "<patient_id>"
    And the response "bed.id" should be "<bed_id>"
    And the bed "<bed_id>" status should now be "Occupied"

  Scenario: Admission fails when bed is already occupied
    Given bed "101-A" has status "Occupied"
    When I send a POST request to "/api/v1/bed-management/admissions" with:
      | patient_id | <patient_id> |
      | bed_id     | <bed_id>     |
    Then the response status should be 400
    And the response "message" should contain "Bed is not available"

  Scenario: Admission fails when bed is in Cleaning status
    Given bed "101-A" has status "Cleaning"
    When I send a POST request to "/api/v1/bed-management/admissions" with:
      | patient_id | <patient_id> |
      | bed_id     | <bed_id>     |
    Then the response status should be 400
    And the response "message" should contain "Bed is not available"

  Scenario: Admission fails when patient does not exist
    When I send a POST request to "/api/v1/bed-management/admissions" with:
      | patient_id | non-existent-uuid |
      | bed_id     | <bed_id>          |
    Then the response status should be 404
    And the response "message" should be "Patient not found"
```

---

## 13. Patient Discharge

### Feature: Patient Discharge

```gherkin
Feature: Patient Discharge
  As a doctor
  I want to discharge a patient
  So that the bed is freed and housekeeping is triggered

  Background:
    Given I am logged in as a user with "bed:update" permission
    And an admission exists with id "<admission_id>" and status "Admitted"
    And the admission is for bed "<bed_id>" with status "Occupied"

  Scenario: Successful discharge with bed set to Cleaning
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/discharge" with:
      | discharge_type        | Routine                       |
      | discharge_summary     | Patient recovered fully       |
      | discharge_disposition | Home                          |
    Then the response status should be 200
    And the response "status" should be "Discharged"
    And the response "actual_discharge" should be set
    And the response "discharge_type" should be "Routine"
    And the bed "<bed_id>" status should now be "Cleaning"
    And a housekeeping log should exist for bed "<bed_id>" with status "Dirty"

  Scenario: Cannot discharge a patient who is already discharged
    Given the admission "<admission_id>" has status "Discharged"
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/discharge" with:
      | discharge_type    | Routine              |
      | discharge_summary | Already discharged   |
    Then the response status should be 400
    And the response "message" should contain "Cannot discharge admission with status"
```

---

## 14. Bed Transfer

### Feature: Bed Transfer

```gherkin
Feature: Bed Transfer
  As a doctor or nurse
  I want to transfer a patient to a different bed
  So that patient placement can be adjusted based on clinical needs

  Background:
    Given I am logged in as a user with "bed:update" permission
    And an admission exists with id "<admission_id>" and status "Admitted"
    And the admission is for bed "<source_bed>" with status "Occupied"
    And a target bed "<target_bed>" exists with status "Available"

  Scenario: Successful bed transfer
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/transfer" with:
      | to_bed_id | <target_bed>              |
      | reason    | Patient needs isolation   |
    Then the response status should be 200
    And the response "from_bed_id" should be "<source_bed>"
    And the response "to_bed_id" should be "<target_bed>"
    And the response "status" should be "Completed"
    And the source bed "<source_bed>" status should now be "Cleaning"
    And the target bed "<target_bed>" status should now be "Occupied"
    And a housekeeping log should exist for source bed "<source_bed>" with status "Dirty"

  Scenario: Transfer fails when target bed is unavailable
    Given the target bed "<target_bed>" has status "Occupied"
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/transfer" with:
      | to_bed_id | <target_bed>            |
      | reason    | Needs a private room    |
    Then the response status should be 400
    And the response "message" should contain "Target bed is not available"

  Scenario: Department transfer with bed reassignment
    Given a target department "Cardiology" exists with id "<cardiology_dept>"
    And a target bed "<cardio_bed>" exists in "Cardiology" with status "Available"
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/department-transfer" with:
      | to_department_id | <cardiology_dept>             |
      | to_bed_id        | <cardio_bed>                  |
      | reason           | Transfer to cardiology care   |
    Then the response status should be 200
    And the source bed "<source_bed>" status should now be "Cleaning"
    And the target bed "<cardio_bed>" status should now be "Occupied"

  Scenario: Department transfer fails when bed does not belong to department
    Given a target bed "<wrong_bed>" exists in department "Neurology" with status "Available"
    When I send a POST request to "/api/v1/bed-management/admissions/<admission_id>/department-transfer" with:
      | to_department_id | <cardiology_dept> |
      | to_bed_id        | <wrong_bed>       |
      | reason           | Transfer needed   |
    Then the response status should be 400
    And the response "message" should be "Target bed does not belong to the specified department"
```

---

## 15. Imaging Order

### Feature: Imaging Order Management

```gherkin
Feature: Imaging Order Management
  As a doctor
  I want to order imaging studies for my patients
  So that diagnostic images support clinical decision-making

  Background:
    Given I am logged in as a user with "radiology:create" permission
    And a patient "Jane Doe" exists with id "<patient_id>"

  Scenario: Create an imaging order
    When I send a POST request to "/api/v1/radiology/orders" with:
      | patient_id            | <patient_id>                |
      | modality              | CT                          |
      | body_part             | Chest                       |
      | laterality            | Bilateral                   |
      | priority              | Urgent                      |
      | clinical_indication   | Rule out pneumonia          |
      | contrast_required     | true                        |
      | contrast_type         | IV Iodinated                |
    Then the response status should be 201
    And the response "status" should be "Ordered"
    And the response should contain "order_number"
    And the response "modality" should be "CT"
    And the response "body_part" should be "Chest"
    And the response "priority" should be "Urgent"
    And the response "ordered_by" should be the authenticated user's ID

  Scenario: Schedule an imaging order
    Given an imaging order exists with id "<order_id>" and status "Ordered"
    When I send a POST request to "/api/v1/radiology/orders/<order_id>/schedule" with:
      | scheduled_at | 2026-03-15T10:00:00Z |
    Then the response status should be 200
    And the response "status" should be "Scheduled"
    And the response "scheduled_at" should be set

  Scenario: Complete an imaging order
    Given an imaging order exists with id "<order_id>" and status "Scheduled"
    When I send a POST request to "/api/v1/radiology/orders/<order_id>/complete"
    Then the response status should be 200
    And the response "status" should be "Completed"
    And the response "completed_at" should be set

  Scenario: Cancel an imaging order
    Given an imaging order exists with id "<order_id>" and status "Ordered"
    When I send a POST request to "/api/v1/radiology/orders/<order_id>/cancel"
    Then the response status should be 200
    And the response "status" should be "Cancelled"
```

---

## 16. Radiology Report

### Feature: Radiology Report Workflow

```gherkin
Feature: Radiology Report Workflow
  As a radiologist
  I want to create, sign, and verify radiology reports
  So that imaging findings are formally documented and communicated

  Background:
    Given I am logged in as a user with "radiology:create" permission
    And an imaging order exists with id "<order_id>"

  Scenario: Create a draft report
    When I send a POST request to "/api/v1/radiology/orders/<order_id>/reports" with:
      | findings          | Consolidation in right lower lobe            |
      | impression        | Findings consistent with community-acquired pneumonia |
      | recommendation    | Follow-up chest X-ray in 4-6 weeks           |
      | clinical_history  | 3-day history of cough and fever              |
      | technique         | CT chest with IV contrast                     |
    Then the response status should be 201
    And the response "status" should be "Draft"
    And the response "version" should be 1
    And the response "reported_by" should be the authenticated user's ID
    And the response should contain "id" as "<report_id>"

  Scenario: Sign report as Final
    Given a radiology report "<report_id>" exists with status "Draft"
    When I send a POST request to "/api/v1/radiology/reports/<report_id>/sign"
    Then the response status should be 200
    And the response "status" should be "Final"
    And the response "signed_by" should be the authenticated user's ID
    And the response "signed_at" should be set
    And the imaging order "<order_id>" status should now be "Reported"

  Scenario: Verify a finalized report
    Given a radiology report "<report_id>" exists with status "Final"
    And I am logged in as an attending radiologist with id "<attending_id>"
    When I send a POST request to "/api/v1/radiology/reports/<report_id>/verify"
    Then the response status should be 200
    And the response "verified_by" should be "<attending_id>"
    And the response "verified_at" should be set

  Scenario: Cannot edit a finalized report
    Given a radiology report "<report_id>" exists with status "Final"
    When I send a PATCH request to "/api/v1/radiology/reports/<report_id>" with:
      | findings | Updated findings |
    Then the response status should be 400
    And the response "message" should be "Cannot edit a finalized report. Use addendum instead."

  Scenario: Cannot verify a report that is not Final
    Given a radiology report "<report_id>" exists with status "Draft"
    When I send a POST request to "/api/v1/radiology/reports/<report_id>/verify"
    Then the response status should be 400
    And the response "message" should be "Report must be signed (Final status) before verification."
```

---

## 17. Critical Finding Alert

### Feature: Critical Radiology Finding

```gherkin
Feature: Critical Radiology Finding
  As a radiologist
  I want to flag critical findings
  So that the ordering provider is immediately notified

  Background:
    Given I am logged in as a radiologist with "radiology:update" permission
    And an imaging order exists with id "<order_id>" ordered by provider "<ordering_provider_id>"
    And the order is for patient "<patient_id>" with order_number "IMG-ABC123"
    And a radiology report "<report_id>" exists for the order

  Scenario: Report a critical finding and notify ordering provider
    When I send a POST request to "/api/v1/radiology/reports/<report_id>/critical-finding" with:
      | critical_finding | Large pleural effusion with mediastinal shift |
    Then the response status should be 200
    And the response "is_critical" should be true
    And the response "critical_finding" should be "Large pleural effusion with mediastinal shift"
    And the response "critical_notified_at" should be set
    And the response "critical_notified_to" should be "<ordering_provider_id>"
    And a notification should be sent to "<ordering_provider_id>" with:
      | type     | CriticalRadiology                                                                     |
      | category | Clinical                                                                              |
      | subject  | Critical Radiology Finding                                                            |
      | body     | Critical finding for order IMG-ABC123: Large pleural effusion with mediastinal shift   |
```

---

## 18. Break-the-Glass Emergency Access

### Feature: Break-the-Glass Access

```gherkin
Feature: Break-the-Glass Emergency Access
  As a staff member
  I want to access patient records in an emergency
  So that I can provide timely care even without normal permissions

  Background:
    Given I am logged in as staff with id "<user_id>"
    And a patient exists with id "<patient_id>"

  Scenario: Emergency access granted with audit log and notification
    When I send a POST request to "/api/v1/break-the-glass" with:
      | patient_id     | <patient_id>                    |
      | justification  | Patient unresponsive in hallway |
      | resource_type  | MedicalRecord                   |
      | resource_id    | <record_id>                     |
      | access_type    | Read                            |
      | ip_address     | 192.168.1.100                   |
      | user_agent     | Mozilla/5.0                     |
    Then the response status should be 201
    And the response "user_id" should be "<user_id>"
    And the response "patient_id" should be "<patient_id>"
    And the response "justification" should be "Patient unresponsive in hallway"
    And the response "privacy_officer_notified" should be true
    And the response "notified_at" should be set
    And a notification should be sent with:
      | type     | alert                                                                                          |
      | category | Clinical                                                                                       |
      | subject  | Break-the-Glass Access                                                                         |
      | body     | Emergency access initiated for patient <patient_id>. Justification: Patient unresponsive in hallway |

  Scenario: Admin reviews a break-the-glass log
    Given a break-the-glass log "<log_id>" exists that has not been reviewed
    And I am logged in as a privacy officer
    When I send a POST request to "/api/v1/break-the-glass/<log_id>/review" with:
      | outcome | Justified |
    Then the response status should be 200
    And the response "privacy_officer_notified" should be true
    And the response "notified_at" should be set

  Scenario: Cannot review an already-reviewed log
    Given a break-the-glass log "<log_id>" exists that has already been reviewed
    When I send a POST request to "/api/v1/break-the-glass/<log_id>/review" with:
      | outcome | Justified |
    Then the response status should be 400
    And the response "message" should be "This log has already been reviewed"
```

---

## 19. Payment Plan

### Feature: Payment Plan Creation

```gherkin
Feature: Payment Plan Creation
  As a billing manager
  I want to create installment payment plans for invoices
  So that patients can pay large bills in manageable amounts

  Background:
    Given I am logged in as a user with "billing:create" permission
    And an invoice exists with id "<invoice_id>" and total_amount 1200.00

  Scenario: Create a monthly payment plan with correct installment amounts
    When I send a POST request to "/api/v1/billing/invoices/<invoice_id>/payment-plans" with:
      | patient_id               | <patient_id> |
      | number_of_installments   | 4            |
      | frequency                | Monthly      |
      | start_date               | 2026-04-01   |
      | down_payment             | 200          |
      | notes                    | 4-month plan |
    Then the response status should be 201
    And the response "status" should be "Active"
    And the response "total_amount" should be 1200
    And the response "down_payment" should be 200
    And the response "installment_amount" should be 250
    And the response "number_of_installments" should be 4
    And the response "installments" should contain 4 items
    And installment 1 should have "due_date" of "2026-04-01" and "amount" of 250 and "status" of "Pending"
    And installment 2 should have "due_date" of "2026-05-01" and "amount" of 250
    And installment 3 should have "due_date" of "2026-06-01" and "amount" of 250
    And installment 4 should have "due_date" of "2026-07-01" and "amount" of 250

  Scenario: Create a weekly payment plan without down payment
    When I send a POST request to "/api/v1/billing/invoices/<invoice_id>/payment-plans" with:
      | patient_id               | <patient_id> |
      | number_of_installments   | 3            |
      | frequency                | Weekly       |
      | start_date               | 2026-04-01   |
    Then the response status should be 201
    And the response "down_payment" should be 0
    And the response "installment_amount" should be 400
    And the response "installments" should contain 3 items
    And installment 1 should have "due_date" of "2026-04-01"
    And installment 2 should have "due_date" of "2026-04-08"
    And installment 3 should have "due_date" of "2026-04-15"

  Scenario: Down payment must be less than invoice total
    When I send a POST request to "/api/v1/billing/invoices/<invoice_id>/payment-plans" with:
      | patient_id               | <patient_id> |
      | number_of_installments   | 3            |
      | frequency                | Monthly      |
      | start_date               | 2026-04-01   |
      | down_payment             | 1500         |
    Then the response status should be 400
    And the response "message" should be "Down payment must be less than invoice total"

  Scenario: Invoice not found
    When I send a POST request to "/api/v1/billing/invoices/non-existent-uuid/payment-plans" with:
      | patient_id               | <patient_id> |
      | number_of_installments   | 3            |
      | frequency                | Monthly      |
      | start_date               | 2026-04-01   |
    Then the response status should be 404
    And the response "message" should be "Invoice not found"
```

---

## 20. Pharmacy Expiry Check

### Feature: Pharmacy Expiry Enforcement

```gherkin
Feature: Pharmacy Expiry Enforcement (BR-04)
  As the system
  I want to block dispensing of expired medications
  So that patient safety is ensured per business rule BR-04

  Background:
    Given a medication "Amoxicillin 500mg" exists with id "<med_id>"
    And a batch exists for "<med_id>" with expiry_date "2026-02-28"
    And the current date is "2026-03-03"

  Scenario: Dispensing is blocked for expired medication
    Given I am logged in as a pharmacist with "pharmacy:dispense" permission
    When I attempt to dispense medication "<med_id>" from the expired batch
    Then the response status should be 400
    And the response "message" should contain "expired"
    And the dispensing should be prevented
    And no inventory deduction should occur

  Scenario: Dispensing is allowed for non-expired medication
    Given a batch exists for "<med_id>" with expiry_date "2027-12-31"
    And I am logged in as a pharmacist with "pharmacy:dispense" permission
    When I attempt to dispense medication "<med_id>" from the valid batch
    Then the response status should be 200
    And the dispensing should proceed normally
    And inventory should be deducted accordingly
```
