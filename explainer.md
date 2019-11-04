# Explainer
In order to perform certain functions that involve interactions with a
third-party such as authentication, sharing, and payments, browsers rely opening
a new tab, performing a redirect, or similar. 

From a UX perspective, although they work, these experiences are often suboptimal 
to those provided by native applications: where authentication flows or payment flows
are more cohesively integrated into the user experience either via native UI componets
or some kind of "webview".  

![embedded applications](https://img.youtube.com/vi/yvStdEd_oCw/hqdefault.jpg)
https://youtu.be/yvStdEd_oCw

The Web Payments WG defined an API for requesting payments resulting in the
invocation of a payment handler (third-party service) that may be either a native app
(e.g. ApplePay from Safari) or a worker from an origin different to the calling
context.

This necessitated the introduction of a new UI component that allows the
worker to display UI to the user in the form of a modal window with the privileges of a top-level
browsing context.

The current solution (implemented in Chrome) defines a platform feature
(`PaymentRequestEvent.prototype.openWindow()`) that is only available inside the context of
a Payment Request.

![payment handler](https://img.youtube.com/vi/IK_SlT6zm4I/hqdefault.jpg)
https://www.youtube.com/watch?5=&v=IK_SlT6zm4I

However, the presented UI component could be useful for other use cases (e.g. authentication,
sharing, access to third-party services) and an innovative use of the payment
APIs (creation of a fake payment context) to demonstrate this is shown in this
demo:

https://rsolomakhin.github.io/pr/apps/password/

![Payment handler going beyond handling payment](https://user-images.githubusercontent.com/870154/68000787-8b6d7f80-fcb5-11e9-8eae-04c4b3c8eab0.png)

## Problem

Many use cases (authentication, payment, share, etc) require invocation of a
third-party service from another origin that requires a distinct user interface.

For example, sharing a URL with the notes application in MacOS:

![Sharing with notes](https://user-images.githubusercontent.com/870154/68001106-c3c18d80-fcb6-11e9-894f-5ad99c43166b.png)

## Solutions Considered

### IFrames

Many third-party services require a top-level browsing context, often for security
reasons. As a minimum security requirement, the origin of the third-party should
be shown along with its user interface.

A challenge with iframes is click-jacking. The mechanisms in place to solve this
often make use cases for which modal windows would be useful impossible to implement 
(e.g., payments, authentication, etc.).

Many services have used iframes in the past to improve, for example, third-party
login flows such as single-sign-on. However, because of pervasive user tracking, 
recent changes in many browser to restrict cross-origin cookies have broken these flows.

### Pop-Ups/new tabs

When a user switches tabs or clicks on the browser window for any reason,
the popup would be hidden behind the browsing window. On mobile devices, 
popups can be confusing to users, as closing a browser "tab" on a mobile 
can require several presses withing a mobile browser's UI. 

If the user is using the popups for payment apps in a comparison-shopping scenario,
it is difficult to keep track of which popup belongs to which tab.

Pop-ups are generally locked down and difficult to invoke reliably due to the
measures introduced by browsers to counter their abuse. Some mobile browsers
also limit the number of tabs that can be opened at any one time 
(leading to potential data loss). 

Given their modal nature, we can’t yet think of a good way to abuse modal
windows. The assumption being that only a single modal window will be allowed at
a time.

### Redirects

This is the most common way to deal with these use cases. It is a high-friction
solution that causes the original context to be torn down and recreated. The
calling origin also loses control of the context so an error in the third-party
service can mean the user never returns... or returns to the wrong place 
or the state or scroll position is lost (leading to a loss of context for the end-user).

## Goals

Provide a mechanism for websites to display a top-level browsing context 
that allows for short-lived cross-origin interactions.

Initially identified requirements for such a UI component are:

1.  acts as a top-level browsing context that displays third party content.
2.  it's modal.
3.  the abilty to position this browsing context at least relative to
    the top or bottom of the container window, and perhaps have the ability to
    visually expand/contract the context (or let the user expand it) - and the ability to
    go fullscreen. The browsing context (not the opener) controls the
    dimensions.
4.  the opener context needs to set the feature policy (e.g., allow web authn,
    camera access, credential management). 
5.  the modal window API itself will be disabled in cross-origin iframes by
    default, except when explicitly enabled with feature policy, e.g.,
    allow=”modal-window”.
6.  the opener context must have a means to have a bi-directional communication
    channel (i.e., some means to post or stream messages back and forth).
7.  Both the opener context and the browsing context must have the ability to
    close the browsing context.
8.  Both the opener and the browsing context must only be available in a secure context.
9.  Only the origin indicated by the opener context should have the ability to
    message back to the opener context.
10. The API might need an ability to indicate the kind of service that's needed
    (e.g., "payment", "authentication", "share", "mixed?") in the future, but
    since the current implementations don’t do anything use-case specific from a
    UX perspective, there’s no product need for this feature yet.
11. only triggered by user activation. 
12. modal window uses a separate browsing context group, which precludes synchronous DOM access, named access to each others' browsing context, etc.
13. Only a slim interface with `postMessage()` can be used to communicate between the opener and the modal window.

## Existing Web Platform solutions 

### `showModalDialog()`

A previous DOM API `window.showModalDialog(...)` was introduced in IE and then
added to various other UAs:
https://msdn.microsoft.com/en-us/ie/ms536759(v=vs.94)

#### Difference from current proposal:

For some details of the history behind `window.showModalDialog(...)` see
[“Removing showModalDialog from the Web Platform”](https://dev.opera.com/blog/showmodaldialog/)

The `showModalDialog()` API blocked execution of JavaScript in the parent window,
similar to `alert()`. This proposal does not intend to do that.

This proposal should address the security concerns raised with the previous
`window.showModalDialog(...)` implementation which appear to be primarily
related to how the feature was implemented and not the feature itself.

### `HTMLDialogElement`

See https://demo.agektmr.com/dialog/

#### Difference from current proposal:

`HTMLDialogElement` is a DOM element (i.e. same origin) that can't display remote content. It can only contain `<iframe>`s. 

### Portals (Chrome)

See https://web.dev/hands-on-portals

#### Difference from current proposal:

Is only interactive when it becomes the top-level browsing context.

## Security Self-Assessment

> 1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

Possibly referrer information and other information that could be leaked via a URL. With a communication channel, any arbitrary information could be exchanged between the opener and the remote browsing context.    

> 2. Is this specification exposing the minimum amount of information necessary to power the feature?

Maybe. Still figuring that out. 

> 3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

It doesn't. It's intended purpose is to facilitate the flow of such information between two trusted parties to, for example, facilitate authentication and payments.  

> 4. How does this specification deal with sensitive information?

It only works in SecureContexts. It tries to limit communication exclusively between the opener and the opened browsing context.  

> 5. Does this specification introduce new state for an origin that persists across browsing sessions?

Unsure.

> 6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

None

7. Does this specification allow an origin access to sensors on a user’s device

Yes, via feature policy applied to the opened browsing context, which itself could access allowed powerful features. 

> 8. What data does this specification expose to an origin? 

This feature is identical in this respect to `window.open`, meaning it can pass along all sorts of information.  

> 9. Does this specification enable new script execution/loading mechanisms?

Yes, the specification enables opening a new top level context, similar to a popup.

> 10. Does this specification allow an origin to access other devices?

Yes. But controlled by feature policy.  

> 11. Does this specification allow an origin some measure of control over a user agent’s native UI?

Yes, to some degree. It allows a website to create a new modal window context which makes the calling context inaccessible until the modal is closed. To mitigate abuse of this capability it is recommended that only a single modal can be open per parent context.

> 13. How does this specification distinguish between behavior in first-party and third-party contexts?

This feature should only be available to third-party contexts if it is explicitly allowed by the first-party (e.g. through feature policy)

> 14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

The behaviour should be no different. The new context that is created will also be in "incognito" mode if the opener is.

> 15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Not yet in this explainer. One will be included with in the final spec.

> 16. Does this specification allow downgrading default security characteristics?

No

## TAG Review

[Requested on 28 September](https://github.com/w3ctag/design-reviews/issues/427)

## WICG

[Posted on 16 October](https://discourse.wicg.io/t/proposal-modal-window/3982)

## Acknowledgements

Contributors:

- Adrian Hope-Bailie (Coil)
- Rouslan Solomakhin (Google)
- Junkee Song (Microsoft)
- Sam Goto (Google)
- Marcos Caceres (Mozilla)
