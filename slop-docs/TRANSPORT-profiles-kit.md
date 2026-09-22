# Transport kit: one profile per Companion platform (draft for amendment #190)

**Status: landed as amendment #190 on 2026-09-21 with all four section-6 decisions taken as drafted; this kit is the provenance record.** This kit
proposes the restructure of SPEC §5 from a single `android-loopback-tcp`
profile into a platform-neutral profile set covering every target the KMP
Companion can be built for today or is expected to be built for: Android,
desktop JVM, native Apple (iOS and macOS), Kotlin/JS on Node, and Kotlin/JS
or Kotlin/Wasm in a browser. Section 4 is normative text written in the
spec's voice and ready to lift; sections 1 to 3 are the fact base and the
reasoning; section 5 is the impact on each endpoint; section 6 lists the
owner decisions the draft takes a position on.

Sibling kit: `WEBSOCKET-transport-kit.md` (2026-07-19) is the source read of
`websocket.el` and the browser-profile sketch; this kit adopts its
conclusions unchanged and cites it rather than repeating the evidence.

## 1. What the current spec pins, and what it does not

Verified against `SPEC.md` 3.1.0-draft (protocol 3):

- §5.1 defines exactly one profile and says a future profile MUST
  completely define addressing, framing, authentication floor, connection
  initiator, and failure behavior before an implementation advertises it.
- §5.1 already decouples protocol role from transport role: "Emacs MUST
  send `session.hello` after a transport connection is established,
  regardless of which endpoint bound or dialed." Every profile below leans
  on that sentence; none of them changes who speaks first.
- §5.2 fixes for the loopback profile: Companion binds, Emacs dials,
  `127.0.0.1:8765`, §6 framing, §9 HMAC floor, single authority,
  supersession, jittered redial, the pre-authentication connection bound,
  and the "not encrypted" caveat that points at "a transport profile that
  provides it" without defining one.
- §5.3 is entirely Android: the wake target is a platform component start.
- §6.1 scopes the "no header field other than Content-Length" rule to the
  loopback profile and requires a future profile that permits more headers
  to define them.
- §9.3 and §23.6 both say a deployment that must defend against a local
  interposer MUST use a stronger profile. That profile does not exist yet.
- §24.1 requires "at least one complete transport profile, including
  Section 6 framing for `android-loopback-tcp`."
- Nothing on the wire names the profile. `session.hello` carries no
  transport member; the profile is implied by the connection itself. The
  string `android-loopback-tcp` appears in the spec, in
  `goldens/wire/manifest.json` (the `profile` field on two fixtures), and
  nowhere in `contract.json`.

## 2. What each Companion platform can actually do

The KMP build today declares `jvm()` and `android` (`ebp-kmp/build.gradle.kts`)
and `jvm()` alone for `wire`. No Apple, JS, or Wasm target is declared. The
`wire` module contains no listener at all (`grep ServerSocket` finds only
`ImageGuards.kt`); the real binder is `jetpacs-poc/companion/.../DeviceBridge.kt`,
which binds `127.0.0.1:8765` with a `java.net.ServerSocket`. That split is
the right one and the draft keeps it: the engine is transport-agnostic and
each platform supplies a transport.

| Platform / KMP target | Can bind a listener | Raw TCP | Same host as Emacs | Consequence |
|---|---|---|---|---|
| Android (`android`) | yes | yes | yes, or bridged by `adb forward` so both ends still see loopback | current profile, unchanged in substance |
| Desktop JVM (`jvm`) on Linux, macOS, Windows | yes | yes; JDK 16+ also AF_UNIX (JEP 380, believed to include Windows 10 1803+, unverified here) | yes | loopback profile verbatim; a Unix-socket variant closes §23.6's port-squatting hole |
| Kotlin/JS on Node (`js` nodejs) | yes (`net`) | yes | yes | loopback profile verbatim |
| macOS native (`macosArm64` etc.) | yes | yes | yes | desktop profile |
| iOS (`iosArm64` etc.) | yes while foregrounded; suspended in background | yes (`Network.framework`) | never — Emacs does not run on iOS | needs an off-box, encrypted profile |
| Browser: Kotlin/JS or Kotlin/Wasm (`wasmJs`) | no | no; WebSocket only | not guaranteed (a hosted page) | needs a WebSocket profile; loopback-only |
| Kotlin/Wasm WASI (`wasmWasi`) | networking immature | no stable API | n/a | out of scope; not a profile |

Facts about the Emacs endpoint that constrain the off-box and browser
profiles (verified on the local Emacs 31.1 by reading built-in docstrings;
the package floor is 30.1 and none of these changed between 30 and 31):

- `make-network-process` accepts `:family 'local` for Unix-domain sockets,
  client and server. Whether that works on Windows builds of Emacs is
  unverified; the draft makes the Unix profile OPTIONAL for that reason.
- `gnutls-boot`: "Currently only client mode is supported." Emacs can dial
  TLS. Emacs cannot terminate TLS as a server without an external helper.
  This single fact decides the initiator for the off-box profile: the
  Companion must be the TLS server and Emacs must dial, which is the same
  direction as loopback, so `ebp.el`'s whole redial machinery carries over.
- Emacs has no built-in WebSocket implementation. `websocket.el` (v1.16,
  pure Elisp, client and server, see the sibling kit) is a package, not a
  built-in, so an Emacs-side WebSocket transport cannot satisfy the
  `ebp.el` repository rule "loadable with only built-ins and files in this
  repository." See decision 6.2.

## 3. The design in one paragraph

Four profiles, one axis each. `loopback-tcp` is the existing profile with
the platform prefix dropped and its wake subsection made Android-specific.
`loopback-unix` is the same profile over a Unix-domain socket, for desktop
hosts that want filesystem-permission protection against local port
squatting. `network-tls` is the off-box profile §9.3 and §23.6 keep
pointing at: Content-Length framing inside TLS, Companion binds, Emacs
dials and pins the Companion's certificate, discovery is out of band. It
is the only profile an iOS Companion can implement and it is equally
useful for an Android or desktop Companion reached over Wi-Fi.
`loopback-websocket` is the browser profile from the sibling kit: Emacs
binds, the browser dials, one message per text frame, Origin allowlist,
loopback only. Nothing on the wire changes in any profile: the same
frames, the same proofs, the same method registry. `offline.wake` remains
grantable only where a platform defines an inert wake target, which today
is Android alone.

## 4. Proposed normative text for SPEC §5

Replace §5.1 through §5.3 with the following. Cross-reference edits
outside §5 are listed in 4.6.

### 4.1 §5.1 Profile invariance (replaces existing)

> Transport facts do not change endpoint authority or method direction.
> Emacs MUST send `session.hello` after a transport connection is
> established, regardless of which endpoint bound or dialed the underlying
> transport, and regardless of the profile in use.
>
> A transport profile is a complete statement of: the address family and
> default address; which endpoint binds and which dials; the framing that
> carries Section 7 messages; the authentication floor; confidentiality;
> the wake mechanism, if any; and failure behavior. This document defines
> the profiles in Sections 5.2 through 5.5. Every profile carries the
> Section 9 mutual HMAC handshake unchanged, the Section 10 session
> lifecycle unchanged, and the single-authority and supersession rules of
> Section 5.6. A profile identifier is a Section 4.4 identifier.
>
> An implementation MUST NOT advertise a profile this document does not
> define, and MUST document, in its pairing UI, which profiles it offers.
> Nothing on the wire names the profile: the connection itself determines
> it, and an endpoint MUST NOT accept a connection under one profile and
> then apply another profile's rules to it.
>
> The Companion and Emacs endpoints of one pairing MAY use different
> profiles across sessions. Pairing identity, tokens, receipts, and every
> Section 3 durable artifact are profile-independent.

### 4.2 §5.2 `loopback-tcp` (renames `android-loopback-tcp`)

> | Property | Requirement |
> |---|---|
> | Binder | Companion endpoint |
> | Dialer | Emacs endpoint |
> | Address family | IPv4 loopback |
> | Default address | `127.0.0.1` |
> | Default port | `8765` |
> | Framing | Section 6 Content-Length framing, no header field other than `Content-Length` |
> | Authentication floor | Section 9 mutual HMAC authentication |
> | Confidentiality | none |
> | Wake | platform-defined; see Section 5.7 |
>
> The Companion MUST bind only to an IP loopback interface. [Retain the
> existing §5.2 text from "It MUST NOT bind an EBP listener to `0.0.0.0`"
> through the pre-authentication bound and the "non-conformant" listener
> sentence, verbatim. Move the single-authority, supersession, and
> jittered-redial paragraph to §5.6 and cite it from here. Retain the
> device-lifetime firing sentence by citing §5.6.]
>
> This profile authenticates endpoints but does not encrypt messages, and
> an unprivileged local process that binds or races the configured port
> can interpose (Section 23.6). A deployment requiring confidentiality or
> interposition resistance MUST use Section 5.3 or Section 5.4.
>
> Platforms: an Android Companion, a desktop Companion on any operating
> system, a Node-hosted Companion, and a bridged deployment in which a
> platform port forward presents the Companion's loopback listener on the
> Emacs host's loopback interface all implement this profile without
> variation. The Android wake mechanism of Section 5.7 applies only to an
> Android Companion.

### 4.3 §5.3 `loopback-unix` (new, OPTIONAL)

> | Property | Requirement |
> |---|---|
> | Binder | Companion endpoint |
> | Dialer | Emacs endpoint |
> | Address family | Unix-domain stream socket, filesystem namespace |
> | Default address | a path in storage private to the Companion's application identity, shown in the pairing UI |
> | Framing | Section 6 Content-Length framing, no header field other than `Content-Length` |
> | Authentication floor | Section 9 mutual HMAC authentication |
> | Confidentiality | none on the wire; interposition requires the socket's filesystem permission |
> | Wake | none |
>
> This profile is `loopback-tcp` with the port replaced by a filesystem
> path, for hosts where both endpoints run under one operating-system user.
> The Companion MUST create the socket with permissions that admit only its
> own user, MUST unlink a stale socket file it owns before binding, and
> MUST NOT bind in a world-writable or shared directory. An abstract-namespace
> socket MUST NOT be used: it has no permission bits and reintroduces the
> squatting exposure this profile exists to remove. Emacs MUST refuse to
> dial a socket file whose owner is not the current user.
>
> A Companion MAY offer this profile alongside `loopback-tcp`; it MUST NOT
> offer it as the only profile on a platform where the Emacs endpoint
> cannot open Unix-domain sockets. Everything in Section 5.2 not concerned
> with IP addressing applies unchanged.

### 4.4 §5.4 `network-tls` (new)

> | Property | Requirement |
> |---|---|
> | Binder | Companion endpoint |
> | Dialer | Emacs endpoint |
> | Address family | TCP over any IP interface the user has approved |
> | Default port | `8766` |
> | Framing | Section 6 Content-Length framing inside a TLS 1.3 or later stream, no header field other than `Content-Length` |
> | Authentication floor | Section 9 mutual HMAC authentication, performed inside the TLS stream, plus certificate pinning below |
> | Confidentiality | TLS |
> | Wake | none |
>
> This is the profile Sections 9.3 and 23.6 require of a deployment whose
> threat model includes an on-path or local interposer, and the only
> profile available when the two endpoints do not share a host. An iOS
> Companion MUST use it; an Android or desktop Companion MAY offer it in
> addition to a loopback profile.
>
> **Certificate.** At pairing time the Companion MUST generate a fresh
> self-signed certificate for the pairing identity, MUST hold its private
> key under the same storage terms Section 9.1 imposes on the token, and
> MUST display the certificate's SubjectPublicKeyInfo SHA-256 fingerprint
> alongside the pairing token, in the same out-of-band channel. Emacs MUST
> record that fingerprint with the pairing, MUST verify it on every
> connection before sending `session.hello`, and MUST close without
> sending any frame when it does not match. Emacs MUST NOT rely on
> certificate-authority validation or hostname matching for this profile.
> Pairing revocation (Section 9.1) MUST destroy the private key.
>
> **Why TLS is not the authentication.** The certificate authenticates the
> Companion to Emacs. The Section 9 handshake, run inside the stream,
> authenticates both endpoints to each other with the pairing token and
> binds the session nonces. Neither replaces the other: an endpoint that
> skips the HMAC handshake because TLS succeeded, or skips pinning because
> the handshake will catch an impostor, is non-conformant. The proof
> domains of Section 9.3 are unchanged.
>
> **Addressing and discovery.** The Companion MUST bind only interfaces the
> user has explicitly approved in the pairing UI, MUST show the listener's
> address and port there, and SHOULD advertise a DNS-SD service instance of
> type `_ebp._tcp` carrying the non-secret pairing ID as a TXT record so
> Emacs can locate it on a link-local network. The advertisement MUST NOT
> carry the token, the fingerprint, or any Section 23.3 sensitive value.
> Emacs MAY be configured with a host and port directly; discovery is an
> aid, never an authority.
>
> **Session scope.** A Companion whose platform suspends listeners in the
> background (iOS) MUST treat every such suspension as a transport close
> under Section 5.6 and MUST NOT report a session as `READY` while it
> cannot service the stream. Durable events accumulate under Section 15
> exactly as when disconnected on any other profile. Emacs's jittered
> redial (Section 5.6) is the reconnection mechanism; there is no wake.
>
> **Limits.** Section 4.5 limits are unchanged. An implementation MAY
> additionally bound the number of concurrent pre-authentication TLS
> handshakes per source address; the Section 5.2 newest-wins eviction rule
> applies after the TLS handshake completes.

### 4.5 §5.5 `loopback-websocket` (new, OPTIONAL)

Adopts the sibling kit's §3 verbatim in substance.

> | Property | Requirement |
> |---|---|
> | Binder | Emacs endpoint |
> | Dialer | Companion endpoint |
> | Address family | IPv4 loopback |
> | Default address | `ws://127.0.0.1:8767` |
> | Framing | one Section 7 message per WebSocket text message (RFC 6455); Section 6 Content-Length framing does not apply |
> | Authentication floor | Section 9 mutual HMAC authentication, plus the Origin rule below |
> | Confidentiality | none; loopback only |
> | Wake | none |
>
> This profile exists for a Companion that runs inside a browser and can
> hold no transport other than WebSocket. Emacs binds; the Companion
> dials. Emacs still speaks first: on each accepted connection Emacs MUST
> send `session.hello` and the Companion MUST answer it exactly as under
> any other profile. Emacs MUST bind only a loopback interface and MUST
> NOT offer this profile off-box.
>
> **Origin.** Emacs MUST validate the WebSocket `Origin` header against a
> user-configured allowlist during the HTTP upgrade and MUST refuse the
> upgrade on mismatch, before any EBP frame is exchanged. An absent Origin
> (a non-browser client) MUST be refused under this profile.
>
> **Message bound.** A message assembled from fragments MUST be bounded by
> `max_frame_bytes`. On overflow the receiver MUST discard the partial
> message, discard continuation frames until the message's final frame,
> SHOULD emit `1400 frame-too-large` on `log.error` under Section 6.2's
> conditions, and MUST keep the connection open. This is the WebSocket
> form of Section 6.2's byte-exact skip.
>
> **Liveness.** WebSocket ping and pong provide transport keepalive. An
> endpoint MAY send pings; every endpoint MUST answer them. No EBP method
> is added for liveness.
>
> **Encoding.** A sender MUST serialize under Section 4.1 and transmit the
> UTF-8 octets as one text message. Binary messages MUST be refused with a
> connection close.
>
> Single authority, supersession, and pre-authentication bounds (Section
> 5.6) apply with the roles as named in the table: Emacs, as binder,
> enforces the pre-authentication connection bound, and the Companion, as
> dialer, applies the jittered redial. `session.superseded` is still sent
> by the Companion, because the Companion owns session authority under
> every profile.

### 4.6 §5.6 Single authority, supersession, and redial (moved from §5.2)

> [The existing §5.2 paragraph beginning "The `android-loopback-tcp` core
> profile permits exactly one paired Emacs authority" through "…uninterrupted
> by session supersession (Sections 21.1, 18.6)" moves here unchanged,
> with the profile name generalized to "every profile in this document."
> The jittered-redial obligation binds whichever endpoint is the dialer
> under the profile in use; the 60-second stand-down after
> `session.superseded` binds Emacs under every profile.]

### 4.7 §5.7 Wake (replaces §5.3)

> A profile MAY define a wake mechanism by which the Companion asks the
> Emacs host to establish a transport. `offline.wake` (Section 22.1) MUST
> NOT be granted unless the Companion's platform defines one below and the
> user has configured its target. Today only one platform does.
>
> **Android, under `loopback-tcp`.** [The existing §5.3 text, verbatim.]
>
> **Every other platform and profile.** No wake mechanism is defined.
> `offline.wake` MUST NOT be granted. In particular: a desktop Companion
> MUST NOT launch or signal an Emacs process, because no inert target that
> satisfies the Android rule's no-arguments requirement is defined for
> desktop hosts; a browser Companion cannot launch anything; and an
> off-box Companion has no host to signal.

### 4.8 Cross-reference edits outside §5

- §1 line 29: "the `android-loopback-tcp` transport profile and its
  framing" becomes "the transport profiles of Section 5 and Section 6
  framing."
- §6.1: "Under the `android-loopback-tcp` profile a sender MUST NOT emit
  any header field other than `Content-Length`" becomes "Under every
  Content-Length-framed profile (Sections 5.2 to 5.4)…". Add: "Section 5.5
  does not use this framing."
- §6.2 receiver-strictness scoping (amendment #34) is unchanged.
- §9.3 last paragraph: "That attacker is outside the threat model of
  `android-loopback-tcp`; a deployment that includes it MUST use a
  stronger transport profile" becomes "…outside the threat model of the
  loopback profiles; a deployment that includes it MUST use Section 5.4."
- §23.6: same substitution, and add one sentence: "Section 5.3 removes the
  port-squatting exposure on hosts that support it; Section 5.4 removes
  the interposition exposure entirely."
- §24.1: "at least one complete transport profile, including Section 6
  framing for `android-loopback-tcp`" becomes "at least one complete
  Section 5 transport profile; a Companion offering a Content-Length
  profile MUST implement Section 6 framing."
- §24.2 (Emacs): add "at least `loopback-tcp`; Sections 5.3 to 5.5 are
  OPTIONAL for the Emacs endpoint."
- §22.1: no change; `offline.wake` row already reads "can attempt the
  `wake` offline policy," which §5.7 now gates per platform.

## 5. Impact per endpoint

### 5.1 `ebp` (this repository)

- `SPEC.md` §5 restructure and the §4.8 cross-references.
- `SPEC-CHANGES.md` row 190. Draft description: **"Transport profiles per
  Companion platform."** §5 generalizes `android-loopback-tcp` to
  `loopback-tcp`, adds OPTIONAL `loopback-unix` (desktop, filesystem-scoped
  local security), `network-tls` (off-box and iOS; the encrypted profile
  §9.3 and §23.6 already required a deployment to have), and OPTIONAL
  `loopback-websocket` (browser Companions; Emacs binds), moves single
  authority and redial into a shared §5.6, and makes wake per-platform with
  Android as the only platform defining a target. Not wire-breaking: no
  frame, member, proof domain, or method changes; the profile is never
  named on the wire. Contract: none. Goldens: `goldens/wire/manifest.json`
  `profile` values renamed.
- `goldens/wire/manifest.json`: two `profile` strings become `loopback-tcp`.
  `validate.py` does not match the profile name anywhere (checked), so the
  manifest strings are the only tooling change.
- `BUILDING-COMPANION.md` is stale independently of this change (it still
  describes NDJSON with `ping`/`pong`, which is EBP 1). Flagged, not fixed
  here.

### 5.2 `ebp.el`

- Docstrings at `ebp.el:59` and `ebp.el:77` reworded: Emacs redials because
  it is the dialer under every profile it implements, not because of
  Android.
- `ebp-connect` gains a `:transport` config key defaulting to
  `loopback-tcp`. `ebp-client--open-transport` branches on it:
  `loopback-unix` passes `:family 'local :service <path>` and checks the
  socket file's owner; `network-tls` passes `:tls-parameters` from
  `gnutls-boot-parameters` with `:verify-error nil` and then verifies the
  pinned fingerprint from `gnutls-peer-status` before `ebp-client-start`,
  closing without a frame on mismatch. A `:certificate-fingerprint` config
  key carries the pin; `ebp-client-forget-pairing` scrubs it with the
  token. Everything else in the file, including the entire reconnect
  lifecycle, is untouched because Emacs is still the dialer.
- `loopback-websocket` cannot live in `ebp.el` under the repository's
  built-ins-only rule. It needs either an optional `ebp-websocket.el` that
  requires the `websocket` package and is exempted from that rule, or a
  home in another repository. Decision 6.2.

### 5.3 `ebp-kmp` and the Companion apps

- `wire` stays transport-free. Add a `Transport` abstraction in
  `commonMain` with two shapes: `Listener` (accept connections; used by
  loopback-tcp, loopback-unix, network-tls) and `Dialer` (open one
  connection; used by loopback-websocket). Both hand the engine a byte
  stream plus a framing selector (Content-Length or message-per-frame).
- Per target: `jvmMain` provides `ServerSocket`, `UnixDomainSocketAddress`
  (JDK 16+), and `SSLServerSocket` with the pairing-scoped self-signed
  certificate; `androidMain` keeps `DeviceBridge.kt`'s loopback listener
  and gains the TLS listener as an option; `appleMain` provides
  `NWListener` with TLS for iOS and macOS; `jsMain`/`wasmJsMain` provide a
  browser `WebSocket` dialer with the message-per-frame framing.
- New targets must be declared in `build.gradle.kts` before any of the
  Apple or browser transports can exist. Today only `jvm` and `android`
  are declared.

## 6. Owner decisions the draft takes a position on

1. **Rename or alias.** The draft renames `android-loopback-tcp` to
   `loopback-tcp` outright. There are no external users, the string is not
   on the wire, and keeping an alias would leave the Android prefix in the
   conformance vocabulary forever. Alternative: keep the old name as a
   deprecated alias for one spec version.
2. **Where the Emacs WebSocket transport lives.** The draft says: an
   optional `ebp-websocket.el` in the `ebp.el` repository, requiring the
   `websocket` package, with `AGENTS.md` amended to exempt that one file
   from the built-ins rule. Alternative: a separate repository, or defer
   the whole browser profile until a browser Companion target is declared.
3. **iOS as TLS server versus a relay.** The draft makes the Companion the
   TLS server because Emacs cannot terminate TLS. The cost is that an iOS
   session exists only while the app is foregrounded, and Emacs must
   locate the phone (DNS-SD or manual). Alternative: a user-run relay on
   the Emacs host that terminates TLS and speaks `loopback-tcp` to Emacs,
   letting the phone dial out. That keeps the phone's background story
   simpler but adds a third process to the trust boundary and to the
   pairing UI. The draft rejects it for the same reason §5.3 rejects
   trampolines: one more authorization-bearing hop.
4. **Ports.** The draft assigns `8766` and `8767` so the three IP-based
   profiles can coexist on one host with distinct defaults. Alternative:
   share `8765` and let the first byte distinguish (TLS ClientHello versus
   `C` of `Content-Length` versus `G` of an HTTP upgrade). Rejected as
   clever; distinct defaults are simpler to document and to firewall.
