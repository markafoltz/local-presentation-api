# Local Presentation Mode

## Background

The [Presentation API](https://w3c.github.io/presentation-api/) allows the user
to request that a [presentation
URL](https://w3c.github.io/presentation-api/#dfn-presentation-url) be shown on
another display. This can be done in two ways.  In 1-UA mode, the same browser
renders the presentation and sends the resulting audio/video output to the
target display.  In 2-UA mode, the browser sends just the URL to the target
display, which is responsible for rendering the presentation content.

In either case, the user agent rendering the presentation [creates a distinct
profile](https://w3c.github.io/presentation-api/#creating-a-receiving-browsing-context)
(storage area) for the presentation, that is discarded when the presentation is
closed.  This minimizes the developer-exposed differences between 1-UA and 2-UA
modes, and improves privacy for users of shared displays.

We have found that 2-UA mode works well for presentations that are mostly
focused on media playback, as they get to use the full audio and video
capabiliites of the target display.  However, for productivity applications,
like presenting a deck of HTML slides, neither 1-UA or 2-UA mode have proven
suitable.

We propose a new mode for the Presentation API that removes the restriction on
presentations to use a separate storage area, calling it "local presentation
mode."  The result is similar to invoking `window.open`, then capturing the
output of the new window and streaming it to a target display.  In turn, local
presentation mode adds additional restrictions on the behavior of the
presentation.

If this mode proves suitable for a wide range of applications, the plan is for
the spec to no longer treat the existing 1-UA mode (with a separate storage
area) as a [possible
implementation](https://w3c.github.io/presentation-api/#introduction) of the
Presentation API.  2-UA mode would remain, with the storage area restrictions in
place when the presentation is rendered on another display.

## Use cases 

Content that uses complex, interactive content assembled from many components
and origins have found it difficult or impossible to use the current
Presentation API for reasons listed below.

The example driving these examples is a Web slide show.  For simplicity I'll use
_controlling page_ for the page that starts the presentation and controls the
slide show, and _presentation_ for the page showing the slides on the target
display.  While the slide show is being presented, the controlling page would
show slide thumbnails, speaker notes, and controls to advance the slides in the
presentation.

Here are some of the issues with the current Presentation API 1-UA mode:

### Third party embeds

The presentation cannot embed private third party resources that require
specific cookies or profile state, since it can't set cookies on the third party
origin.
  
> Example: A slide embeds a private YouTube video that requires a youtube.com
> login cookie.

### Offline
  
The presentation does not have access to offline storage, and the controlling page
cannot easily present content that is offline.  There is no URL to navigate to,
and all of the offline storage has to be serialized and sent to the presentation.

> Example: A salesperson stores their slideshow offline and wants to present it
> on a wired display without connecting to WiFi.

### WSIWYG

Editing the content of the presentation while presenting requires significant
changes to the application.  The entire data model (stored as Javascript objects
and/or Web components) has to be synchronized between the controlling page and
the presentation using messages.

> Example: The user is editing the slide show content and immediately wants to
> see the results while presenting. 

### Interaction (touch/gesture)

It is difficult to interact directly with the presentation in response to mouse
or touch on the controlling page.  The touch or mouse event has to be captured,
serialized, and coordinates translated to find the correct element on the
presentation.  The presented component then has to be updated to respond to
these synthetic events.

>  Example: On the controlling page, the salesperson uses a touchpad to
>  highlight an element of the slide with popup content.  They want the same
>  popup to appear on the presentation.

## Local Presentation Mode

The proposal is to meet the use cases above by adding a new mode to the
Presentation API that requests that the presentation URL be opened in a window
that functions similar to a window created by `window.open`.  The presentation
shares the storage area with the controlling page.  If the presentation and the
controlling page are same-origin, they will have access to each other's
`window` object.

In turn the presentation has the following restrictions relative to a
presentation started in the regular (non-local) mode:
  * It will always be rendered on the controlling user agent.
  * No `PresentationConnection` is returned, so cross-origin messages have to be
    exchanged using `postMessage` on the respective `window` objects.
  * It cannot be accessed after navigation by the controlling page via
    `PresentationRequest.reconnect()`.

## Sample code

This code creates a `PresentationRequest` that, when started, opens the URL in a
new local window, returns the corresponding `window` object, and streams the content
of the window to the chosen presentation display.

```javascript
<!-- Controlling page, https://www.example.com/index.html -->
let presentationWindow;
const request = new PresentationRequest('https://www.example.com/slides.html',
    {isLocal: true});
request.start()  // User is prompted to choose a presentation display.
    .then(result => {
     <!-- URL is rendered in a new window and its object is returned. -->
     presentationWindow = result;
     <!-- Since presentationWindow is same-origin, presentationWindow.opener == this. -->
});
```

## Proposed IDL Changes

Changes:
* Adds a dictionary `PresesntationRequestOptions` with a single property `isLocal`.
  * This allows local presentation mode to be feature detected before the
    constructor is invoked.
* Adds `PresentationRequestOptions` to the `PresentationRequest` constructor.
  * Note that only a single URL is supported when local mode is requested. 
* Adds `isLocal` property to `PresentationRequest` that reflects the
  option value.
* Adds `Window` as a legal type to resolve the `start` Promise.


```javascript
  dictionary PresentationRequestOptions {
    boolean isLocal = false;
  };

  [Constructor(USVString url, optional PresentationRequestOptions options),
   Constructor(sequence<USVString> urls, optional PresentationRequestOptions options),
   SecureContext,
   Exposed=Window]
  partial interface PresentationRequest {
    readonly boolean attribute isLocal;
  };
  

  partial interface PresentationRequest {
    Promise<PresentationConnection|Window>   start();
  }
```

## Privacy and security considerations

The privacy and security considerations for local presentation mode are similar
to [existing ones for the Presentation API](
https://w3c.github.io/presentation-api/#security-and-privacy-considerations).

Because the presentation will be rendered on the same user agent, the privacy
concerns about leaving browser state on a shared display do not apply to local
presentation mode.

We need to ensure same-origin policy is not violated when the presentation
window contains cross-origin content and is streamed to the display.  Displays
should be trusted by the controlling user agent and not under the control of the
controlling page.

Since the presentation window will typically start in fullscreen on the target
display, the user agent should display the origin of the presentation URL when
presentation is requested.  (This recommendation is already included as a
User Interface Guideline.)
