# 5½-day soak verification — infra Jenkins (2026-08-30)

Follow-up to [`ROLLOUT-VERIFICATION-2026-08-25.md`](ROLLOUT-VERIFICATION-2026-08-25.md).
That record closed the rollout on 18 hours of traffic. This one asks whether
enforcement *held* once the novelty wore off. Everything here is read-only
measurement; nothing was changed to produce it.

**Result: still clean, and prod coverage is broadening rather than merely holding.**

## Method

Read-only CloudTrail sweep of ops (`685502069032`), **2026-08-25T03:44Z →
2026-08-30T15:13Z** — picking up at the exact instant the previous sweep ended, so
the two records are contiguous with no gap.

**49,252 `AssumeRole` events**, 986 pages, paginated with the boto3 paginator and
**no `--max-items` cap**. That matters: `lookup-events` returns newest-first, so a
cap silently truncates to the *tail* and can hide precisely the misses you are
looking for. The 25 Aug sweep defended against this with 30-minute slices plus a
`NextToken`-absent assertion per slice; full pagination is the simpler guarantee.

Jenkins-originated events are identified by **caller ARN** containing
`ck-ops-jenkins-master-instance-iam-role` — never by source IP, which is a shared
NAT fronting ~25 instances.

## The evidence: an exact accounting, not a count

Counting `jk-` events cannot detect a miss — a job that never engaged the plugin
produces no `jk-` event and no error, so it is invisible to a count. The check has
to close the *total*.

Of the **3,045** events whose caller is the Jenkins instance role:

| | count |
|---|---|
| `jk-*` attributed | **2,917** — 929 distinct build sessions |
| `ck-aws-selfassume-probe` | **106** — every one `errorCode: none`, **0 denials** |
| second hop (build names its own session) | **22** |
| **unattributed** | **0** |

2,917 + 106 + 22 + 0 = **3,045 = 3,045.** The total closes.

**Zero errors from any Jenkins caller.** There are 525 `AccessDenied` in the window,
none with a Jenkins caller — all from `ck-ssm-portal-beta-TaskRole`, a ~15-minute
failure cycle that predates the rollout and is unrelated to it (see below).

## Prod coverage is expanding

**22 distinct prod jobs**, up from 14 at close-out and 3 on 24 Aug:

`azure-insights`, `batchprocessor`, `billdesk`, `ck-drupal-deploy-app`, `ckauto`,
`cloudfront-cache-clear`, `cloudkeeper-drupal-drush-command`, `config-server`, `fe`,
`finops`, `gcp-insights`, `lens`, `lens-ai`, `prod_icon_upload_to_s3`,
`profitability`, `rivon`, `schedulers`, `tuner`, `tuner-event`, `tuner-recom`,
`tunerextension`, `war-report`.

The growth is the point: jobs that simply had not run during the 18-hour window are
now appearing, each one attributed on first sight. Nothing needed to be done to
include them.

## Second hop — unchanged and still cosmetic

22 events across exactly two session names, both known:

| session name | count | origin |
|---|---|---|
| `jenkins-session` | 12 | `qa-virtuoso-resource-creation` |
| `Route53BackupSession` | 10 | the route53 backup/restore jobs |

Both are builds that call `sts assume-role` themselves and name their own session.
An explicit session name always wins — no plugin can override it — but each remains
attributable one join back through its `jk-` caller. Optional repo-side polish is a
single line: `--role-session-name "${CK_AWS_SESSION_NAME:-jenkins-session}"`.

## Unrelated, and still nobody is watching it

525 `AccessDenied` in the window, every one from
`ck-ssm-portal-beta-TaskRole-nIAiq6X8XnJz` on a ~15-minute cycle. This predates the
rollout, is not Jenkins and is not ck-aws — it was found incidentally while sweeping
CloudTrail, and it has now been failing continuously for well over a week. Worth
raising with whoever owns that service.

## POC environment retired

With enforcement verified over 5½ days, the plugin's POC clone `poc-jenkins-2`
(`i-0cdd407bce366be0f`) was terminated on 2026-08-30 along with its volumes, four
security groups, and both POC AMIs and their snapshots.

It had been **stopped since 2026-08-19** — before the install was validated and
before every enforcement step — so the entire rollout happened with it powered off.
It was also on a different AMI from the live controller and 11 days adrift in
config, so it could no longer faithfully reproduce infra anyway.

The teardown note's instruction to "retrieve
`/var/lib/poc-artifacts/ck-aws-2.2-VALIDATED-f5150ba3.jpi` first" was **stale**:
`f5150ba3` is a superseded pre-release. Infra runs `2.2.0-final`, sha256
`f2d3a59e…`, archived outside the repo with a verified matching checksum.

Verified after the fact: infra Jenkins running and healthy behind its target group,
on an unrelated security group, still attributing builds.
