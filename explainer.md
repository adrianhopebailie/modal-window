# Explainer

Authors:

- Adrian Hope-Bailie (Coil)
- Marcos Caceres (Mozilla)

Contributors:

- Rouslan Solomakhin (Google)
- Junkee Song (Microsoft)
- Sam Goto (Google)

In order to perform certain functions that involve interactions with a
third-party such as authentication, sharing, and payments, we require either
pop-ups, opening a new tab, a redirect or similar.

The Web Payments WG defined an API for requesting payments resulting in the
invocation of a payment handler (third-party service) which may be a native app
(e.g. ApplePay from Safari) or a worker from an origin different to the calling
context.

This has necessitated the introduction of a new UI component that allows the
worker to display UI to the user in the form of a modal window with top-level
context.

The current solution (implemented in Chrome) defines a platform feature
(PaymentRequestEvent.openWindow()) that is only available inside the context of
a Payment Request.

This component is clearly useful for other use cases (e.g. authentication,
sharing, access to third-party services) and an innovative use of the payment
APIs (creation of a fake payment context) to demonstrate this is shown in this
demo:

https://rsolomakhin.github.io/pr/apps/password/

## Problem

Many use cases (authentication, payment, share, etc) require invocation of a
third-party service from another origin that requires a user interface.

## Solutions Considered

### IFrames

Many third-party services require a top-level context, often for security
reasons. As a minimum security requirement, the origin of the third-party should
be shown along with its user interface.

A challenge with iframes is click-jacking. The mechanisms in place to solve this
often make use cases for which modal windows would be useful (payments,
authentication etc) impossible to implement.

Many services have used iframes in the past to improve, for example, third-party
login flows such as single-sign-on. Recent changes in many browser to restrict
cross-origin cookies have broken these flows.

### Pop-Ups

When a user switches tabs or clicks on the browser window for any reason,
the popup would be hidden behind the browsing window.

If the user is using the popups for payment apps in comparison shopping scenario,
it is difficult to keep track of which popup belongs to which tab.

Pop-ups are generally locked down and difficult to invoke reliably due to the
measures introduced by browsers to counter their abuse.

Given their modal nature, we can’t yet think of a good way to abuse modal
windows. The assumption being that only a single modal window will be allowed at
a time.

### Redirects

This is the most common way to deal with these use cases. It is a high-friction
solution that causes the original context to be torn down and recreated. The
calling origin also loses control of the context so an error in the third-party
service can mean the user never returns.

## Goals

Provide a mechanism for websites to display a window that allows for short-lived
cross-origin interactions.

Common requirements of the user interface for these use cases are:

1.  a top-level browsing context that displays third party content.
2.  it's modal.
3.  it should be possible to position this browsing context at least relative to
    the top or bottom of the container window, and perhaps have the ability to
    visually expand the context (or let the user expand it) - and the ability to
    go fullscreen. The browsing context (not the opener) controls the
    dimensions.
4.  the opener context needs to set the feature policy (e.g., allow web authn,
    camera access, credential management).
5.  the modal window API itself will be disabled in cross-origin iframes by
    default, except when explicitly enabled with feature policy, e.g.,
    allow=”modal-window”.
6.  the opener context must have a means to have by bi-directional communication
    channel (i.e., message ports or just post message).
7.  Both the opener context and the browsing context must have the ability to
    close the browsing context.
8.  Both the opener and the browsing context must be in secure context.
9.  Only the origin indicated by the opener context should have the ability to
    message back to the opener context.
10. The API might need an ability to indicate the kind of service that's needed
    (e.g., "payment", "authentication", "share", "mixed?") in the future, but
    since the current implementations don’t do anything use-case specific from a
    UX perspective, there’s no product need for this feature yet.

## Prior Art

### showModalDialogue

A previous DOM API `window.showModalDialogue(...)` was introduced in IE and then
added to various other UAs:
https://msdn.microsoft.com/en-us/ie/ms536759(v=vs.94)

#### Difference from current proposal:

For some details of the history behind `window.showModalDialogue(...)` see
[“Removing showModalDialog from the Web Platform”](https://dev.opera.com/blog/showmodaldialog/)

The `showModalDialog()` API blocked execution of JavaScript in the parent window,
similar to `alert()`. This proposal does not intend to do that.

This proposal should address the security concerns raised with the previous
`window.showModalDialogue(...)` implementation which appear to be primarily
related to how the feature was implemented and not the feature itself.

### HTMLDialogueElement

See https://demo.agektmr.com/dialog/

#### Difference from current proposal:

`HTMLDialogueElement` is a DOM element (i.e. same origin) which would be
challenging to use to host, for example payment handler UI from a different
origin.

### Portals (Chrome)

See https://web.dev/hands-on-portals

#### Difference from current proposal:

Is only interactive when it becomes the top level context.

## Example API Usage

### Opener context (Window or Worker)

```javascript
const modalWindow = await window.openModal(
    'https://authorization-server.com/auth?response_type=code&scope=photos&state=1234zyx');
// modalWindow is an instance of Window (https://developer.mozilla.org/en-US/docs/Web/API/Window)
window.addEventListener('message', (e) => {
  // Check origin
  if ( e.origin === 'https://authorization-server.com' ) {
      // Retrieve data sent in postMessage
      const data = e.data;
      // Send reply to source of message
      e.source.postMessage('some reply', e.origin);
  }
}, false);
modalWindow.postMessage('some message', 'https://authorization-server.com');
```

### Modal Window Context

```javascript
window.addEventListener('message', (e) => {
  // Check parent origin is for a valid client
  const client = getClientFromOrigin(e.origin)
  if ( client ) {
      // Retrieve data sent in postMessage
      const data = e.data;
      // Send reply to source of message
      e.source.postMessage('some reply', e.origin);
  }
}, false);

window.parent.postMessage('some message', '*');
```

## Security Self-Assessment

1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

> None

2. Is this specification exposing the minimum amount of information necessary to power the feature?

> Yes

3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

> Not Applicable

4. How does this specification deal with sensitive information?

> Not Applicable

5. Does this specification introduce new state for an origin that persists across browsing sessions?

> No

6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

> None

7. Does this specification allow an origin access to sensors on a user’s device

> No

8. What data does this specification expose to an origin? 

> This feature is identical in this respect to `window.open`

9. Does this specification enable new script execution/loading mechanisms?

> Yes, the specification enables opening a new top level context, similar to a popup.

10. Does this specification allow an origin to access other devices?

> No

11. Does this specification allow an origin some measure of control over a user agent’s native UI?

> Yes, to some degree. It allows a website to create a new modal window context which makes the calling context inaccessible until the modal is closed. To mitigate abuse of this capability it is recommended that only a single modal can be open per parent context.

13. How does this specification distinguish between behavior in first-party and third-party contexts?

> This feature should only be available to third-party contexts if it is explicitly allowed by the first-party (e.g. through feature policy)

14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

> The behaviour should be no different. The new context that is created will also be in "incognito" mode if the opener is

15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

> Not yet in this explainer. One will be included with in the final spec.

16. Does this specification allow downgrading default security characteristics?

> No
