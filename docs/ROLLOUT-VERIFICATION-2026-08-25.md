# Rollout verification — all jobs, infra Jenkins (2026-08-25)

The close-out record for the ck-aws 2.2.0 rollout on infra Jenkins
(`i-0007d48a74436e085`). Everything here is read-only measurement; nothing was
changed to produce it.

**Result: enforcing on all 806 jobs, zero unattributed Jenkins-originated calls,
zero plugin-caused build failures.**

## Timeline

| When | What |
|---|---|
| 2026-08-20 15:09 IST | Estate-wide observe-only opened, all 806 jobs |
| 2026-08-21 ~11:40 IST | Non-prod enforcing — `(dev\|qa\|uat)[0-9]*/.*\|ckaws-canary-.*` |
| 2026-08-24 14:58 IST (09:28Z) | **`jobNamePattern` blanked — every job, prod included** |
| 2026-08-25 | Full-traffic verification below |

## The evidence: an exact accounting, not a count

Counting `jk-` events cannot detect a miss — a job that never engaged the plugin
produces no `jk-` event and no error. The check has to close the *total*.

Read-only CloudTrail sweep of ops (`685502069032`), 2026-08-24T09:28Z →
2026-08-25T03:44Z (18h), **6,715 AssumeRole events**, 30-minute slices with
recursive sub-slicing on truncation.

Of the **279** AssumeRole events whose *caller* is
`ck-ops-jenkins-master-instance-iam-role`:

| bucket | count |
|---|---|
| `jk-*` — attributed builds | **253** (141 distinct builds) |
| `ck-aws-selfassume-probe` | **14** — all OK, **0 denials** |
| second hop (tool names its own session) | **12** — every one with a `jk-` caller |
| **unattributed** | **0** |

The remaining 6,436 events in the account are not Jenkins: 4,860
`AWSServiceRoleForECS` autoscaling, 162 `ck-ssm-portal-beta-TaskRole`, 156
`PortalProdTaskRole` → reconcile-system, 100 `efk-instance-role` → logstash, 3
CKPrism admin, 3 route53-backup. Bucketed by calling principal, not assumed.

**Prod coverage is real: 14 distinct prod jobs** — finops (×7), billdesk (×4),
azure-insights, lens, fe, notifications, gcp-insights, schedulers, marketplace,
ckauto, lens-ai, auditor, tuner-event, tuner-mcp.

## Second hop — transitivity confirmed empirically

`CLAUDE.md` argues attribution is transitive: a tool that assumes a role itself
picks its own session name, but CloudTrail records the *caller*, so one join
recovers the build. That is now measured rather than reasoned. All 12 second-hop
events carry a `jk-` caller:

| generated session | caller |
|---|---|
| `cleanup-275595855473` | `jk-ck-iam-role-cleaner-delete-8`, `-restore-13…18` |
| `jenkins-session` | `jk-qa-virtuoso-resource-creation-484` |
| `Route53BackupSession` | `jk-shared-route53-backup-…-757`, `jk-53-backup-restore-…-762` |

So the second-hop item is **cosmetic, not a coverage gap**. Repo-side
`session_name` / `CK_AWS_SESSION_NAME` remains optional polish.

## Long job names are handled

`jk-53-backup-restore-ck-dns-management-route53-backup-471f9e-762` is exactly 64
characters — the STS session-name limit. The plugin truncates and appends a hash
rather than failing. Observed in production, not just in tests.

## No plugin-caused failures — four independent checks

1. **0 probe denials in 14 runs.** The self-assume probe gates the `[default]`
   append; a denial is the one way the plugin could break a bare `aws` call.
2. **0 AssumeRole errors from any Jenkins caller.**
3. **No trust policy gates on `sts:RoleSessionName`** — checked
   `terraform-assume-role` in both ops and nonprod plus the Jenkins instance role.
   The rename therefore cannot itself cause a denial. This closes the only path by
   which the plugin could break a build *after* it has credentials.
4. **Named-profile path proven live.** `qa1/cloudkeeper-analytics` runs
   `aws ssm get-parameter --profile non_prod` clean end to end under the plugin —
   ECR login, layer upload, `PutImage`, ~42 × `GetParameter`+`Decrypt`,
   `RegisterTaskDefinition`, `UpdateService`, all with no error code. That job was
   failing at the time for an unrelated reason (Superset gunicorn workers hitting
   `WORKER TIMEOUT` then `SIGKILL` at a 920 MB soft memory reservation with no hard
   limit, so the ALB health check fails and ECS redeploys).

## Jenkins log recorder — clean, but a weak detector

Export of 2026-08-25: 537 PDF pages, but only **256 log entries** — the rest is
stack traces. **Zero `prepareOnce` failures, zero additions-only declines.** Every
entry is one benign signature, and `IOException` is the only exception class in the
whole document (256/256):

```
WARNING ManagedAwsContext guarded
ck-aws: contributing nothing; this build authenticates exactly as it would have without the plugin
java.io.IOException: no FlowNode in this context
```

Coverage straddles the blanking — oldest entry 24 Aug 11:56, newest 25 Aug 04:35,
so **223 of the 256 are in all-jobs scope**.

**256 is not a coincidence: it is Jenkins' `RingBufferLogHandler` default cap.**
The 2026-08-21 read also returned exactly 256. The recorder is saturated and
rolling — everything before 24 Aug 11:56 has been discarded, and at the observed
peak (63 entries in the 20:00 hour) the buffer turns over in **~4 hours**. It can
therefore prove "nothing bad in the newest 256 warnings", never "no decline ever
happened".

**This promotes the `isBenignRace()` one-liner from cosmetic to a detector fix.**
Adding `"no FlowNode in this context"` beside
`"cannot start writing logs to a finished node"` drops this case to FINE and leaves
the buffer holding only real signal. It needs a plugin install and therefore a
restart — see `docs/INCIDENT_2026-08-18_JENKINS_OUTAGE.md` and the systemd 90s
start timeout. If a restart happens anyway, also set
`-Dhudson.util.RingBufferLogHandler.defaultSize=2000`.

**Why "contributing nothing" is not an attribution gap.** It is a Jenkinsfile
reading `env.X` inside a Groovy closure with no FlowNode; that `contribute()` call
is a redundant no-op while the real decoration happens in `prepareOnce`, which
covers both Pipeline and Freestyle. The two detectors corroborate: the recorder
says "contributed nothing" 256 times over exactly the window in which CloudTrail
accounted for 279/279 with zero unattributed. Both can only be true if that path is
redundant.

## Method notes — how to repeat this

- **Never trust one `lookup-events` call.** It returns newest-first, so a
  `--max-items` cap silently truncates to the *tail*: a busy account can return the
  last 90 seconds of a 20-minute window and look clean. Slice the window and assert
  `NextToken` is absent per slice.
- **To find Jenkins-originated misses, filter on caller ARN** containing
  `ck-ops-jenkins-master-instance-iam-role` — not source IP. The agents' public IP
  `34.236.189.62` is a shared NAT fronting ~25 instances and nearly produced a false
  finding on 2026-08-21.
- **`Username` in `lookup-events` matches the *caller* of an event**, not the
  session being created. Looking up a name seen in `requestParameters.roleSessionName`
  returns nothing.
- **"Out of scope" vs "silently missed":** a node the plugin prepared on emits
  `ck-aws-selfassume-probe`. On a freshly launched agent there is no memoized probe,
  so *no probe since launch* proves the plugin never engaged.

## Unrelated, but observed

~90 `AccessDenied` events in the window, every one from
`ck-ssm-portal-beta-TaskRole-nIAiq6X8XnJz` on a ~15-minute cycle, predating the
blanking. Not Jenkins, not the plugin — but it has been failing steadily for 18h+
and appears unmonitored.
