# Salesforce Integration Methods

This document describes the main integration methods with Salesforce used in the system, specifying the Salesforce object, HTTP method, URL, and the output object (Nimblr).

---

## 1. List Calendars

- **Salesforce Object:** ServiceTerritory
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/queryAll?q=SELECT+Id,+Name,+IsDeleted,+IsActive,+State,+ParentTerritoryId+FROM+ServiceTerritory`
- **Nimblr Output Object:** Calendar[]

---

## 2. List Event Types

- **Salesforce Object:** WorkType
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/queryAll?q=SELECT+Id,+Name,+Description,+DurationInMinutes,+IsDeleted+FROM+WorkType`
- **Nimblr Output Object:** EventType[]

---

## 3. List Events

- **Salesforce Object:** ServiceAppointment
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/queryAll?q=SELECT+...+FROM+ServiceAppointment+WHERE+ServiceTerritoryId='...'`
- **Nimblr Output Object:** Event[]

---

## 4. Get Event

- **Salesforce Object:** ServiceAppointment
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/sobjects/ServiceAppointment/{eventId}`
- **Nimblr Output Object:** Event

---

## 5. Insert Event

- **Salesforce Object:** ServiceAppointment
- **HTTP Method:** POST (Composite)
- **URL:** `/services/data/v48.0/composite`
- **Nimblr Output Object:** Event

---

## 6. Patch Event

- **Salesforce Object:** ServiceAppointment
- **HTTP Method:** POST (Composite PATCH)
- **URL:** `/services/data/v48.0/composite`
- **Nimblr Output Object:** Event

---

## 7. Delete Event

- **Salesforce Object:** ServiceAppointment
- **HTTP Method:** POST (Composite PATCH)
- **URL:** `/services/data/v48.0/composite`
- **Nimblr Output Object:** EventId (string)

---

## 8. List Available Slots

- **Salesforce Object:** TimeSlots (getAppointmentCandidates)
- **HTTP Method:** POST
- **URL:** `/services/data/v48.0/scheduling/getAppointmentCandidates`
- **Nimblr Output Object:** Slot[]

---

## 9. Search Contacts

- **Salesforce Object:** Contact
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/queryAll?q=SELECT+...+FROM+Contact+WHERE+...`
- **Nimblr Output Object:** Contact[]

---

## 10. Get Contact

- **Salesforce Object:** Contact
- **HTTP Method:** GET
- **URL:** `/services/data/v48.0/sobjects/Contact/{contactId}`
- **Nimblr Output Object:** Contact

---

## 11. Insert Contact

- **Salesforce Object:** Contact
- **HTTP Method:** POST (Composite)
- **URL:** `/services/data/v48.0/composite`
- **Nimblr Output Object:** Contact

---

## 12. Acquire Authorization

- **Salesforce Object:** OAuth2 Token
- **HTTP Method:** POST
- **URL:** `/services/oauth2/token`
- **Nimblr Output Object:** Credentials

---
