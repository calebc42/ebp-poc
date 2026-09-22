# Transport kit: `emacs-unix`, the dial-direction rule, and the Android agent (amendment #191)

**Status: landed as amendment #191 on 2026-09-22; this kit is the provenance
record.** It answers the question the Emacs 32 canvas patch raised — whether
a single-APK Emacs, drawing its own pixels, should invert who dials — and the
question a desktop deployment raises on its own: whether a daemon Emacs
should bind. Section 4 is the normative text as landed; sections 1 to 3 are
the fact base and reasoning; section 5 is the impact on each endpoint;
section 6 records the owner decisions.

Sibling kits: `TRANSPORT-profiles-kit.md` (amendment #190, the four-profile
restructure this builds on) and `WEBSOCKET-transport-kit.md` (the first
Emacs-binds profile).

## 1. What #190 pinned, and what it left unsaid

- §5.1 decouples protocol role from transport role: Emacs speaks first
  regardless of who bound. §5.5 uses that once, for browsers.
- Every other profile has the Companion bind and Emacs dial, and #190's log
  row gives two reasons: the redial machinery in `ebp.el` carries unchanged,
  and Emacs's TLS stack is client-only. Neither is a principle; the first is
  a cost and the second applies to one profile.
- §5.7 (then 5.3) defined one wake mechanism, an Android component start,
  and tied it by name to `loopback-tcp` (the 5.2 wake row and the 5.2
  platforms paragraph), while 5.3 `loopback-unix` carried `Wake: none`. An
  Android Companion offering both profiles would have lost wake under the
  safer one.
- §6.2's receiver-strictness excuse for Emacs rests on a fact — "Emacs dials
  only its authenticated Companion" — that any Emacs-binds profile
  falsifies. §5.5 patched that locally with a narrow sentence naming only the
  §4.1 rejections, not §6.2's own header obligations.

## 2. Facts that constrain the design

Verified against the working trees on 2026-09-22.

**The canvas patch is orthogonal to transport.** Emacs 32's `:type canvas`
image is a writable ARGB32 buffer inside Emacs's own redisplay, refreshed
by `canvas-refresh` from Elisp or a dynamic module. It dissolves the display
rationale for a separate renderer app; it does nothing for the lifecycle
rationale, because a canvas dies with the Emacs process. The `jetpacs-native`
spike says the same: the process-lifecycle rationale moves from an app
boundary to a process boundary inside one APK. The Companion therefore stays
a separate process (`android:process=":agent"`), and Emacs, still the
volatile end on Android, still dials it.

**Emacs can bind a filesystem Unix socket.** `make-network-process :server t
:family 'local` is how `server.el` serves `emacsclient`, and
`server-ensure-safe-dir` is the precedent for the directory rule: created
0700, no symlink, owned by the user, no group or other bits.

**The Companion's platforms.** Android exposes
`android.net.LocalServerSocket(FileDescriptor)`, `LocalSocket.bind` with
`LocalSocketAddress.Namespace.FILESYSTEM`, and `LocalSocket.getPeerCredentials`
(all public in `android-37.0`); the one-argument `LocalServerSocket(String)`
is the abstract namespace §5.3 forbids. The JVM has Unix-domain channels from
JDK 16 (`UnixDomainSocketAddress`), absent on Android. `Application.getProcessName()`
is API 28 and the port's minimum is 30.

**The single-APK process split as it stands.** The fork's `EmacsApplication.onCreate`
calls `EmacsNative.setupSystemThread()` unconditionally, so the `:agent`
process loads `libemacs.so` today despite the manifest comment; a
process-name guard is part of the implementation, not the spec.
`EmacsService.startEmacsService(Context)` is the single entry that starts
Emacs (a foreground service on Android O+), and `EmacsService.onBind`
returns null. A bound service is the only same-package signal that can carry
no extras at all: a component start carries an `Intent` whose extras are
data the launcher activity reads, which is exactly the exposure §5.8's
informative note describes.

**The in-process M1 host is a harness, not a barrier.** `EmacsEbpHost`
runs the Java endpoint on an executor in the main Emacs process; its plan
says it proves recovery, not presentation independence. It stays as is.

## 3. The design in one paragraph

State the rule: the endpoint with the shorter expected lifetime dials. A dial
at startup costs nothing; a redial loop against a dead peer spends the budget
of the durable endpoint, which exists to conserve it. On Android the Emacs
process is the volatile end, so Emacs dials under `loopback-tcp` and
`loopback-unix` and the wake ends in an Emacs dial. A browser page cannot
listen, so it dials. A daemon Emacs outlives its desktop frontends, so they
dial: that is `emacs-unix`, Emacs binding a filesystem Unix socket under the
`server.el` directory rules, with the strictness the binding endpoint owes.
`network-tls` is the one direction fixed by mechanism. The single-APK
deployment keeps the current direction and gains two things the spec already
had room for: `loopback-unix` on Android (same UID, app-private directory,
squat-proof) and a second Android wake mechanism, a same-package bound
service, inert by construction.

## 4. Normative text as landed

See SPEC.md: §5.1 (the table row and the informative aside), §5.6 in full,
the Emacs-binds paragraph of §5.7, the retitled Android component-start
mechanism and the new bound-service mechanism in §5.8, and the scoping
edits in §4.5, §6.1, §6.2, §9.3, §24.2, and §24.6. The text is not repeated
here; the amendment row in `SPEC-CHANGES.md` summarizes it.

## 5. Impact per endpoint

### 5.1 `ebp`
- `SPEC.md` as above; `SPEC-CHANGES.md` row 191; this kit; the README
  slop-docs pointer. No contract change. No golden change: the manifest's
  `profile` field is informative (validate.py never reads it) and every wire
  fixture is profile-neutral Content-Length bytes, so an `emacs-unix`
  duplicate would claim coverage it does not add.

### 5.2 `ebp.el`
- A new optional feature `ebp-unix.el`, built-ins only: the listener via
  `make-network-process :server t :family 'local`, directory safety via
  `server-ensure-safe-dir`, and a strict manual-filter connection that runs
  the reference decoder rather than jsonrpc.el's tolerant reader, because
  §5.6 makes §6.2 REQUIRED of a binding Emacs.
- The pre-authentication pool and the strict receive path move out of
  `ebp-websocket.el` into `ebp.el` as `ebp--binder-*`, since §5.7 is
  profile-independent and both binders need them.
- The `loopback-unix` dialer needs nothing: it already refuses a foreign
  owner and retries a missing socket with backoff.

### 5.3 `ebp-java`
- A transport-neutral accept path (the engine already consumes bytes and
  writes frames through a sink) and the §5.7 pre-authentication pool, which
  the reference listener lacked: bound, 10-second deadline, newest-wins
  eviction.
- Android: a `LocalServerSocket` listener for `loopback-unix` with the
  peer-UID check on accept. JVM: a JDK-16 tier, never vendored into the
  Emacs tree, with a `loopback-unix` listener and an `emacs-unix` dialer.

### 5.4 The Emacs Android fork
- An `:agent` service hosting the Companion over `loopback-unix` in the
  app-private directory, with its own pairing identity and store; a
  non-exported wake service in the Emacs process whose binder exposes
  nothing; a process-name guard keeping `libemacs.so` out of the agent.

## 6. Owner decisions

1. **Renumber or append.** Inserted as §5.6 and renumbered (5.7
   supersession, 5.8 wake). Four places cite the profiles as the contiguous
   range "5.2 through 5.N"; appending after two non-profile sections would
   have broken that. The #188 log row keeps its pre-#190 section numbers:
   log rows are dated history and cite numbering as of their date.
2. **Name.** `emacs-unix`, naming the binder, as `loopback-unix` names its
   medium. Alternatives considered: `daemon-unix` (names a deployment),
   `loopback-unix-reverse` (encodes a direction the spec says is not a role).
3. **Bound service or component start for the same-package case.** Bound
   service. It cannot carry extras, the component is not exported, and the
   binding props the host's importance while it lasts. The component start
   stays defined and available to the same Companion, because the platform
   may refuse the host's foreground-service start from a background bind;
   that refusal is not event rejection.
4. **Stale sockets.** Emacs unlinks a stale socket it owns and never one that
   accepts connections: a connectable socket is another Emacs authority on
   that path, and single authority (§5.7) says the later bind fails closed
   rather than stealing the path.
5. **Two Companions on one device.** The M1 in-process host and the agent are
   two Companion endpoints with two pairing identities. Nothing forbids that;
   the Lisp side must never point one client at the other's credentials.
6. **Agent service class.** A plain started service, promoted to a foreground
   service only around a wake attempt. A persistent foreground agent is a
   product decision, not a protocol one, and invariant 8 licenses the
   listener's absence.
