# Multi-Screen Window Placement on the Web - Creating Fullscreen Popup Windows

## Introduction

This proposal introduces an additional way for web applications to initiate fullscreen experiences on multi-display configurations. The proposed enhancement would allow web applications to open a new window directly into fullscreen on a specified display with a single API call using the existing [window.open()](https://html.spec.whatwg.org/multipage/window-object.html#dom-open-dev) API.

## Background

[Multi-Screen Window Placement](https://github.com/webscreens/window-placement)
is a new permission-gated web platform API that provides web applications with
information about the device's connected screens, allows them to open and
position windows or request fullscreen on any of those screens.

The HTML living standard specifies
[Tracking User Activation](https://html.spec.whatwg.org/multipage/interaction.html#tracking-user-activation),
to gate some web platform APIs on user interaction signals. The transient user
activation signal (e.g. from a mouse click) is required and consumed when web
application scripts open a popup window via `Window.open()`, and when they request
fullscreen via `Element.requestFullscreen()`.

## Problem

Web applications are limited in the ways that they can [Initiate Multi-Screen Experiences](https://github.com/w3c/window-placement/blob/main/EXPLAINER_initiating_multi_screen_experiences.md) across multi-display configurations. Specifically, opening a popup and transitioning the window to fullscreen requires two separate [user activations](https://html.spec.whatwg.org/multipage/interaction.html#tracking-user-activation), consumed by `window.open()` and `Element.requestFullscreen()` respectively. This results in a cumbersome user experience for web applications which utilize multiple displays.

Multiple partners/early adopters of the API have specifically requested this enhancement (e.g. [window.open should support the 'fullscreen' option](https://github.com/w3c/window-placement/issues/7)) and it's considered a prerequisite to at least one other feature request (e.g. [Feature request: Fullscreen support on multiple screens](https://github.com/w3c/window-placement/issues/92))


## Use Cases

Some basic illustrative examples which may be achieved with this proposal are listed below:

* Video streaming app launches a video directly to fullscreen on a secondary display.
* 3D modelling app allows user to launch a preview in fullscreen on a secondary display.

Web applications may use this enhancement in conjunction with the existing [ability to enter fullscreen and open a popup from a single user gesture](https://w3c.github.io/window-placement/#usage-overview-initiate-multi-screen-experiences). By doing so, web applications can launch fullscreen experiences on the initial / primary display and a secondary display from a single user gesture. Some examples are listed below:

* Financial app opens a fullscreen dashboard on the primary monitor and a fullscreen stock tracker window on a secondary display.
* Gaming app launches a fullscreen window on both primary and secondary displays for a continuous widescreen view.
* Virtual desktop app launches fullscreen windows on primary and secondary displays to mimic a remote monitor configuration.


## Goals

The goal of this document is to outline the necessary spec changes that would permit web applications to open a fullscreen window on a secondary display. It also aims to explore abuse and other security / privacy considerations. Some specific goals are as follows:

* Extend the `Window.open()` algorithm to change the behavior of the opened window based on parameters passed.
* Explore the behavior of user agents when the launched window exits fullscreen.

The [Initiating Multi-Screen Experiences Explainer](https://github.com/w3c/window-placement/blob/main/EXPLAINER_initiating_multi_screen_experiences.md) covers some related goals (i.e. launching N fullscreen windows on N displays) not covered in this proposal. 

## Proposal

Allow sites with the `window-placement` permission to open a fullscreen window on a secondary display from a single user gesture. When the device has multiple screens and `Window.open()` targets the secondary screen with an appropriate flag to create the window as fullscreen (via the `features` argument), the user agent should open the window as fullscreen on the display which contains the coordinates (`left`, `top`, `width`, `height`) of the new window. When the new window ultimately exits fullscreen, the window shall be restored as a popup style (bordered) window matching the coordinates originally passed to `Window.open()`.


### Example Code

```js
 const otherScreen = screenDetails.screens.find(s => s !== primaryScreen);
  window.open(url, '_blank', `left=${otherScreen.availLeft},` +
                             `top=${otherScreen.availTop},` +
                             `width=${otherScreen.availWidth},` +
                             `height=${otherScreen.availHeight},` + 
                             `fullscreen=true`);
```

### Spec Changes

The spec does not contain an exhaustive list of valid features in the `features` argument, however, the spec does contain procedures for determining if a [popup window is requested](https://html.spec.whatwg.org/multipage/window-object.html#popup-window-is-requested), and the specific `tokenizedFeatures` to look for. Following this model, the spec shall be updated to define a similar algorithm for checking if a **fullscreen window** is requested. A fullscreen window must also first be considered a popup window and also have `left` and `top` set in `features`. `left` and `top` determine which display the window will go fullscreen on, and the position of the popup when fullscreen is exited.

After [checking if a popup window is requested](https://html.spec.whatwg.org/multipage/window-object.html#popup-window-is-requested), add the following section:

To **check if a fullscreen window is requested**, given *tokenizedFeatures*:


1. If *tokenizedFeatures* is empty, then return false. 
2. Let *popup* be the result of [checking if a popup window is requested](https://html.spec.whatwg.org/multipage/window-object.html#popup-window-is-requested).
3. If *popup* is false, then return false.
4. If *tokenizedFeatures*["`left`"] and *tokenizedFeatures*["`top`"] do not [exist](https://infra.spec.whatwg.org/#map-exists), then return false.
7. Let *fullscreen* be the result of [checking if a window feature is set](https://html.spec.whatwg.org/multipage/window-object.html#window-feature-is-set), given *tokenizedFeatures*, "`fullscreen`", and false.
8. If *fullscreen* is false, then return false.
9. Let *permissionState* be [request permission to use](https://w3c.github.io/permissions/#dfn-request-permission-to-use) "`window-placement`".
10. If *permissionState* is not "[granted](https://w3c.github.io/permissions/#dom-permissionstate-granted)", then throw a "[NotAllowedError](https://webidl.spec.whatwg.org/#notallowederror)" [DOMException](https://webidl.spec.whatwg.org/#idl-DOMException).
11. Return true.

### Open questions

#### Feature detection

There is no standard API for querying which features are supported in the [Widow.open()](https://html.spec.whatwg.org/multipage/window-object.html#dom-open-dev) `features` argument. There is no way for a web application to check that the user agent supports the `fullscreen` key in the `features` argument without calling the API and detecting if the resulting window is fullscreen (i.e. window boundary comparisons).

The ability to detect supported keys of the `features` argument is out of scope for this proposal.

#### Exiting fullscreen (Restoring bounds)

The living standard algorithm for [exit fullscreen](https://fullscreen.spec.whatwg.org/#exit-fullscreen) does not specify the user agent's behavior of the restored window:

```
9. If resize is true, resize doc’s viewport to its "normal" dimensions.
```

It may be worth exploring an extension of the spec to specify that a window opened directly to fullscreen using this proposal, should be restored to the bounds originally specified in the (`left`, `top`, `width`, `height`). In the meantime, a user agent may decide how the window is restored.


#### Overlapping Fullscreen Windows / Invalid coordinates

This proposal & spec change does not define the behavior of the user agent when the requested coordinates (`left`, `right`) overlap an existing (fullscreen) browser window. The intention of this proposal is for web applications to launch popups on secondary displays, however that may not always happen due to user / developer errors or defects. For example, a web application creating a fullscreen window on in a single display environment with an already fullscreen browser window.

Each user agent may define their own behavior and security safeguards for these scenarios. For example pre-existing protections prevent placing popups over fullscreen:
* Chrome exits fullscreen (see
    [ForSecurityDropFullscreen](https://source.chromium.org/chromium/chromium/src/+/main:content/public/browser/web_contents.h?q=ForSecurityDropFullscreen))
* Firefox exits fullscreen
* Safari opens the popup in a separate fullscreen space

## Security Considerations

A notable security consideration stems from the fact that the web application may launch a fullscreen window on a display that the user is not looking at, since the [user activation](https://html.spec.whatwg.org/multipage/interaction.html#tracking-user-activation) (i.e. button click) may have occurred on another display which the user is focused on. The user may not notice the fullscreen window transition, nor the fullscreen bubble (e.g. Firefox's "\<origin\> is now full screen [Exit Full Screen (Esc)]" or Chrome's "Press [Esc] to exit full screen") which could allow for a malicious application to mimic other applications or the operating system without the user realizing that it is a browser window.

This is partially mitigated by gating this feature on the "`window-placement`" permission. Users should only grant permissions for [powerful features](https://w3c.github.io/permissions/#dfn-powerful-feature) to web applications that they trust.

This could also be mitigated further by having the user agent show the fullscreen bubble only when the user first interacts with the cross-display fullscreen window. In this scenario, there is a higher confidence that the user will see the fullscreen bubble.

## Privacy Considerations

This feature does not expose any information to sites, and there are no privacy considerations to note beyond those already documented in the Multi-Screen Window Placement [Privacy Considerations](https://www.w3.org/TR/window-placement/#privacy) section. The feature is gated behind the "`window-placement`" permission, so it does not expose any information outside what is already available from the API when permission is granted.

## Alternatives Considered

 * Keeping the existing behavior. Web applications can create a popup, and then expand it's bounds to consume the entire display to mimic fullscreen content.
   * This contains window borders and decorations not appropriate for some use cases.
 * Allow a target-screen fullscreen request after opening a cross-screen popup
   * This is less intuitive for the application developer and requires more complex changes to user activation signals which has a higher risk for misuse or abuse.