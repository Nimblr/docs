# Integrations
==============================================================================

A facade for a *real* calendar

The calendar module is really just a facade on top of real Calendar APIs. As such it will
manipulate events that do not live inside Nimblr&#39;s realm but some other cloud.

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
[`#deleteContact`](#deleteContact) | Delete contact
[`#getContact`](#getContact) | Get contact
[`#insertContact`](#insertContact) | Insert contact that match the criteria
[`#acquireAuthorization`](#acquireAuthorization) | Acquires authorization via the user knowing the provider name and the credentials for it. Once the authorization

### Authorization

**
