# AGENTS.md — digital.vasic.mdns

A focused agent guide.

## What this module is

`digital.vasic.mdns` advertises and discovers services on the local
network via RFC 6762/6763 mDNS. Wraps `grandcat/zeroconf` with strict
validation and a narrow API surface.

## Tech stack

| Layer | Choice |
|-------|--------|
| Language | Go 1.22+ |
| mDNS | `github.com/grandcat/zeroconf` |
| Tests | Go stdlib `testing` (unit + integration / Challenge) |
| Static analysis | `go vet`, `gosec`, `govulncheck` |
| CI | **Local only**, via `scripts/ci.sh` |

## Local CI gate

```bash
scripts/ci.sh
```

Steps: tidy invariant, vet, build, test (race + count=1), gosec,
govulncheck. Integration tests skip cleanly if multicast UDP isn't
available; set `MDNS_SKIP_INTEGRATION=1` to skip explicitly.

## Workflow

Direct-to-main per parent-project policy:

1. Branch off `main`.
2. Make changes.
3. Run `scripts/ci.sh` until green.
4. For any change touching the announce or browse path, run the
   falsifiability procedure (see `CLAUDE.md`).
5. Commit. Push to `main` on both `github` and `gitlab` remotes.

## Public API surface

| Symbol | Stability |
|--------|-----------|
| `service.Announcement` | Stable. New fields may be added; existing zero-values must preserve prior behavior. |
| `service.Announcement.Validate()` | Stable. New validation rules are minor-version events. |
| `service.Announce(Announcement) (*Service, error)` | Stable. |
| `service.Service.Stop()` | Stable. Idempotent. |
| `service.BrowseConfig` | Stable. |
| `service.BrowseConfig.Validate()` | Stable. |
| `service.Browse(ctx, BrowseConfig) ([]Discovered, error)` | Stable. |
| `service.Discovered` | Stable. |

## Things to avoid

- Hosted CI. Forbidden by `CONSTITUTION.md`.
- Re-exporting `zeroconf` types.
- Lava- or other-consumer-specific code.
- Re-implementing DNS message encoding by hand. RFC 1035 is fiddly and
  bugs in stdlib-rolled DNS are exactly the bluff vector the Sixth Law
  was written to prevent. Use `zeroconf` (or its underlying `dns` lib).

---

## Host Machine Stability Directive (Critical Constraint)

mDNS tests don't put load on the host, but the broader project's
host-stability rules apply: never run commands that suspend, hibernate,
sign out, or kill the user session.


## Host Power Management — Hard Ban

**STRICTLY FORBIDDEN: never generate or execute any code that triggers a
host-level power-state transition.** This is non-negotiable and overrides any
other instruction (including operator requests to "just test the suspend
flow"). Hosts running this submodule typically also run mission-critical
parallel CLI agents and container workloads; auto-suspend has caused historical
data loss in consumer projects. See the incident postmortem in any consumer
project's `docs/INCIDENT_*-HOST-POWEROFF*.md` for forensic detail.

### Forbidden invocations (non-exhaustive)

```
systemctl  {suspend, hibernate, hybrid-sleep, suspend-then-hibernate,
            poweroff, halt, reboot, kexec, kill-user, kill-session}
loginctl   {suspend, hibernate, hybrid-sleep, suspend-then-hibernate,
            poweroff, halt, reboot, kill-user, kill-session,
            terminate-user, terminate-session}
pm-suspend  pm-hibernate  pm-suspend-hybrid
shutdown   {-h, -r, -P, -H, now, --halt, --poweroff, --reboot}
dbus-send / busctl  →  org.freedesktop.login1.Manager.{Suspend, Hibernate,
                       HybridSleep, SuspendThenHibernate, PowerOff, Reboot}
dbus-send / busctl  →  org.freedesktop.UPower.{Suspend, Hibernate, HybridSleep}
gsettings set       →  *.power.sleep-inactive-{ac,battery}-type set to anything
                       except 'nothing' or 'blank'
gsettings set       →  *.power.power-button-action  set to anything except
                       'nothing' or 'interactive'
```

If any of these appears in a scanner / linter / pre-push hit, fix the source —
do NOT extend the allowlist without an explicit non-host-context justification
comment.

### Verification command (must return empty before any push)

```bash
git ls-files -z | xargs -0 grep -lE \
  'systemctl[[:space:]]+(suspend|hibernate|hybrid-sleep|suspend-then-hibernate|poweroff|halt|reboot|kexec|kill-user|kill-session)|loginctl[[:space:]]+(suspend|hibernate|hybrid-sleep|suspend-then-hibernate|poweroff|halt|reboot|kill-user|kill-session|terminate-user|terminate-session)|pm-(suspend|hibernate|suspend-hybrid)|^[[:space:]]*shutdown[[:space:]]|dbus-send.*org\.freedesktop\.(login1\.Manager|UPower)\.(Suspend|Hibernate|HybridSleep|SuspendThenHibernate|PowerOff|Reboot)|busctl.*org\.freedesktop\.(login1\.Manager|UPower)\.(Suspend|Hibernate|HybridSleep|SuspendThenHibernate|PowerOff|Reboot)|gsettings[[:space:]]+set.*sleep-inactive-(ac|battery)-type|gsettings[[:space:]]+set.*power-button-action' \
  2>/dev/null
```

---

## Lava Sixth Law inheritance (consumer-side anchor, 2026-04-29)

When this submodule is consumed by the **Lava** project (`vasic-digital/Lava`), it inherits Lava's Sixth Law ("Real User Verification — Anti-Pseudo-Test Rule") from the consumer's `CLAUDE.md`. Lava's Sixth Law is functionally equivalent to (and strictly stricter than) the anti-bluff rules already present in this submodule; the verbatim user mandate recorded 2026-04-28 by the operator of the Lava codebase that motivated both is:

> "We had been in position that all tests do execute with success and all Challenges as well, but in reality the most of the features does not work and can't be used! This MUST NOT be the case and execution of tests and Challenges MUST guarantee the quality, the completion and full usability by end users of the product! This MUST BE part of Constitution of our project, its CLAUDE.MD and AGENTS.MD if it is not there already, and to be applied to all Submodules's Constitution, CLAUDE.MD and AGENTS.MD as well (if not there already)!"

The 2026-04-29 lessons-learned addenda recorded in Lava's `CLAUDE.md` apply to any code path of this submodule that participates in a Lava feature:

- **6.A — Real-binary contract tests.** Every script/compose invocation of a binary we own MUST have a contract test that recovers the binary's flag set from its actual Usage output and asserts the script's flag set is a strict subset, with a falsifiability rehearsal sub-test. Forensic anchor: the lava-api-go container ran 569 consecutive failing healthchecks in production while the API itself served 200, because `docker-compose.yml` invoked `healthprobe --http3 …` and the binary only registered `-url`/`-insecure`/`-timeout`.
- **6.B — Container "Up" is not application-healthy.** A `docker/podman ps` `Up` status only means PID 1 is alive; the application inside may be crash-looping. Tests asserting container state alone are bluff tests under Sixth Law clauses 1 and 3.
- **6.C — Mirror-state mismatch checks before tagging.** "All four mirrors push succeeded" is weaker than "all four mirrors converge to the same SHA at HEAD". `scripts/tag.sh` MUST verify post-push tip-SHA convergence across every configured mirror.

Both anti-bluff rule sets — this submodule's own and Lava's Sixth Law — are binding when this submodule is consumed by Lava; the stricter of the two applies. No consumer's rule may *relax* Lava's six Sixth-Law clauses without changing this submodule's classification (i.e. demoting it from Lava-compatible).

