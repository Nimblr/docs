# Integrations
==============================================================================

A facade for a *real* calendar

The calendar module is really just a facade simulating a real Calendar.  As such it will manipulate events that do not live anywhere inside Nimblr&#39;s.

It&#39;s necessary that the real calendar is consulted frequently to have any update in the calendar (new appointments, cancellations, etc.) in the shortest possible time.

Nimblr’s API is able to map any “external” object into a Nimblr’s object. An external object can be in JSON or XML format.


----
## Index

[`Methods`](#Methods-)

[`Objects`](#Objects-)

[`Method Request Example`](#Example-)

## Methods <!-- ========================================================== -->

Name | Description
--- | ---
[`#listCalendars`](#listCalendars) | Retrieves a list of the calendars for this provider.
[`#listEvents`](#listEvents) | List calendar events. `showCancelled` may have different behavior depending on the implementation use cases. Calendars methods will adopt the most restrictive interpretation (minimum number of events).
[`#getEvent`](#getEvent) | Get event from a calendar.
[`#insertEvent`](#insertEvent) | Insert a new event into a calendar
[`#patchEvent`](#patchEvent) | Modifies an existing event in a calendar
[`#deleteEvent`](#deleteEvent) | Deletes event from calendar
[`#listAvailableSlots`](#listAvailableSlots) | List all available calendar slots, done only once. The returned slots will always be in chronological order.
[`#searchContacts`](#searchContacts) | Get contacts with the matched criteria
[`#getContact`](#getContact) | Get contact
[`#insertContact`](#insertContact) | Insert contact with the matched criteria
[`#listEventTypes`](#listEventTypes) | List calendar event types. Minimim attritubes needed: `id` and `name`
[`#listLocations`](#listLocations) | List locations. Minimim attritubes needed: `id` and `name`

### #listCalendars
Retrieves a list of the calendars for this provider.

Returns @Array[[Calendar]](#Calendar)

### #listEvents
List calendar events. `showCancelled` may have a different behavior depending on the implementation (every EHR manages cancelled/deleted events in different ways). `showBumped` handles the EHR's which don't use `showCancelled` method. Calendars methods will adopt the most restrictive interpretation (minimum number of events).

Note: if there is an error, all calendars MUST give status code via `statusCode`.  It's mandatory that the calendar implemantation have the right status. `statusCode` it&#39;s a standard name in Node

Returns @Array[[Event]](#Event)

### #getEvent

Get event from calendar.

Returns @[Event](#Event)

### #insertEvent

Insert a new event into the calendar.

Returns @[Event](#Event)

### #patchEvent

Update the variables of an event (status, comments , etc).

Returns @[Event](#Event)

### #deleteEvent

Delete or cancel a specific event.

Returns @[Event](#Event)

### #listAvailableSlots

List all available calendar's slots. The returned slots should always be in chronological order.

Returns @[Slot](#Slot)

### #searchContacts
Get contacts that match the criteria

Returns @Array[[Contact]](#Contact)

### #getContact
Get contact

Returns @[Contact](#Contact)

### #insertContact
Insert contact that match the criteria

Returns @[Contact](#Contact)

### #listEventTypes
List calendar event types. Minimum attributes required: `id` and `name`

Returns @Array[EventType]

### #listLocations

List locations. Minimum attributes required: `id` and `name`

Returns @Array[Location]

## Objects <!-- ========================================================== -->

#### Calendar

A calendar is a facade of a real calendar (in EHR's, Google Apps, SalesForce, etc). A calendar has a (server) implementation that handles the heavy lifting of communication with the real calendar via dependency injection.

name | type | description | length
--- | --- | --- | ---
`selfEmail` | string | The email/id from the calendar owner. It is not easy to obtain it via OAuth. As soon as a new appt is created, we see and grab it. | 256
`name` | string | Calendar Name. It normally comes from the provide's name (i.e. Google calls it `summary`) | 128
`externalId` | string | Reference to the original calendar ID | 512
`timeZone` | string | Time zone according to the IANA Time Zone Database. | 128
`provider` | string | The name of the implementation provider (i.e. Google, Salesforce, SetMore, etc) | 64
`location` | string | Reference to the original location. Sometimes APIs require that this ID passed back, in order to do that, we reserve a field for it. For example, Google has free flowing text, Athena has "practice ID". NOTE: This concept is not the same as locations | 512
`profile` | object | A contact associated with the calendar. We store: name, adresss, etc. MUST follow Nimblr's contact structure.
`selfOwned` | boolean | Whether the calendar is owned by the associated user or not. Important for EHR's with calendar delegation.


#### Event

This is a transient model that describes the common schema for events that Nimblr handles. Notation is taken from CalDAV (see https://tools.ietf.org/html/rfc4791)

name	|	type	|	description	|	length
---	|	---	|	---	|	---
`externalId`| string	|	Reference to the original calendar event ID (For example, Google uses a longish UUID)	|	
`prefix`	| string	|	A small prefix with event status (icons that reflect progress)	|	16
`summary`	| string	|	Event summary shown as in the event title	|	
`type`  	| string	|	Type of event. It's calendar provider specific. It may not exist for some calendars (i.e. Google/Outlook). It can also be appt type (i.e. 'First Visit') or 'BLOCK' for blocking slots	|	
`location`	|	string	|	Event location	|
`room`	|	string	|	Room identifier for the event (only for EHRs that manage "rooms" as calendars)	|	32
`color`	|	string	|	Event color. Note that the event colors used for now are: green, yellow and red. Each implementation will map the colors accordingly	|	16
`description`	|	string	| Full description of the meeting	|	
`organizer`	|	string	|	Email/ID of the organizer (it's the Owner of the calendar)	|	
`creator`	|	string	|	Email/ID of the creator (if the calendar has been delegated, the creator will be the person who created the event)	|	
`start`	|	date	|	DateTime when the appt starts in UTC	|	
`end`	|	date	|	DateTime when the appt ends in UTC	|	
`allDay`	|	boolean	|	This is a one (or multiple) days with time duration of a complete day	|	
`deleted`	|	boolean	|	True if the event has been deleted or cancelled by the organizer	|	
`timeZone`	|	string	|	Time zone according to the IANA Time Zone Database name. If it is not set, the calendar time zone will be used |
`contacts`	|	object	|	Map from email to contact (if embedded in the event itself)	|	
`comments`	|	object	|	Map from email to comment (only for those attendees that have left comments)	|	
`overlappable`	|	boolean	|	An indicator that this event can be overlapped.	|	
`created`	|	date	|	DateTime when the event was created	|	
`modified`	|	date	|	DateTime when the event was modified	|	
`address`	|	string	|	 Event's address. Allows to override preferences and calendar address	|	
`phone`	|	string	|	Specific phone of the event. Allows to override preferences and calendar phone	|	
`url`	|	string	|	Contains the url for a telemedicine appt|

#### Slot
name  |	type | description | length
---	  |	---	 | ---	       |	---
`start` | date | DateTime when the slot starts |
`end`   | date | DateTime when the slot ends |
`room`  | String | Room identifier for the event (only for providers handling rooms) |

#### Contact

This is a transient model that describes the common schema for contacts that Nimblr handles. Notation is taken from CalDAV (see https://tools.ietf.org/html/rfc4791)

name	|	type	|	description	|	length
---	|	---	|	---	|	---
`externalId`	|	string	|	Reference to the original contact object (Google for examples uses a longish UUID)	|	
`name`	|	string	|	Full name for the contact	|	64
`title`	|	string	|	Title for the contact (i.e. Mr, Mrs, Count, etc)	|	32
`email`	|	string	|	Contact email	|	128
`firstName`	|	string	|	First Name	|	32
`lastName`	|	string	|	Last Name	|	32
`contactable`	|	boolean	|	Whether we have permission to contact this person. This is different from 'reachable' which is whether we could send a message to the phone number we have	|	
`reachable`	|	boolean	|	Going in the premise we have permission to contact, were we able to reach this person?	|	
`postalCode`	|	string	|	Postal Code	|	32
`mobile`	|	string	|	Mobile phone number	|	64
`work`	|	string	|	Work phone number	|	64
`home`	|	string	|	Home phone number	|	64
`language`	|	string	|	Language spoken by user in two letter codes (ISO639-1)	|	
`created`	|	date	|	DateTime when the contact was created	|	
`modified`	|	date	|	DateTime when the contact was created	|	

## Example <!-- ========================================================== -->

The following example shows a request from Nimblr’s API  and response external API using a getEvent method. The response can be in XML or JSON format.
The response is processed by mapping the external object and convert it into an internal object.

#### Request:
```javascript
{
  id : "500ABCD001"
}
```

#### Response:
```javascript
{
    profile: "",
    office: 0,
    color: "#FF5733",
    duration: 15,
    appt_is_block: false,
    id: "500ABCD001",
    scheduled_time: "2020-05-30T12:00:00-0700",
    doctor: 123456789,
    status: "pending",
    patient: 987654321,
    clinic_room: 2,
    overlap: false,
    deleted_flag: false,
    notes: "New Patient-First Appt",
    timeZone : "America/New_York",
    appt_created : "2020-04-15T17:23:00-0700",
    appt_modified : "2020-04-15T17:23:00-0700",
    creator_id : 123456789,
    service_type : "AB11",
    owner_id : 678912345
}
```


#### Object Mapping

Nimblr Property | External Object | Value
--- | --- | ---
`externalId` |  `id` | "500xABCD001"
`prefix` |  ---  |  ---
`summary` | --- | ---
`type` | `service_type` | "ABCD1123"
`location` | --- | ---
`room` |  `clinic_room` | 2
`color` | `color` | "#FF5733"
`description` | --- | ---
`organizer` | `owner_id` | 678912345
`creator` | `creator_id` | 123456789
`start` | `scheduled_time` | "2020-05-30T12:00:00-0700"
`end` | `scheduled_time` + `duration` |  "2020-05-30T12:00:15-0700"
`allDay` | --- | false
`deleted` | `deleted_flag` | false
`timeZone` | `timeZone` | "America/New_York"
`contacts` | `patient` | 987654321
`comments` | `notes` | "New Patient-First Appt"
`overlappable` | `overlap` | false
`created` | `appt_created` | "2020-04-15T17:23:00-0700"
`modified` | `appt_modified` | "2020-04-15T17:23:00-0700"
`address` | --- | ---
`phone` |  --- | ---
`url` |  --- | ---


