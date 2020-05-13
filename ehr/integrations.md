# Integrations
==============================================================================

A facade for a *real* calendar

The calendar module is really just a facade on top of real Calendar APIs. As such it will
manipulate events that do not live inside Nimblr&#39;s realm but some other cloud.

----
## Index

[`Methods`](#Methods-)

[`Objects`](#Objects-)

## Methods <!-- ========================================================== -->

Name | Description
--- | ---
[`#listCalendars`](#listCalendars) | Retrieves a list of the calendars for this provider.
[`#listEvents`](#listEvents) | List calendar events. `showCancelled` may have a different behavior depending on the implementation cases they may be redundant and the implementation will adopt the most restrictive interpretation (meaning the one returning the least number of events).
[`#getEvent`](#getEvent) | Get event from calendar.
[`#insertEvent`](#insertEvent) | Insert event into calendar
[`#patchEvent`](#patchEvent) | Patches calendar event
[`#deleteEvent`](#deleteEvent) | Deletes event from calendar
[`#listAvailableSlots`](#listAvailableSlots) | List all available calendar slots without retry. The returned slots should always be in chronological order.
[`#searchContacts`](#searchContacts) | Get contacts that match the criteria
[`#getContact`](#getContact) | Get contact
[`#insertContact`](#insertContact) | Insert contact that match the criteria
[`#listEventTypes`](#listEventTypes) | List calendar event types. Each type will be different, but at a minimum they have two attributes: `id` and `name`
[`#listLocations`](#listLocations) | List locations. Each type will be different, but at a minimum they have two attributes: `id` and `name`

### #listCalendars
Retrieves a list of the calendars for this provider.

Returns @Array[[Calendar]](#Calendar)

### #listEvents
List calendar events. `showCancelled` may have a different behavior depending on the implementation
as every provider managed cancelled/deleted events in a different way. That is why we add `showBumped`. In some
cases they may be redundant and the implementation will adopt the most restrictive interpretation (meaning the one
returning the least number of events).

Note that if there is an error, all calendars should give status code via `statusCode`. If natively they do it
some other way, then the calendar implementation must make sure that `statusCode` has the right status. We
standarize around this name because it&#39;s the one Node uses.

Returns @Array[[Event]](#Event)

### #getEvent

Get event from calendar.

Returns @[Event](#Event)

### #insertEvent

Insert a new event into calendar.

Returns @[Event](#Event)

### #patchEvent

Update the details of the event (status, comments , etc).

Returns @[Event](#Event)

### #deleteEvent

Delete or cancel a specific event.

Returns @[Event](#Event)

### #listAvailableSlots

List all available calendar slots without retry. The returned slots should always be in chronological order.

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
List calendar event types. Each type will be different, but at a minimum they have two attributes: `id` and `name`

Returns @Array[EventType]

### #listLocations

List locations. Each type will be different, but at a minimum they have two attributes: `id` and `name`

Returns @Array[Location]

## Objects <!-- ========================================================== -->

#### Calendar

A calendar is a facade to a real calendar (in Google Apps, SalesForce, SetMore, etc). A calendar has a (server) implementation that handles the heavy lifting of communicating with the real calendar via dependency injection.

name | type | description | length
--- | --- | --- | ---
`selfEmail` | string | The email/id from the calendar owner. It is not easy to obtain via OAuth, so as soon as we see an event that has it, we grab it. | 256
`name` | string | Calendar Name. It normally comes from the provide's name (Google calls it `summary`) | 128
`externalId` | string | Reference to the original calendar ID | 512
`timeZone` | string | Time zone according to the IANA Time Zone Database name. | 128
`provider` | string | The name of the implementation provider (i.e. Google, Salesforce, SetMore, etc) | 64
`location` | string | Reference to the original location. Sometimes APIs require that this ID be passed back so we reserve a field for it. Google for example has free flowing text, Athena has the practice ID. . NOTE that this is not necessarly the same concept of location for events | 512
`profile` | object | A contact associated with the calendar to store name, adresss, etc. Should follow Nimblr's contact structure.
`selfOwned` | boolean | Whether the calendar is owned by the associated user or not. Important for system with calendar delegation.


#### Event

This is a transient model that describes the common schema for events that Nimblr handles. Notation is taken from CalDAV (see https://tools.ietf.org/html/rfc4791)

name	|	type	|	description	|	length
---	|	---	|	---	|	---
`externalId`| string	|	Reference to the original calendar event ID (Google for examples uses a longish UUID)	|	
`prefix`	| string	|	A small prefix that tells the status of the event (icons that reflect progress)	|	16
`summary`	| string	|	Event summary shown as the event title	|	
`type`  	| string	|	Type of event. It's calendar provider specific, and it may not exist for some calendars (i.e. Google/Outlook). It may be the type of appointment (i.e. 'First Visit') or 'BLOCK' for blocking events	|	
`location`	|	string	|	Event location	|
`room`	|	string	|	Room identifier for the event (only for providers that have rooms)	|	32
`color`	|	string	|	Event color. Note that the event colors used for now are: green, yellow and red. Each implementation will translate accordingly	|	16
`description`	|	string	|	Longer description of the meeting	|	
`organizer`	|	string	|	Email/ID of the organizer (it's the owner of the calendar)	|	
`creator`	|	string	|	Email/ID of the creator (if the calendar has been delegated, then this will be the person that created the event)	|	
`start`	|	date	|	DateTime when the meeting starts in UTC	|	
`end`	|	date	|	DateTime when the meeting ends in UTC	|	
`allDay`	|	boolean	|	This is a one (or multiple) all day event	|	
`deleted`	|	boolean	|	True if the event has been deleted or cancelled by the organizer	|	
`timeZone`	|	string	|	Time zone according to the IANA Time Zone Database name. If it is not present, then the calendar time zone |
`contacts`	|	object	|	A map from email to contact (if embedded in the event itself)	|	
`comments`	|	object	|	A map from email to comment (only for those attendees that have left comments)	|	
`overlappable`	|	boolean	|	An indicator that this event can be overlapped with, so it should not block other events from being inserted	|	
`created`	|	date	|	DateTime when the event was created	|	
`modified`	|	date	|	DateTime when the event was created	|	
`address`	|	string	|	Specific address of the event. Allows to override preferences and calendar address	|	
`phone`	|	string	|	Specific phone of the event. Allows to override preferences and calendar phone	|	
`url`	|	string	|	Contains the url for the video call	|

#### Slot
name  |	type | description | length
---	  |	---	 | ---	       |	---
`start` | date | DateTime when the slot starts |
`end`   | date | DateTime when the slot ends |
`room`  | String | Room identifier for the event (only for providers that have rooms) |

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