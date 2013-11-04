For App Developers
==================

An app developer wishes to get presence information, the following can be addd to the app manifest:

.. code::

    {
        "permissions": {
            "push": {"description": "To receive notifications about the newest phone releases"},
            "presence": {"description": "To broadcast presence"}
        },
        "messages": [
            {"push": "/view_to_launch.html"},
            {"push-register": "/view_to_launch.html"}
        ]
    }

Note that the existing bits for SimplePush are retained, as Presence merely add's broadcast of the user's Presence
to the app.

The application developer must register their application separately with Mozilla Presence and set a "Webhook URL"
that will be called with presence updates for users that look like the following:

.. code::

    POST /some/webhook/callback
    
    {[
        ["UID2949293", "away"],
        ["UID4823888", "online"],
        ["UID482838", "offline"],
        ["UID4828382", "online"]
    ]}

When the user grants access, your application will get a unique ID that you should associate with the user record
your application stores.

A doorhanger will be needed in applications to prompt the user to grant an application Presence notification data
should the user not grant it initially when installing the app (and for websites that don't necessarilly install
as 'apps').

Setup of a doorhanger will require the app developer to register their application with Mozilla Services and get
an application token. This application token can be used with Mozilla TokenServer to register access with the
Mozilla Presence server. The app developer may then enter the "Webhook URL" in their Settings page for Mozilla
Presence.

When a user signs-in to your application/website and you wish to ask for Presence notifications, you may redirect
the user to Mozilla Services asking for "Presence" access. The user will be prompted to grant access or deny and
redirected back to your application. If the user granted access, the redirect will include an encoded UID that you
should store and associate with the user as this UID will be the one sent in the Webhook calls.

App developers should account for the possibility that 'updates' may get lost, and automatically mark a user as
offline if no status is sent in an hour.

For Device Users
================

A separate Settings page is available, that looks similar to the iOS Notifications page, showing a list of apps and
websites that have been granted Presence access. The page also lets one set a "Presence Policy" that determines
what liveliness is communicated, for example:

```Presence Policy```

Show me online when I've used this device in the past:
    5 - 10 - 15 - 20   (minutes)

Show me away when I haven't used this device in at least:
    5 - 10 - 15 - 20   (minutes)

Show me offline when I haven't used this device in at least:
    10 - 25 - 35       (minutes)


--------------

Perhaps these will be sliders, UX is not determined this is only a possible example of how the user determines
what type of 'livliness' data is sent based on whether they're using the device or not, and how recently.

A list of toggles for each app the user has granted access to will be here, so that a user may turn off Presence
for individual apps temporailly without fully revoking possible future Presence broadcasts to the app/website.

Mozilla Presence
================

This service is hooked onto SimplePush, such that when a client ping comes in that includes presence info (either O,
A, or U to indicate Online/Away/Unavailable based on the devices Presence policy dictated by the devices user
and its idel time), then SimplePush will broadcast that to Presence somehow, that client id XXYYY is now O/A/U or
whatever. Presence then updates its record and relays the status update to the services the user has authorized.

The service stores records for every user that has Presence 'activated'. The device is tied to the users FxA such
that if the device is lost, when they get a new one they will still identify the same. If/When a client needs to
re-register for SimplePush, the SimplePush client-id is sent to Presence to tie it to the existing FxA record.

Presence stores a set of data (rough schema):

Client-Mappings:
    Client-ID   | Corresponds to the client-id that is used for SimplePush
    FxA-ID      | The FxA of the presence user

Presence-Authentications
    FxA         | The FxA of the presence user
    Service-ID  | Service ID that the app developer registered for this service
    UUID        | A unique ID for this user on this service, to avoid leaking the FxA ID
    Status      | Boolean flag on whether the user currently wants presence broadcast to this service

.. note::
    
    When the user changes the toggle on their device to turn on/off Presence, their device must be able to
    contact our server so that we can update their Presence choice in the Presence-Authentications table.
    This is because Presence is broadcast in the background by the SimplePush UA, and we don't want to
    have to include a list of all the 'apps' that should get the presence update in every ping request.
    
When Presence gets a status update from simplepush, it looks up the client-id in the Client-Mappings, then uses that
to lookup all the services in Presence-Authentications that the user wants notified of this change. It filters out
services that the user has currently chosen not to recieve presence. The remaining services have their webhook URL's
called with the UUID and presence change so that the services can decide what they want to do.

To avoid a call for every single status change, the Presence service may batch updates to the various services so
that the webhook URL for each service is only called every 30+ seconds with all the updates intended for it (in the
list POST format explained above).

Usage Scenarios
===============

Facebook
--------

Jeff wants to appear online on Facebook (he's already determined on Facebook who can see him, etc).

He's already installed the Facebook app on his FFOS phone, he goes to the Settings and touch, "Authorize Presence",
his screen loads a doorhanger (provided by Mozilla Presence) asking if he wants to authorize the app.

Jeff clicks "Yes" on the doorhanger page, and the Facebook app waits while Facebook recieves the redirect, stores
his UID, and closes the doorhanger (and likely needs to do something else to register this with the device so that
it will appear in the Presence Settings page).

The next day, Jeff wakes up, and goes to check his e-mail on his phone. Upon seeing his idle drop, the phone's
simple-push client (which is always running) includes an 'O' in its next PING to simplepush to indicate the user
is now online. Mozilla Presence gets notified of this and acts on it to batch the status change to Facebook. The
batch of updates goes to Facebook including Jeff's, so Facebook knows that Jeff is now online, and updates its
database indicating this.

Jeff's friend Marsha goes to Facebook to see if any of her friends are online. Marsha sees that Jeff is now online
and sends a chat request. Facebook uses SimplePush to notify Jeff of the chat request. Jeff sees the chat request
and switches over to his Facebook app to talk to Marsha.

When Jeff is using his phone, Facebook recieves a webhook call indicating he is online
