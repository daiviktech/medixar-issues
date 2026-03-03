# Medixar - Issue Tracker

This repository is for reporting bugs, enhancements, and gaps found during testing of the **Medixar Hospital Management System**.

**Staging URL:** https://medixar.daiviksoft.com

## Login Credentials

| Role | Email | Password |
|------|-------|----------|
| Admin | `admin@medixar.daiviksoft.com` | `Admin123!` |

Leave the **Organization** field blank when logging in.

## How to Report an Issue

1. Go to [**New Issue**](https://github.com/daiviktech/medixar-issues/issues/new/choose)
2. Pick a template: **Bug Report**, **Enhancement**, or **Gap**
3. Fill in all required fields
4. Submit

## Modules Available for Testing

| Module | URL Path | Description |
|--------|----------|-------------|
| Dashboard | `/` | Overview stats, quick actions |
| Patients | `/patients` | Patient CRUD, search, timeline |
| Staff | `/staff` | Staff management, credentials |
| Departments | `/departments` | Department hierarchy |
| Appointments | `/appointments` | Booking, slots, check-in |
| Billing | `/billing` | Invoices, payments, insurance |
| Payment Plans | `/billing/payment-plans` | Installment plans |
| Remittances | `/billing/remittances` | Insurance remittance records |
| Bed Management | `/bed-management` | Wards, rooms, beds, admissions |
| Rostering | `/rostering` | Shift templates, schedules, clock in/out |
| Radiology | `/radiology` | Imaging orders, studies, reports |

## Priority Guide

| Priority | When to use |
|----------|------------|
| **Critical** | App crashes, data loss, can't use a feature at all |
| **High** | Major functionality broken, wrong data displayed |
| **Medium** | Feature works but has issues, UI problems |
| **Low** | Cosmetic issues, minor improvements |

## Tips for Good Bug Reports

- Include **screenshots** or **screen recordings** when possible
- Note the **exact steps** to reproduce the issue
- Mention what you **expected** to happen vs what **actually** happened
- Specify which **browser** you're using if it's a UI issue
