# Native hook deferral prototype

This branch accompanies the Juju `4.1-defer-prototype` implementation. It requires
a Juju agent providing `hook-defer` and `JUJU_HOOK_REISSUED=1` on redispatch.

From a hook handler:

```python
def _on_upgrade_charm(self, event: ops.UpgradeCharmEvent) -> None:
    if not self._handover_ready():
        self._publish_handover_progress()
        self.framework.defer_hook()
        return
    self._upgrade_replica()
```

The helper methods above represent the charm's own coordination and upgrade
logic. Every prerequisite must be achievable by this repeated hook, another
unit, or an independent workload process: other local hooks cannot interleave.

`defer_hook()` runs the hook command and returns normally. It does not terminate
the handler or flush writes immediately. Other observers still run; they can
check the read-only `framework.hook_deferred` property. Normal status evaluation
and framework commit finish before Juju commits the successful invocation and
reissues the hook. Each invocation starts with `hook_deferred` false.

On redispatch, Ops leaves its saved deferred-event queue untouched and emits the
current hook with fresh event arguments. Duplicate-event suppression is disabled
for that invocation so saved notices cannot swallow the reissued hook or its
custom events. Saved events become eligible on the next normal invocation.

The framework rejects native deferral from actions, restricted hooks, historical
event replays (including their synchronous custom events), and outside a current
hook handler. It also rejects mixing `event.defer()` and `defer_hook()` while
handling the current hook, including across observers. Synchronous custom events
emitted by the current hook may request native deferral of that enclosing hook.

`ops.hookcmds.hook_defer()` provides the low-level command wrapper. It bypasses
framework tracking and guards, so Ops charms should use `framework.defer_hook()`.
`ops.JujuContext.hook_reissued` exposes the marker for alternative dispatchers.
Missing commands and rejected requests raise errors; there is no fallback to
ordinary Ops deferral. Juju rejects combining native deferral with reboot.

This is a prototype: no pacing, retry budget, Scenario/Harness simulation, or
live PostgreSQL validation is included. No new tests are added. A later
production design should extend the existing deferral guidance in
`docs/explanation/defer-guidance.md` and cover failure and restart behaviour.

Verification: `tox -e format` and `tox -e lint` pass. The existing
`tox -e unit -- -n 4` suite passes: 2,445 tests plus 13 release-script tests,
with 56 skipped. No live Juju or PostgreSQL exercise has been run.
