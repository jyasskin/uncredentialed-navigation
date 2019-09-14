# Uncredentialed Navigation

## Problems

Navigation to signed [web packages](https://github.com/WICG/webpackage) [needs
to be done without credentials](https://github.com/WICG/webpackage/issues/423)
to prevent the distributor from conveying its user ID to the publisher. If the
package's contents are all same-origin with the distributor, this isn't strictly
necessary, but we should probably require it for signed packages anyway to
minimize the chance that they accidentally include personalized data.

Cross-origin *prefetches* need a similar capability per discussion in
https://github.com/w3c/resource-hints/issues/82. Specifically, if credentials
are sent, this allows the two servers to look at which users made requests at
close to the same time, and correlate cross-origin user IDs without a navigation
by the user. However, it's not clear that the matching navigation needs to be
flagged as uncredentialed: if there's an acceptable prefetch available, maybe it
should just use that?

## Security

There are worries that uncredentialed navigation could be used in an attack on
the target site, since the main request won't send or save cookies, but
subrequests will. We don't actually know a concrete attack that's enabled by
uncredentialed navigations, but we'd like to prevent surprises anyway.

The current idea is to require that the response either is a signed web package
(which, being intended for redistribution, shouldn't be personalized anyway) or
includes a response header declaring that it's expecting to be used without
credentials. The current proposed spelling for this header is
[`Allowed-Uncredentialed-Navigation`](https://github.com/w3c/resource-hints/issues/82#issuecomment-529951528).

We might want a request header declaring that the current request omits
credentials in order to tell the server that it ought to send the response
header.

If the response header is missing in a prefetch, the browser should drop the
prefetch, so any subsequent navigation is done with credentials. This argues for
not-flagging `<a>` links that are expected to be provided by a prefetch

If a site requests an uncredentialed navigation as would be required by a web
package, but hits another kind of resource instead, and the
`Allowed-Uncredentialed-Navigation` header isn't present, the navigation should
probably be restarted as a credentialed navigation.

## Solutions using existing things

The existing [`crossorigin`
attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_settings_attributes)
looks sufficient for prefetches in its `anonymous` mode (same-origin credentials
mode), and if the `<a>` needs to be tagged to match, it could also use the
`crossorigin` attribute. However, since prefetches will simply be ignored if
they don't have this attribute,
[Anne](https://github.com/w3c/resource-hints/issues/82#issuecomment-531276550)
may be right that we should just invent a new `rel` value.

Web Packages, however, need to omit credentials even for same-origin requests,
so there's no existing `crossorigin` value that meets the need. A prefetch of a
Web Package needs the same behavior.

Any way of doing this will require changes to
[Fetch](https://fetch.spec.whatwg.org/), which currently assumes that
navigations include credentials.

## Novel solutions for packages

### `<a href="https://target" credentials="omit">`

Directly declares that the fetch should be done with [the "omit" credentials
mode](https://fetch.spec.whatwg.org/#concept-request-credentials-mode).

### `<a href="https://target" crossorigin="no-credentials">`

The same semantics as `<a href="https://target" credentials="omit">`, but
reusing the existing
[`crossorigin=""`](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_settings_attributes)
attribute. In browsers that don't implement this it will be equivalent to
`crossorigin="anonymous"` (= same-origin credentials mode).

### `<a href="https://target" fetchoptions='{"credentials": "omit"}'>`
### `<a href="https://target" fetchoptions="credentials: omit">`

This is a general way of setting the value of the [_init_ parameter to
`fetch()`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch#Parameters).
This may be more attractive than a dedicated attribute to set the credentials
mode because it solves a larger class of problems.

Folks have long discussed a general way to set fetch options on various
fetch-causing HTML elements, unifying the current `crossorigin=""`,
`referrpolicy=""`, and `integrity=""` attributes, and gives a better future
extension point. The two hard parts here are

1. Getting people to swallow putting JSON in an attribute, or alternately coming
   up with a new microsyntax that has enough capabilities.
2. Checking all of the options for possible security holes.

### `<a href="https://package" publisherurl="https://signed_inner_url">`

An attribute declares the expected start_url of the package (or inner url of a
signed exchange). If these don't match ... redirect?

### `<a href="https://signed_inner_url" fetchfrom="https://package">`

Naturally works on browsers that haven't implemented packages. Allows any
browser to skip the package if it wants to make sure the publisher is
notified/checked/etc.

### `<a href="https://signed_inner_url" distributor="https://distributor.origin">`

This relies on us defining a single path within `distributor.origin` that serves
a given signed URL, as
https://github.com/WICG/webpackage/issues/423#issuecomment-510630261 speculates.
It has similar properties to the `fetchfrom` option otherwise.

### `<a href="https://sxg" ispackage>`

Just tell the browser directly to expect a package.

### `<a href="package://package">`

Basically encodes the `ispackage` flag into the scheme of the URL. The package
is fetched by changing the scheme to `https`. If we develop a more general
package scheme, it's probably usable for this.

### `<a href="https://signed_inner_url">`

This assumes there has already been a non-credentialed prefetch that found a
package holding the mentioned URL, and uses that package for the navigation. It
needs one of the above schemes to mark the `<link rel="prefetch">` as
uncredentialed in the first place.

This idea comes from Ben Schwartz as a way to prevent packages from being used
for click tracking.




