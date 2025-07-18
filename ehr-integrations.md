# Integrations <!-- ========================================================== -->

The calendar module in Nimblr acts as a facade for integrating with external calendar systems, such as EHRs or Google Calendar. It enables Nimblr to manage external events seamlessly, ensuring real-time synchronization and efficient communication with external platforms. This integration allows Nimblr users to manage external events efficiently, reducing manual effort and improving communication with patients.

The calendar module acts as a facade simulating a real Calendar. It manipulates events that do not reside within Nimblr's system. Ensuring seamless synchronization and communication with external platforms.

It is essential to frequently consult the real calendar to ensure updates (new events, cancellations, etc.) are reflected in the shortest possible time.

Nimblr’s API can map any “external” object into a Nimblr object. An external object can be in JSON or XML format.

This document describes:

* **Application overview & core workflows** Holly automates.
* **API methods** exposed by the calendar module.
* **Canonical object models** (Event, Contact, Slot, etc.).
* **A concrete mapping example** from an external EHR payload → Nimblr’s internal schema.
* **Glossary** of key terms.

----
## Index

The following sections provide detailed information about the methods, objects, and examples available in the calendar module. Use the links below to navigate to the relevant section.

| Section | Description |
|---------|-------------|
| [Application Overview](#application-overview) | High-level product description. |
| [Core Workflows](#core-workflows) | How Holly orchestrates patient flows and which API methods she calls. |
| [Synchronization Processes](#synchronization-processes) | Background jobs that keep Nimblr & EHR in sync |
| [Methods](#methods) | Reference for every `#method` endpoint |
| [Objects](#objects) | Data-model reference for Calendar, Event, Contact, etc. |
| [Example](#example) | End-to-end JSON mapping walkthrough. |
| [Glossary](#glossary) | Definitions of common terms used in this doc. |

---

## Application Overview

**Holly** books incoming calls, texts, and web visits **24/7** while managing follow-ups, rescheduling, and wait-lists automatically. Once configured, she handles two high-level workflows:

1. **Inbound Appointment Management**, which includes
   * Scheduling new appointments
   * Search next patient appointment
   * Cancellation and Rescheduling patient appointments

2. **Outreach Appointment Management**, which includes
   * Confirmation
   * Cancellation
   * Rescheduling of (a) patient-canceled appointments, (b) provider-canceled “bumps”, and (c) no-shows
   * Waitlist

Behind the scenes Holly calls the calendar-module endpoints described later in this doc. The next section shows the step-by-step mapping.

---

## Core Workflows

### 1  Inbound Appointment Management <!-- ================================================== -->

#### Schedule a New Appointment
| Step # | Patient / Holly interaction | Calendar-module call(s) |
|-------:|----------------------------|-------------------------|
| 1 | **Patient texts** to request a new visit. | — |
| 2 | Holly asks **“What day works for you?”** and searches for open slots. | [`#listAvailableSlots`](#listavailableslots) |
| 3 | Holly **identifies or creates the patient**. | It is important that the patient can be searched by DOB and social security number [`#searchContacts`](#searchcontacts)  / [`#insertContact`](#insertcontact) |
| 4 | (Optional) Holly **updates demographics** (e-mail, language, gender…). | [`#updateContact`](#updatecontact) with existing `externalId` |
| 5 | (Optional) Patient uploads an **insurance-card photo**. | [`#uploadInsuranceCard`](#uploadinsurancecard) |
| 6 | (Optional) Holly **validates insurance**. | _(If configured, Holly checks whether the patient’s insurance is accepted by the provider.)_ |
| 7 | Holly **books the appointment**. | [`#insertEvent`](#insertevent) |
| 8 | (Optional) Holly **adds extra questions/notes**. | [`#patchEvent`](#patchevent) |

#### Search next patient appointment
| Step # | Patient / Holly interaction | Calendar-module call(s) |
|-------:|----------------------------|-------------------------|
| 1 | **Patient texts** to request to find their next visit. | — |
| 2 | Holly **identifies the patient**. | Patients must be searched by DOB and social security number [`#searchContacts`](#searchcontacts) |
| 3 | Holly searches **next patient appointment**. | [`#listEventsByContact`](#listEventsByContact) queried by patient `externalId` |

#### Cancellation and Rescheduling patient appointments
| Step # | Patient / Holly interaction | Calendar-module call(s) |
|-------:|----------------------------|-------------------------|
| 1 | **Patient texts** to request to cancel/reschedule an appointment. | — |
| 2 | Holly **identifies the patient**. | Patients must be searched by DOB and social security number [`#searchContacts`](#searchcontacts) |
| 3 | Holly searches **patient appointments**. | [`#listEventsByContact`](#listEventsByContact) queried by patient `externalId` |
| 4 | The patient choose to cancel or reschedule the appointment | [`#patchEvent`](#patchevent) using the `externalId` of the event |
| 5 | Holly asks **“What day works for you?”** and searches for open slots. When patient choose to reschedule  | [`#listAvailableSlots`](#listavailableslots) |
| 6 | Holly **books the appointment**. | [`#insertEvent`](#insertevent) |
| 7 | (Optional) Holly **adds conversation notes**. | [`#patchEvent`](#patchevent) |

---

### 2  Outreach Appointment Management <!-- ===================================== -->

Every 20-40 min Holly polls upcoming events (2-14 days out, configurable) and triggers outbound SMS flows according to event status or patient answers.

| Sub-workflow | Trigger condition | Key calendar-module calls |
|--------------|------------------|---------------------------|
| **Confirmation** | A new event is added. | [`#listEvents`](#listEvents), [`#getEvent`](#getevent) → [`#patchEvent`](#patchevent) |
| **Cancellation** | Patient **Declines** appointment confirmation reminder. | [`#patchEvent`](#patchevent) |
| **Reschedule — After Patient cancellation** | Patient cancels inside confirmation thread and wants to reschedule. | [`#listAvailableSlots`](#listavailableslots) → [`#patchEvent`](#patchEvent) → [`#insertEvent`](#insertEvent) |
| **Reschedule — Bump** | Provider cancels the appointment in the calendar with a Bumped status and the patient wants to reschedule (`status == BUMPED`). | [`#listEvents`](#listEvents), [`#getEvent`](#getevent) → [`#patchEvent`](#patchEvent) → [`#listAvailableSlots`](#listAvailableSlots) → [`#insertEvent`](#insertEvent) |
| **Reschedule — No-show** | The provider change the appointment status when the patient no shows to the appointment (`status == NOSHOW`). | [`#listEvents`](#listEvents), [`#getEvent`](#getevent) → [`#patchEvent`](#patchEvent) → [`#listAvailableSlots`](#listavailableslots) → [`#insertEvent`](#insertEvent) |
| **Waitlist Offer** | When a patient cancels, the free slot is offered next patient in the waitlist | [`#getWaitlist`](#getWaitlist), [`#insertEvent`](#insertEvent), [`#deleteWaitlist`](#deleteWaitlist) |


> **Why `#getEvent` and`#patchEvent` matters** — Holly never overwrites existing data blindly; she first **GETs** the latest event information, merges changes, then **PATCHes**—preventing race conditions with clinic staff who may be editing the same record.

---

## Synchronization Processes <!-- ========================================================== -->

Nimblr runs background jobs that keep our local state aligned with each connected EHR.

| Process | Frequency | Purpose | **Key calendar-module methods** |
|---------|-----------|---------|---------------------------------|
| **Login / Refresh Token** | Every 12 h | Renew OAuth/refresh tokens | — |
| **Insert New Appointment** | Once per Booking or Rescheduling conversation with a patient. | Lookup/create patient & book event | [`#searchContacts`](#searchContacts), [`#insertContact`](#insertContact), [`#updateContact`](#updateContact), [`#insertEvent`](#insertEvent) |
| **Handle No-show** | When a NOSHOW appointment status detected | Update notes, status | [`#getEvent`](#getEvent), [`#patchEvent`](#patchEvent), [`#getContact`](#getContact) |
| **Handle Cancellation** | When cancellation detected | Update notes, status | [`#getEvent`](#getEvent), [`#patchEvent`](#patchEvent), [`#getContact`](#getContact) |
| **Confirm Appointment** | For each confirmed event | Mark confirmed | [`#getEvent`](#getEvent), [`#patchEvent`](#patchEvent) |
| **Validate Event Details** | Before updating an  event | Ensure status/date/time/notes current.  | [`#getEvent`](#getEvent) |
| **Lookup Providers, Locations, Appointment Types** | On setup page & per booking | Populate event metadata. When customer logs into Nimblr's UI to set Holly preferences. | [`#listOrganizers`](#listOrganizers), [`#listLocations`](#listLocations), [`#listEventTypes`](#listEventTypes) |
| **Waitlist Offer** | When cancellation frees a slot | Offer slot to next patient | [`#getWaitlist`](#getWaitlist), [`#insertEvent`](#insertEvent), [`#deleteWaitlist`](#deleteWaitlist) |

---

## Methods <!-- ========================================================== -->

Name | Description
--- | ---
[`#listCalendars`](#listCalendars) | Retrieves a list of calendars for this EHR.
[`#listEvents`](#listEvents) | Lists calendar events. This method is crucial for tracking contact events, including those that need confirmation, rescheduling, or have a no-show status. Holly uses this information to manage outbound communication flows effectively.
[`#listEventsByContact`](#listEventsByContact) | Lists calendar events by contact.
[`#getEvent`](#getEvent) | Retrieves an event from a calendar.
[`#insertEvent`](#insertEvent) | Inserts a new event into a calendar.
[`#patchEvent`](#patchEvent) | Modifies an existing event in a calendar.
[`#deleteEvent`](#deleteEvent) | Deletes an event from a calendar.
[`#listAvailableSlots`](#listAvailableSlots) | Lists all available calendar slots in chronological order.
[`#searchContacts`](#searchContacts) | Retrieves contacts matching the criteria.
[`#getContact`](#getContact) | Retrieves a contact.
[`#insertContact`](#insertContact) | Inserts a contact matching the criteria.
[`#updateContact`](#updateContact) | Updates contact information.
[`#listEventTypes`](#listEventTypes) | Lists calendar event types. Minimum attributes needed: `id` and `name`.
[`#listLocations`](#listLocations) | Lists locations. Minimum attributes needed: `id` and `name`.
[`#listOrganizers`](#listOrganizers) | Lists organizers. Minimum attributes needed: `id` and `name`.
[`#getWaitlist`](#getWaitlist) | Retrieves the waitlist for a specific calendar.
[`#deleteWaitlist`](#deleteWaitlist) | Deletes a patient from the waitlist.
[`#uploadInsuranceCard`](#uploadInsuranceCard) | Uploads an insurance card photo for a contact.
[`#getInsuranceCard`](#getInsuranceCard) | Retrieves the insurance card photo for a contact.

### #listCalendars
Retrieves a list of calendars for this EHR.

Listing the calendars is important, as it allows using their definitions to manage events. Therefore, the `externalId` of the calendar is used to retrieve the events.
If needed, events can be obtained using the `externalId` of the organizer or the `externalId` of the location, these can be used if the concept of a calendar does not exist in the EHR.

Returns @Array[[Calendar]](#Calendar)

### #listEvents
Lists calendar events.

Within the events, events for contacts must be perfectly mapped, containing all necessary information to process the communication with the patient. Important information includes the Contact `externalId`, which is used to query patient information, and the event status, which indicates whether the event is canceled, the patient did not show up, or it is confirmed. This information helps initiate Holly's outbound flows. To retrieve events, they must be queried by calendar `externalId` within a specified time period.

Returns @Array[[Event]](#Event)

### #listEventsByContact
Lists calendar events by contact.

This function allows retrieving events for a specific contact within a specified time period.
To retrieve events by contact, they must be queried by patient `externalId` within a specified time period.

Returns @Array[[Event]](#Event)

### #getEvent
Retrieves an event from a calendar.

This method is used to obtain updates for a specific event. It is primarily used before making updates to ensure that no information is lost on the EHR side.
To retrieve an event, the `externalId` of the event must be provided.

Returns @[Event](#Event)

### #insertEvent
Inserts a new event into a calendar.

This method is used to book a new event in the calendar.

Returns @[Event](#Event)

### #patchEvent
Updates the variables of an event (status, comments, etc.).

This method is used to update the status of an event. It is important to update the event status to reflect confirmation, cancellation, or rescheduling following patient's conversation with Holly.

Returns @[Event](#Event)

### #deleteEvent
Deletes or cancels a specific event.

Returns @[Boolean](#Boolean)

### #listAvailableSlots
Lists all available calendar slots in chronological order.
To retrieve available slots, the method must be queried by calendar `externalId` within a specified time period. They can additionally be obtained or filtered by adding the `externalId` of the organizer/event type/location in the query.

Returns @[Slot](#Slot)

### #searchContacts
Retrieves contacts matching the criteria.

Contact searches are performed when the `externalId` of the contact is unknown. The search can be conducted using parameters such as SSN, last name, first name, date of birth, phone, email, or any other parameter that allows for an effective search to obtain a single contact.

Returns @Array[[Contact]](#Contact)

### #getContact
Retrieves a contact.
This method retrieves all the information of a contact using their `externalId`. Unlike the previous method (searchContacts) which is used when the externalId is unknown, this method allows obtaining complete details of the contact, including their name, phone number, and other important parameters necessary to contact them, either by call or SMS.

Returns @[Contact](#Contact)

### #insertContact
Inserts a contact matching the criteria.
This method is used to insert a new contact into the EHR.

Returns @[Contact](#Contact)

### #updateContact
Updates contact information (email, language, gender).

This method updates contact information using their `externalId`. The information is updated when the user enables Holly to update the contact information.

Returns @[Contact](#Contact)

### #listEventTypes
Lists calendar event types.
This method lists the event types available to be provided. It requires a minimum of two attributes: `externalId` and `name`.
This can be used to map the services that the clinic directly provides within the Electronic Health Record (EHR) system.

Returns @Array[[EventType]](#EventType)

### #listLocations
This method lists locations, which can represent physical places such as offices or clinics.
These locations are identified by their minimum required attributes: `externalId` and `name`.
The returned array of locations can help in grouping events when listing events by calendar.

Returns @Array[[Location]](#Location)

### #listOrganizers
This method lists organizers, which can represent healthcare providers or treatment providers related to an event.
These organizers are identified by their minimum required attributes: `externalId` and `name`.
The returned array of organizers can help in associating events when listing events by calendar.

Returns @Array[[Organizer]](#Organizer)

### #getWaitlist (Optional)
This method retrieves the waitlist for a specific calendar.
The waitlist is a list of patients who are waiting for an earliest available slot in the calendar. The waitlist can be used to offer open calendar slots to patients who are waiting for an earlier spot.

The waitlist can be queried by calendar, organizer, location, or event type using the `externalId` of any or all of these parameters. The more refined the search, the better the offer to move the event to the correct patient.

Returns @Array[[Waitlist]](#Waitlist)

### #deleteWaitlist (Optional)
This method deletes a patient from the waitlist.
This method is used to remove a patient from the waitlist after they have been booked for an event.

Returns @[Boolean](#Boolean)

### #uploadInsuranceCard (Optional)
Uploads an insurance card photo for a contact.

This method is used to upload a photo of the insurance card for a specific contact. The photo can be used for verification and record-keeping purposes.

Returns @[Boolean](#Boolean)

### #getInsuranceCard (Optional)
Retrieves the insurance card photo for a contact.

This method is used to retrieve the photo of the insurance card for a specific contact. It helps in verifying the insurance details of the contact.

Returns @[String](#String)

## Objects <!-- ========================================================== -->

#### Calendar
A calendar is a facade of a real calendar (in EHRs, Google Apps, Salesforce, etc.). A calendar has a server implementation that handles the heavy lifting of communication with the real calendar via dependency injection.

name | required | type | description | length
--- | --- | --- | --- | ---
`selfEmail` | true | string | The email/id of the calendar owner. It is not easy to obtain via OAuth. As soon as a new event is created, it is seen and grabbed. | 256
`name` | true | string | Calendar Name. It usually comes from the organizer's name (e.g., Google calls it `summary`). | 128
`externalId` | true | string | Reference to the original calendar ID. | 512
`timeZone` | true | string | Time zone according to the IANA Time Zone Database. | 128
`provider` | false | string | The name of the implementation provider (e.g., Google, Salesforce, SetMore, etc.). | 64
`location` | false | string | Reference to the original location. Sometimes APIs require this ID to be passed back. For example, Google has free-flowing text, Athena has "practice ID". NOTE: This concept is not the same as locations. | 512
`profile` | false | object | A contact associated with the calendar. Stores: name, address, etc. MUST follow Nimblr's contact structure. |
`selfOwned` | false | boolean | Whether the calendar is owned by the associated user or not. Important for EHRs with calendar delegation. Default *true*. |

#### Event
This is a transient model that describes the common schema for events that Nimblr handles. Notation is taken from CalDAV (see https://tools.ietf.org/html/rfc4791).

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original calendar event ID (e.g., Google uses a long UUID). |
`calendar` | true | string | Reference to the original calendar ID. |
`status` | true | string | Event status. It can be 'confirmed', 'tentative', 'cancelled', 'no-show', etc. Look at the table below | 32
`prefix` | false | string | A small prefix with event status (icons that reflect progress). | 16
`summary` | false | string | Event summary shown as the event title. |
`type` | false | string | Type of event. It's calendar provider-specific. It may not exist for some calendars (e.g., Google/Outlook). It can also be event type (e.g., 'First Visit') or 'BLOCK' for blocking slots. |
`location` | false | string | Event location externalId. |
`room` | false | string | Room identifier for the event (only for EHRs that manage "rooms" as calendars). | 32
`color` | false | string | Event color. Note that the event colors used for now are: green, yellow, and red. Each implementation will map the colors accordingly. | 16
`description` | false | string | Full description of the meeting. |
`organizer` | true | string | Email/externalId of the provider (the Owner of the calendar). |
`creator` | true | string | Email/externalId of the creator (if the calendar has been delegated, the creator will be the person who created the event). |
`start` | true | date | DateTime when the event starts in UTC. |
`end` | true | date | DateTime when the event ends in UTC. If the end time does not exist, the duration is required to calculate the end time. |
`allDay` | false | boolean | This is a one (or multiple) day event with a duration of a complete day. Default *false*. |
`deleted` | false | boolean | True if the event has been deleted or cancelled by the organizer. |
`timeZone` | false | string | Time zone according to the IANA Time Zone Database name. If not set, the calendar time zone will be used. |
`contacts` | true | object | Contact object or externalId of the Contact. |
`comments` | false | object | Map from email to comment (only for those Contact that have left comments). |
`overlappable` | false | boolean | An indicator that this event can be overlapped. |
`created` | true | date | DateTime when the event was created. |
`modified` | true | date | DateTime when the event was modified. |
`address` | false | string | Event's address. Allows overriding preferences and calendar address. |
`phone` | false | string | Specific phone number for the event. Allows overriding preferences and calendar phone. |
`url` | false | string | Contains the URL for a virtual event. |

##### Event Status Mapping

The table below outlines the internal status values Holly uses to manage events. This is for reference only and does not limit the possible statuses available in an EHR.
They help Holly manage communication flows and event processing effectively

status | description
--- | ---
NEEDS_ACTION | The Contact has not been contacted.
TENTATIVE | The contact process with the Contact has started.
ACCEPTED | Contact has been contacted and agreed to show up.
BUMPED | Event was cancelled by the organizer.
DECLINED | Contact has declined to attend.
RESCHEDULED | Contact has expressed desire to reschedule.
NOSHOW | Contact did not show up for the event.
PAID | Event has been paid for by the Contact.
UNPAID | Mask that prevents confirmation messages from being sent until an event is paid.
UNKNOWN | To use this status with certain calendars that don't support Contact status or as a default.

#### Slot
name | required | type | description | length
--- | --- | --- | --- | ---
`start` | true | date | DateTime when the slot starts. |
`end` | true | date | DateTime when the slot ends. |
`room` | false | string | Room identifier for the event (only for providers (EHRs) handling rooms). |
`externalId` | true | string | Reference to the original slot ID. |
`location` | false | string | Event location externalId. |

#### Contact
This is a transient model that describes the common schema for contacts that Nimblr handles. Notation is taken from CalDAV (see https://tools.ietf.org/html/rfc4791).

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original contact object (e.g., Google uses a long UUID). |
`name` | true | string | Full name of the contact. | 64
`title` | false | string | Title for the contact (e.g., Mr, Mrs, Count, etc.). | 32
`email` | false | string | Contact email. Default *""*. | 128
`firstName` | true | string | First Name. | 32
`lastName` | true | string | Last Name. | 32
`gender` | false | string | Gender of the contact. | 32
`contactable` | false | boolean | Whether permission to contact this person is granted. This is different from 'reachable' which is whether a message could be sent to the phone number provided. Default *true*. |
`reachable` | false | boolean | Assuming permission to contact is granted, whether this person could be reached. |
`postalCode` | false | string | Postal Code. | 32
`address` | false | string | Address. | 256
`mobile` | true | string | Mobile phone number. | 64
`work` | false | string | Work phone number. | 64
`home` | false | string | Home phone number. | 64
`language` | false | string | Language spoken by the user in two-letter codes (ISO639-1). |
`ssn` | false | string | Social Security Number. | 32
`dateOfBirth` | false | date | Date of Birth. |
`insuranceName` | false | string | Name of the insurance company. | 64
`insuranceNumber` | false | string | Insurance number. | 32
`insuranceMemberId` | false | string | Insurance member ID. | 32
`Pharmacy` | false | string | Pharmacy name. | 64
`EmergencyContactName` | false | string | Emergency contact name. | 64
`EmergencyContactMobile` | false | string | Emergency contact phone number. | 64
`ReferedBy` | false | string | Name of the person who referred the contact. | 64


#### EventType
This is a transient model that describes the common schema for event types that Nimblr handles.

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original event type ID. |
`name` | true | string | Name of the event type. | 64
`duration` | false | integer | Duration of the event type in minutes. Default *0*. Note: If the endpoint to list available slots does not exist, this field becomes mandatory. |

#### Location
This is a transient model that describes the common schema for locations that Nimblr handles.

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original location ID. |
`name` | true | string | Name of the location. | 64
`address` | false | string | Address of the location. | 256
`phone` | false | string | Phone number of the location. | 64
`zipcode` | false | string | Zip code of the location. | 16
`city` | false | string | City of the location. | 64
`state` | false | string | State of the location. | 64
`country` | false | string | Country of the location. | 64
`timezone` | false | string | Time zone according to the IANA Time Zone Database. | 128

#### Organizer
This is a transient model that describes the common schema for organizers that Nimblr handles.

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original organizer ID. |
`name` | true | string | Name of the organizer. | 64
`title` | false | string | Title of the organizer. | 32
`email` | false | string | Email of the organizer. | 128
`phone` | false | string | Phone number of the organizer. | 64
`timezone` | false | string | Time zone according to the IANA Time Zone Database. | 128
`specialty` | false | string | Specialty of the organizer. | 64
`location` | false | string | Event location externalId. |

#### Waitlist
This is a transient model that describes the common schema for waitlists that Nimblr handles.

name | required | type | description | length
--- | --- | --- | --- | ---
`externalId` | true | string | Reference to the original waitlist ID. |
`contacts` | true | object | Contact object or externalId of the Contact. |
`type` | false | string | Type of event. It's calendar provider-specific. It may not exist for some calendars (e.g., Google/Outlook). It can also be event type (e.g., 'First Visit') or 'BLOCK' for blocking slots. |
`location` | false | string | Event location externalId. |
`organizer` | false | string | Organizer externalId. |
`calendar` | true | string | Reference to the original calendar ID. |


## Example <!-- ========================================================== -->

The following example shows how an external event object is mapped to a Nimblr event object. The external object is in JSON format, and the system converts it into an internal object using the specified mapping rules.

#### Request:
```curl
GET /ehr-api/provider/678912345/event/500ABCD001
```

#### Response:
```javascript
{
    profile: "",
    office: 0,
    color: "#FF5733",
    duration: 15,
    event_is_block: false,
    id: "500ABCD001",
    scheduled_time: "2025-03-30T12:00:00-0700",
    doctor: 123456789,
    patient: 987654321,
    clinic_room: 2,
    overlap: false,
    deleted_flag: false,
    notes: "New Patient-First event",
    timeZone : "America/New_York",
    event_created : "2025-02-15T17:23:00-0700",
    event_modified : "2025-02-15T17:23:00-0700",
    creator_id : 123456789,
    service_type : "AB11",
    owner_id : 678912345,
    status: "CONFIRMED"
}
```

#### Object Internal Mapping

Nimblr Property | External Object | Value
--- | --- | ---
`externalId` |  `id` | "500ABCD001"
`prefix` |  ---  |  ---
`summary` | --- | ---
`type` | `service_type` | "ABCD1123"
`location` | --- | ---
`room` |  `clinic_room` | 2
`color` | `color` | "#FF5733"
`description` | --- | ---
`organizer` | `owner_id` | 678912345
`creator` | `creator_id` | 123456789
`start` | `scheduled_time` | "2025-03-30T12:00:00-0700"
`end` | `scheduled_time` + `duration` |  "2025-03-30T12:00:15-0700"
`allDay` | --- | false
`deleted` | `deleted_flag` | false
`timeZone` | `timeZone` | "America/New_York"
`contacts` | `patient` | 987654321
`comments` | `notes` | "New Patient-First event"
`overlappable` | `overlap` | false
`created` | `event_created` | "2025-02-15T17:23:00-0700"
`modified` | `event_modified` | "2025-02-15T17:23:00-0700"
`address` | --- | ---
`phone` |  --- | ---
`url` |  --- | ---
`status` | `status` => `internal status` | "ACCEPTED"

## Glossary <!-- ========================================================== -->

- **Facade**: A design pattern that provides a simplified interface to a complex subsystem, such as an external calendar system. It hides the complexity of the underlying system and provides a unified interface for interaction.
- **Dependency Injection**: A design pattern in which an object receives other objects that it depends on.
- **CalDAV**: A protocol used to access calendar data on a server.
- **IANA Time Zone Database**: A database that provides time zone information.
- **OAuth**: An open standard for access delegation, commonly used for token-based authentication.
- **UUID**: Universally Unique Identifier, a 128-bit number used to uniquely identify information.
- **JSON**: JavaScript Object Notation, a lightweight data-interchange format.
- **XML**: Extensible Markup Language, a markup language that defines a set of rules for encoding documents.
- **EHR**: Electronic Health Record, a digital version of a patient’s paper chart.
- **API**: Application Programming Interface, a set of rules that allows different software entities to communicate with each other.
- **Slot**: A specific time period available for scheduling events.
- **Waitlist**: A list of patients waiting for an available slot for an event.
- **organizer**: A healthcare or treatment organizer/provider related to an event.
- **EventType**: A specific type of event that can be scheduled in a calendar.
- **Location**: A physical place such as an office or clinic where events can occur.
- **Contact**: An individual whose information is stored and managed within the system.
- **Event**: A scheduled occurrence in a calendar, such as an appointment or meeting.
- **Calendar**: A system for organizing and managing events, often integrated with other systems like EHRs or Google Calendar.
- **ExternalId**: A unique identifier used to reference an object (such as a calendar, event, or contact) in an external system. This ID ensures synchronization between Nimblr and external platforms like EHRs or Google Calendar.
- **TimeZone**: A region-specific time standard that follows the IANA Time Zone Database. It is used to ensure that events and reminders are displayed in the correct local time for users.
- **Event status**: A field that indicates the current state of an event, such as 'confirmed', 'cancelled', 'tentative', or 'no-show'. This status helps determine the appropriate communication flow with the patient.
