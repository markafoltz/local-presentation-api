# Local Presentation API

## Background

The Presentation API allows a page to request that a URL be rendered on another
display. This can be implemented in one of two ways, called 2-UA and 1-UA modes.

In 2-UA mode, the URL is sent to a distinct user agent and rendered on a
completely separate device.

In 1-UA mode, the URL is rendered in tab in the same user agent, and the tab
contents mirrored to a remote display, or shown on a secondary attached
display. (Chrome has implemented both 2-UA and 1-UA modes.)

In either case, the user agent rendering the presentation must create a distinct
profile (storage area) for the presentation.  The profile is discarded when the
presentation is closed.  This minimizes the developer-exposed differences
between 1-UA and 2-UA modes, and improves privacy for users of shared displays.

While 2-UA mode has worked well for media-centric applications (video and audio
playback), productivity applications prefer 1-UA to target a broader range of
content and screens, including cloud-connected screens that only support 1-UA
mode.

## Use cases

Productivity sites that use complex, interactive content assembled from many
components and origins have found it difficult or impossible to use the
Presentation API in 1-UA mode for reasons listed below.

The canonical example driving this is a Web slide show.  For simplicity I'll use
_controlling page_ for the page that controls the slide show and _presented
page_ for the page showing the slides on the target display.

### Third party embeds

The presented page cannot embed private third party resources that require specific
cookies or profile state, since it can't set cookies on the third party origin.
  
> Example: A slide embeds a private YouTube video that requires that the browser
> be logged into YouTube.

### Offline
  
The presented does not have access to offline storage, and the controlling page
cannot easily present content that is offline.  There is no URL to navigate to,
and all of the offline storage has to be serialized and sent to the presented
page.

> Example: A salesperson stores his slideshow offline and wants to present it on a
> wired display without connecting to WiFi.

### WSIWYG

Editing the content of the presented page while presenting requires significant
changes to the application.  The entire data model (stored as Javascript objects
and/or Web components) has to be synchronized between the controlling page and
the presented page using messages.

> Example: The user is editing the slide show content and immediately wants to
> see the results while presenting. 

### Interaction (touch/gesture)

It is difficult to interact directly with the presented page in response to
mouse or touch on the controlling page.  The touch or mouse event has to be
captured, serialized, and coordinates translated to find the correct element on
the presented page.  The presented component then has to be updated to respond
to these synthetic events.

>  Example: On the controlling page, the salesperson uses a touchpad to
>  highlight an element of the slide with popup content.  They want the same
>  popup to appear on the presented page.

## Local Presentation API

Some of these issues could be addressed through restructuring of the Web
application around a multi-document and multi-media syncronization library
like [MediaScape](http://www.mediascapeproject.eu/).

However, the majority of content was not written with multi-screen in mind, and
the use case is not prevalent enough to justify a full rewrite.  For them,
enabling a different presentation mode could significantly lower the bar for
adoption by adapting web applications that already use popup windows to render
alternate views.

Requirements:
  * Allow the page to detect availability of a remote display capable of
    mirroring a window (tab).
  * Allow direct script access between the controlling and presented page.
  * Allow sharing of local storage (including cookies and offline storage)
    between pages.
  * Make it simple to translate input events from the controlling page to the
    presentation page.
    
Not required:
  * Connection from other pages/devices
  * Reconnection after the initiating page closes or navigates.

Most of these needs can be met by presentation of a same-origin window created
by (and accessible to) the controlling page.  The new window would be presented
on the remote display through wired or wireless mirroring (1-UA mode).  Input
event translation needs more investigation.

## API shape and rationale

The basic idea is that a page that requests a presentation with a special URL,
`about:blank`, will cause that the user agent create a new blank window in the
same browser profile and render that window on a display of the user's choice.

This can be thought of as calling `window.open('about:blank')`, where the user
gets to choose where the popup window goes.  However, the page will not get to
control the appearance or size of the new window - the only guarantee is that it
will be rendered on the target screen (typically taking up the entire screen).

In the current usage of the Presentation API, an ID is returned that allows the
controlling page to reconnect to a running presentation.  In local presentation
mode, no ID is returned and reconnection is not possible.  This mirrors the
current relationship between a window and its opener (i.e., the a window's
opener cannot be changed over its lifetime).

## Sample code

The first sample is for a page that only wants to use the local presentation
mode.

```javascript
<!-- Controlling page -->
let presentationWindow;
const request = new PresentationRequest('about:blank');
request.start().then(result => {
  <!-- Only a window object can be returned for 'about:blank'. -->
  presentationWindow = result;
  <!--
    presentationWindow is opened to about:blank and is mirrored to the display. 
    This page has access to presentationWindow and presentationWindow.opener
    points to window.
  -->
});
```

The second sample is a controlling page that wants to use either local
presentation mode, or remote presentation (2-UA) mode.  This allows a site to
target the widest range of presentation displays, if it is willing to implement
somewhat different behavior for each type.

```javascript
<!-- Controlling page -->
let presentationWindow;
let presentationConnection;
const request = new PresentationRequest(['about:blank',
    'https://www.example.com/remote-slides.html']);
request.start().then(result => {
  <!-- Window object is only returned for 'about:blank'. -->
  if (result.__proto__.isPrototypeOf(window)) {
    presentationWindow = result;
  } else {
    // Handle connection to remote presentation.
    presentationConnection = result;
  }
});
```

## Proposed IDL

```
partial interface PresentationRequest {
  Promise<PresentationConnection|Window>   start();
}
```

## Feature detection

If shipping implementations do not treat 'about:blank' as a compatible URL, then
the site can use screen availability of that URL to feature detect local
presentation mode.

TODO: Check Firefox behavior

## Privacy and security considerations

We need to ensure same-origin policy is not violated by if the presentation
window is streamed to another device.  Target displays should be trusted,
similar to the displays used for features like tab mirroring or desktop
screenshare.

Since the presentation window starts in fullscreen, the user agent should
display the current origin of presentation windows elsewhere, and allow the user
to view its location bar and security enamel by dropping it out of fullscreen.
