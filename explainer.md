# Local Presentation API

## Background

The Presentation API renders Web content on another display by creating a
distinct profile (storage area) and navigating to a new document there.  This
profile is completely separate from the profile that is used to start the
presentation.  This allows the presentated document to be rendered on the
display itself given its URL (aka 2-UA mode) without creating privacy issues and
inconsistencies with 1-UA mode.

2-UA mode is a good fit for media centric applications.  Media players (like a
fullscreen video element or a photo slideshow) typically include resources from
a known set of origins, isn't highly interactive (i.e., responsive to mouse or
touch events), and authentication information can be passed via messaging.
These use cases match what we have seen adopted by the Presentation API thus
far.

## Use cases for complex content

Complex, interactive content assembled from many components and origins creates
some difficulties when a site attempts use the Presentation API.

The canonical example driving this is a Web slide show.  For simplicity I'll use
_controlling page_ for the page that started the slide show and _presented page_
for the page with the slides.

### Third party embeds

Slides that embed private third party resources that require specific cookies or
profile state can't easily be presented.
  
> Example: A slide embeds a private YouTube video that requires that the browser
> be logged into YouTube.

### Offline
  
Presentating slides that are stored offline is difficult.  There is no URL to
navigate to, and all of the offline storage has to be serialized and sent to the
presented page.

> Example: A salesperson stores his slideshow offline and wants to present it on a
> wired display without connecting to WiFi.

### WSIWYG

Editing the slides while presenting requires significant changes to the
application.  The entire data model (stored as Javascript objects and/or Web
components) has to be synchronized between the controlling page and the
presented page using messages.

> Example: The user is editing the slide show content and immediately wants to
> see what it looks like in presentation mode.

### Interaction (touch/gesture)

It is difficult to interact directly with the presented content in response to
mouse or touch on the controlling page.  The touch or mouse event has to be
captured, serialized, and coordinates translated to find the correct element on
the presented page.  The presented component then has to be updated to respond
to messages in addition to native events.

>  Example: On the controlling page, the salesperson uses a touchpad to
>  highlight an element of the slide with popup content.  They want the same
>  popup to appear on the presented page.

## Local Presentation API

Some of these issues could be addressed through restructuring of the web
application in combination with libraries that convert operations and events to
messages and vice versa.  It is worth exploring this approach further for
applications that are willing to invest the effort to adopt these libraries.

However, enabling a different presentation mode could significantly lower the
bar for adoption by existing web applications that fit in this category.

Requirements:
  * Allow the page to detect availability of a remote display capable of
    mirroring a window.
  * Allow direct script access between the controlling and presented page.
  * Allow sharing of local storage (including cookies and offline storage)
    between pages.
  * Make it simple to translate input events from the controlling page to the
    presentation page.
    
Not required:
  * Connection from other pages/devices
  * Reconnection after navigation

Most of these needs can be met by presentation of a same-origin window created
by (and accessible to) the controlling page.  The new window would be presented
on the remote display through wired or wireless mirroring (1-UA mode).  Input
event translation needs more investigation.

## Sample code (Presentation API based)

```javascript
<!-- Controlling page -->
let presentationWindow;
const request = new PresentationRequest('about:blank');
request.start().then(connection => {
  <!-- Window object is only exposed for 'about:blank'. -->
  presentationWindow = connection.window;
});
<!--
  presentationWindow is mirrored to the display.  The controlling page
  has access to presentationWindow.
-->
```

## Sample code (window.open based)

```javascript
<!-- Controlling page -->

<!-- Tracks availability of a presentation display.
     We could use PresentationAvailability for this too. -->
let remoteDisplayAvailable = false;
window.remote.watchAvailability(availability => {
  remoteDisplayAvailable = availability;
});

<!-- UI to start presentation when remoteDisplayAvailable == true. -->
let presentationWindow;
button.onclick = _ => {
  presentationWindow = window.open('about:blank', 'My_Slideshow', 'remote=true');
};
<!-- 
  User is asked to select a remote display and presentationWindow is
  mirrored to that display.  The controlling page has access
  to presentationWindow.
-->
```

(Several API solutions are possible and these are only a starting point for
discussion.)

## Privacy/Security

We need to ensure same-origin policy is not violated by capture and streaming of
the presentated window to a third party display or device.  Remote displays
should be trusted, similar to the displays used for features like tab mirroring
or desktop screenshare.

Even though the presented window is primarily being viewed on a remote display,
the controlling user agent should make the user aware that presentation is going
on and be able to (at a minimum) view the origin that opened the presented
window.
## Explainer

### Examples
