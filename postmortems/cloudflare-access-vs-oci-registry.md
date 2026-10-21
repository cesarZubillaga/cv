# An SSO login page served as a container image

**Date:** 18–19 October 2026 · **Impact:** image automation stalled silently; image scanning broken

## Summary

The self-hosted Git server also serves as the OCI registry, behind a Cloudflare Tunnel
with Access in front of it. Access answers `/v2/` — the registry API root — with a 302 to
its login page, including for requests originating inside the cluster. Flux's image
reflector therefore saw no tags and stopped updating deployments, without reporting an
obvious error. Trivy, pulling the same images, failed with a parse error whose first
character was the giveaway.

## The two symptoms

Flux went quiet. There was no `UNAUTHORIZED`, which is what the stale comment in the
repository had led me to expect; the reconciliation simply stopped yielding new tags.

Trivy was louder, and more informative:

```
unable to get the image ID: invalid character '<' looking for beginning of value
```

That `<` is the opening angle bracket of the Access login page's HTML. The scanner asked
for a JSON manifest and was handed a web page.

## What was actually wrong

A path-based Access bypass added the previous day had opened `api/actions`,
`api/actions_pipeline` and `info/refs` — the paths the CI runners and `git` need. It did
not include `/v2/`, so every registry client still landed on the login redirect.

## Resolution

Rather than widen the Access policy, the pull path was moved off the public hostname:

- **For Flux:** a `hostAliases` entry applied **only to the image-reflector-controller
  pod**, resolving the public hostname to the in-cluster IP of the ingress controller,
  over plain HTTP (hence `insecure: true`).
- **For Trivy:** scan through the registry's internal service name with `--insecure` and
  a credential — the packages of a private repository are not anonymously readable.

The detail that makes the first fix work: **the Git server derives the `realm` of its
authentication challenge from the request's `Host` header, not from its configured root
URL.** So the second hop of the token dance — `/v2/token` — follows the alias as well. If
it had used the configured root URL, the client would have been bounced back out to the
public hostname and into the login page again.

The name has to stay the public one, because that is the string written into the
manifests, and because the registry credential is bound to that exact host — the pull
libraries do no hostname equivalence, and a mismatch surfaces as `UNAUTHORIZED`.

## What I changed afterwards

- **This affects everything that pulls through the public hostname**, not just Flux. I
  went looking for the other clients instead of declaring it fixed when Flux recovered.
- **`hostAliases` takes addresses, not names.** If the ingress controller's cluster IP
  ever changes, this breaks with `no such host`, so the check for that is written down
  next to the patch.
- **An authenticating proxy in front of a machine API is a tripwire.** Humans see a login
  page and sign in; machines see HTML where JSON should be and report something that
  sounds like data corruption.
