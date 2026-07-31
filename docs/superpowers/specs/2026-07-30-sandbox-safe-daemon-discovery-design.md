# Sandbox-Safe Daemon Discovery Design

## Problem

The roborev CLI discovers a running daemon by probing the endpoint published in
the daemon runtime file. A sandbox can deny access to TCP loopback even though
the runtime file and daemon process are visible. The probe then looks identical
to an unavailable daemon. Current lifecycle handling may restart that known
daemon or start another daemon, while `roborev status` reduces the underlying
error to `Daemon: not running`.

Daemon-backed agent skills are especially exposed because they invoke the CLI
inside Codex or Claude command sandboxes. Generic sandbox retry guidance is not
reliable here: roborev converts the connection failure into a successful status
command with a misleading semantic result.

## Goals

- Never restart, replace, kill, or auto-start a daemon because a probe was
  denied by local permissions.
- Give Unix clients a private Unix-domain-socket fallback when TCP loopback is
  blocked.
- Explain the sandbox failure clearly in CLI output and installed agent skills.
- Preserve the configured endpoint as the daemon's primary public endpoint.
- Preserve Windows and systemd socket-activation behavior.

## Non-goals

- Detect whether a command was invoked by an agent skill.
- Test or configure Codex, Claude, or operating-system sandbox implementations.
- Add a second TCP listener when a user explicitly configured a Unix-only
  daemon.
- Change manual review deduplication or refine's completed-review reuse policy.

## Transport Design

When the configured daemon endpoint is TCP and the platform supports Unix
sockets, a normally started daemon will attempt to bind the existing default
private Unix socket in addition to TCP. The TCP endpoint remains primary. The
socket directory and file retain their existing owner-only permissions (`0700`
and `0600`).

The auxiliary socket is best-effort. Failure to create its directory, validate
permissions, remove a stale socket, bind, or set permissions logs a warning and
does not prevent TCP service from starting. Shutdown closes both listeners and
best-effort removes the auxiliary socket.

An explicitly configured Unix endpoint remains Unix-only. Systemd socket
activation continues using only the activated listener. Windows remains
TCP-only.

The same HTTP handler and daemon state serve both listeners. Worker startup,
runtime publication, health checks, and shutdown remain owned by one daemon
process.

## Runtime Metadata and Discovery

The runtime record retains its current `network` and `address` fields for the
primary endpoint. When the auxiliary socket is available, its network and
address are stored as optional runtime metadata. This uses the existing
`metadata` field from `go.kenn.io/kit/daemon.RuntimeRecord`, so older clients
continue reading the primary endpoint and new clients can discover the
alternate without a parallel runtime-file format.

New clients construct an ordered endpoint list from a runtime record: primary
first, alternate second. Discovery probes each endpoint and accepts the first
valid roborev response. The selected endpoint is returned to callers, so every
subsequent command in that invocation uses the transport that actually worked.

Alternate metadata is ignored unless both its network and address form a valid
local endpoint. TCP endpoints must continue to satisfy the loopback-only
policy.

## Permission-Denied Behavior

Probe failures wrapping `EACCES` or `EPERM` are classified as access denied.
Discovery still attempts the alternate endpoint. If no endpoint succeeds and
at least one probe was denied, it returns a distinct access-denied error rather
than `not found`.

Lifecycle code propagates this error. It must not invoke zombie cleanup,
restart, or auto-start paths in response. Zombie cleanup also treats an
access-denied probe as indeterminate and leaves the process, runtime record, and
socket untouched.

`roborev status` reports that daemon status is unavailable and includes an
actionable suggestion: a command sandbox may be blocking local loopback or
Unix-socket access, so retry outside the sandbox or permit access to the local
daemon endpoint. It does not suggest starting the daemon for this error.

Other failures retain existing behavior. Connection refusal, missing runtime
records, dead PIDs, invalid daemon identity, and version mismatch are not
permission denials.

## Agent Skill Guidance

Every shipped Codex and Claude skill that invokes daemon-backed roborev
commands states that command sandboxes can block daemon access. The guidance
uses each agent's native tool vocabulary:

- Codex retries the roborev command with elevated sandbox permissions.
- Claude retries the Bash call with `dangerouslyDisableSandbox: true`.

The guidance says not to start or restart the daemon when status is
inaccessible or permission denied. Non-roborev commands remain sandboxed.
Claude skills derived from Codex sources use the existing derivation mechanism
for agent-specific wording; independently maintained Claude review skills
receive the same behavioral guidance directly.

## Testing

Focused tests will cover behavior owned by roborev:

- runtime records expose the primary and valid alternate endpoints in order;
- discovery succeeds through the alternate after a primary probe denial;
- an all-endpoints access denial remains distinct from daemon absence;
- ensure/status paths do not restart or auto-start on access denial;
- zombie cleanup leaves access-denied runtimes untouched;
- a daemon is reachable over both listeners when auxiliary setup succeeds;
- auxiliary setup failure is warning-only and TCP remains reachable;
- rendered Codex and Claude skills contain their respective sandbox guidance.

Tests will use temporary directories, loopback listeners, Unix sockets where
supported, injected probe seams, and the test agent. They will not assert
stdlib networking behavior, inspect sandbox configuration, or touch the live
roborev data directory or daemon.

## Documentation

User-facing daemon configuration and command documentation will explain that
TCP daemons expose a best-effort private Unix fallback on supported platforms,
and that permission-denied status output is intentionally non-destructive.
