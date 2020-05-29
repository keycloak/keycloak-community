# Feedback wanted on Events use case

In the current console, users can configure and view events of one specific realm including event listeners, login events and admin events. There are two kinds of events that can be saved and displayed.
* Login events: Every single event that happens to a user can be recorded and viewed.
* Admin events: Any action an admin performs within the admin console can be recorded for auditing purposes.

But there are some use cases related to events that need to be discussed. In this document, there are two main parts: personas and use cases, and listener types.


### Personas and use cases
According to the previous interview and our discussion, we listed some personas and use cases. If you have any use cases or proposals please feel free to leave comments.
* User support staff - “I am in charge of end user management and support. I mainly monitor users' events and troubleshoot login problems.” I want to:
  * View login data:
     * how many logins and login errors
     * the percentage of failure.（Adding a dashboard to show these information may be good.）
     * how many requests are coming to Keycloak
     * metrics for logins per minute and number of active sessions(actual user traffic)
  * Check some information for troubleshooting.
  * Search the event log for specific user’s events by user name, instead of user ID.
  * Easily see what the user did. (In the Users menu, there could be another tab with events where I could easily see what the user did.)
  * Monitor response times.


* Client Developer - “I manage the clients and need to change some configurations of clients or realm, eg, client settings, roles, authenticators, etc.” I want to:
  * Easily see what I've done after settings have been configured.
  * Check out what has changed and who did that.
  * View and filter events by authSessionId.  (Adding authSessionId to events and showing it/filtering by it.)


* Realm administrator - “I need to monitor the health of the system and configure some realm settings.”  I want to:  
  * Select the event listener for this realm and configure what types of events are sent to the different event listeners.
  * Decide what kinds of events can be recorded.
  * See what has updated and who has done that in the realm.
  * Track security vulnerabilities and get pushed into other RH products like Satellite.


### Event listener types
In the current console, there are two predefined listeners: Jboss-logging listener and email listener. Both of them should be configurable listeners.

There will be more than one place to send and save events. In the future, there may be some other listener types such as user listener, admin listener, registration web hooks and so on. Event listeners will not only be added, but also have configuration options when added (similar to user federation or identity providers). There can also be multiple instances of the same event listener (with different configuration). We have listed the current listeners, and if you have any updates or proposals please feel free to leave comments.


* Jboss-logging listener: It writes to a log file whenever an error event occurs and is enabled by default.

* Email listener: It sends an email to the user’s account when an event occurs. It can configure what types of events are sent to email listeners.
