# Project: CK AWS Plugin

## What this is

A Jenkins plugin that centralizes **AWS identity** for Jenkins builds. It decides
which AWS role a build is allowed to be, under a deterministic,
build-attributable STS session name (`jk-<job>-<build>`), and delivers that
decision to the build through standard AWS configuration — so that every AWS tool
the build already runs picks it up unchanged.

It does **not** run AWS commands. It does not know what a deployment is, what
ECS is, or that CloudKeeper exists. Anything that consumes AWS credentials —
the AWS CLI, boto3, Terraform, Docker, a future SDK — consumes them the same way
any AWS tool anywhere does.

Originally a proof of concept (M0–M5, complete and validated against real AWS).
As of the M6 architecture review it has an agreed long-term direction, recorded
below. **This document is the authoritative architecture document for the
project.**

## Status

| Milestone | State |
|---|---|
| M0 — plugin scaffold | Complete |
| M1 — auth core (AssumeRole + `jk-` session naming, unit-tested) | Complete |
| M2 — first pipeline step wired to the auth core | Complete (as `ckAwsAssumeRole`) |
| M3 — real pipeline on local Jenkins, real AWS, CloudTrail verified | Complete |
| M4 — live AWS validation of the AssumeRole flow | Complete |
| M5 — production packaging, installed on Infra Jenkins | Complete |
| M6 — layered architecture: block-scoped auth, JCasC mapping, agent-side execution | Complete |
| M7 — deployment-library integration (`cln-deployment-scripts` @ `ck-aws-plugin`) | Complete |
| M7v — validation: local, STS, CloudTrail, Infra Jenkins, Backend UAT deployment | Complete |
| M8 — optional `ckAws.run([...])` convenience executor | Planned, optional |
| ~~M9 — RunListener default injection~~ | **Superseded by M11** |
| M10 — IAM trust-policy enforcement | Future, outside Jenkins |
| M11 — Managed Authentication (generated config from the Jenkins mapping) | Implemented and locally validated — **insufficient for the M12 requirement**; retained as the override layer |
| M12 — Managed Authentication: decorate the node's own configuration | Implemented, validated in production on two unmodified pipelines |
| ~~M12i — production incident investigation (Gradle 403)~~ | **Closed: the plugin was not the cause.** See below |
| v2.0 — unprofiled attribution, output verification, runtime controls | Superseded by 2.1 before wide rollout |
| v2.1 — Freestyle coverage, trailing-blank fix, no-role profiles | Implemented, 201 tests — was on infra Jenkins (switch off) until **replaced by 2.2.0** |
| **v2.2 — context shadowing, workspace anchoring, stale memo, parallel race, observe-only, additions-only environment invariant, per-node unprofiled attribution** | Implemented, 220 tests, `ck-aws 2.2` (`sha256 f5150ba3…`). Installed and validated on the POC clone. Superseded by 2.3 |
| v2.3 — static unprofiled ARN removed from the form | Implemented. **Installed on the POC clone, so the number is spent** |
| **v2.2.0 — THE INFRA RELEASE** | **239 tests, five adversarial review passes, 14/14 canaries. `sha256 f2d3a59e…`** |
| **Infra rollout — all 806 jobs** | **COMPLETE 2026-08-24, verified under full traffic 2026-08-25.** `jobNamePattern` blank, `observeOnly` false. 279/279 Jenkins-originated AssumeRole events accounted for, **zero unattributed**; 14 distinct prod jobs; zero plugin-caused failures. See `docs/ROLLOUT-VERIFICATION-2026-08-25.md` |
| **5½-day soak — 2026-08-30** | **STILL CLEAN.** 49,252 AssumeRole events swept; of 3,045 with a Jenkins caller: 2,917 `jk-*` (929 builds) + 106 probes (0 denials) + 22 second-hop + **0 unattributed** = 3,045. Zero Jenkins-caller errors. **22 distinct prod jobs**, up from 14. See `docs/SOAK-VERIFICATION-2026-08-30.md` |

**Versioning: `major.minor.patch`, and versions track INSTALLATIONS, not builds.**

- **major** — breaking change to the configuration contract or to what the plugin guarantees
  about a build. Removing a form entry is *not* major while the property still loads from XML.
- **minor** — new capability, or a change in what gets attributed.
- **patch** — defect fix, or a UI/doc change with no behaviour change.

The number must change before an artifact is *installed* on a controller, so that "the
controller says 2.2.0" answers "which 2.2.0" unambiguously. A number that has been installed is
**spent forever** and must never be reused. A number only ever built locally is **not** spent.

**Numbering.** POC iterations run on the **2.1.x** line — 2.1.1, 2.1.2, … — patches on top of
what infra already runs, one number per POC install. The **infra** install claims **2.2.0**, and
that number is never spent on the clone, so "the controller says 2.2.0" can only mean the infra
release. A plain `mvn verify` yields `2.1.1-SNAPSHOT (private-<hash>-<user>)`, deliberately not
installable; a release needs `mvn -Dchangelist= clean verify`.

Applying that rule, as of 2026-08-17:

| Number | Status |
|---|---|
| **2.1** | Was on infra Jenkins, master switch off, until replaced by 2.2.0. Spent |
| **2.2** (`f5150ba3…`) | Test install on the POC clone. The last build validated against real jobs. Spent |
| **2.3** (`bc4d59e1…`) | Test install on the POC clone. Spent |
| 2.4 | Built, never installed, superseded by the code-review fixes. **Not** spent |
| **2.1.1** | Current POC line, carrying every code-review fix |
| 2.2.0 | **Reserved for the infra install** — never to be used on the clone |

Several *earlier* artifacts also called themselves 2.2, 2.3 or 2.4 during the August 2026 defect
work and their hashes circulated. **All of those are void** — the first still contained the
`DynamicContext` shadowing defect. Superseded hashes are listed in MEMORY.md, Session 23.

Build the release at install time with `mvn -Dchangelist= clean verify` and record its hash then.

This rule replaces the older "raise `<revision>` before any artifact leaves the build machine",
which produced a version inflation of three unused numbers. The problem it was written for is
still real — during the v2.0 rollout three different artifacts all reported `Plugin-Version: 2.0`,
making "the controller says 2.0" meaningless — but the fix is to pin the number to the
*installation*, not to every `mvn package`.

---

# ⚠️ PRE-INSTALL CHECKLIST — read before touching infra Jenkins

Infra Jenkins is installed to **exactly once**. Rewritten 2026-08-18 after the release was
cut; every item below is something that was learned the hard way, not a precaution.

### 1. Install this exact binary — do not rebuild it

```
file    : /home/radhika/workspace/poc-jenkins-setup/artifacts/ck-aws-2.2.0-final.hpi
version : 2.2.0
build   : 1bf157ee74259afe5ff28734c401357fcfa91d06
sha256  : f2d3a59eb808ccf4ffb0a9166f21eef43edf9d95b0b6b6ce72691ec46dbbbaa6
```

`.hpi` jars embed build timestamps, so **`mvn clean verify` on unchanged source produces a
different `sha256` every time**, and a rebuild is therefore a *different artifact* that
nothing has been run against. Three earlier 2.2.0 builds exist in `artifacts/` carrying the
same version number and different content, so verify the manifest, not the filename:

```
unzip -p <file>.hpi META-INF/MANIFEST.MF | grep -iE '^Plugin-Version|^Implementation-Build'
```

`Implementation-Build` must read `1bf157e…` and must match the commit you believe you are
shipping. It did not, once: the artifact was built before the last review fixes were
committed, and only a manifest check caught it. Releases need
`mvn -Dchangelist= clean verify` — a plain `mvn verify` yields `2.2.0-SNAPSHOT (private-…)`,
deliberately not installable.

**Precedent — what is on infra right now cannot be identified.** `plugins/ck-aws.jpi` says
version 2.1 and `plugins/ck-aws.bak` says 2.0, and *both* record
`Implementation-Build 2288bd59` — a docs commit from 05 Aug 12:24. The next commit after it is
`1e60d07`, "v2.1 — Freestyle coverage, unprofiled attribution", made 08 Aug at **21:50**. The
2.1 artifact was built at 16:23 that day, five hours earlier. So it was built from a working
tree carrying three days of uncommitted work, and the hash it recorded — honestly — is the last
commit, which is all `Implementation-Build` can see.

The version numbers were no help either: `<revision>` in `pom.xml` read `1.0` at that commit and
never carried 2.0 or 2.1 at all. Both labels were passed as `-Drevision=` on the command line.

Two consequences. The rule: **commit first, build second, and check the manifest against the
commit you meant to ship** — a stamped hash proves nothing about uncommitted edits. And the
reassurance: replacing an unidentifiable binary with `1bf157e`, whose version *is* committed
(`bcdbdd5`) and whose manifest was verified against HEAD, is a strict improvement in
traceability regardless of what else the upgrade does.

**Evidence behind this binary:** 239 unit tests, five adversarial review passes, 14/14
canaries re-run on this artifact after the rebuild, and real jobs of every type (see the
coverage table in MEMORY.md). `poc-jenkins-2` is running it now.

### 2. Infra's pre-install state — MEASURED 2026-08-18, not assumed

Read-only survey of `i-0924a915a1c76f33e` (`jenkins-17`) over SSM with `ops-admin`.

`$JENKINS_HOME/io.github.rads4.ckaws.config.CkAwsGlobalConfiguration.xml`, 334 bytes,
last written 2026-08-12:

```xml
<io.github.rads4.ckaws.config.CkAwsGlobalConfiguration plugin="ck-aws@2.1">
  <profiles/>
  <managedAuthentication>false</managedAuthentication>
  <diagnostics>false</diagnostics>
  <credentialSource>Ec2InstanceMetadata</credentialSource>
</io...>
```

**The master switch is off, confirmed.** And there is no `<observeOnly>` element — that field
arrived in `59fc321`, after 2.1 — so the initialiser survives unmarshalling and infra lands on
the new default `true`. Both fields therefore land in the safe position without anyone touching
them. `<jobNamePattern>` and `<attributeUnprofiledAsNodeRole>` are absent too, so they come up
blank and off. **The upgrade is inert: nothing changes for any build until someone ticks
*Managed authentication* in the UI.**

| Compatibility | Required | Infra has | |
|---|---|---|---|
| Jenkins | 2.479.2 | 2.479.2 | exact |
| Java | 17 | OpenJDK 17.0.19 | ok |
| `workflow-step-api` | ≥ `700.v6e45cb_a_5a_a_21` | `710.v3e456cc85233` | ok |

2.2.0 declares **the same single dependency as the 2.1 already loading successfully**, so the
upgrade adds no new plugin surface. The diff from infra's installed build (`2288bd59`) to the
release is additive — 2853 insertions, 67 deletions, no file removed — so **no step or symbol
that a Jenkinsfile could reference has been withdrawn**. Independently: zero of the 808 job
configs reference `ckAwsWithProfile` or `ckAwsAssumeRole`.

Nothing re-applies configuration at boot: **no `init.groovy.d`, no JCasC, no backup plugin.**
Disk has 85 GB free. `plugins/ck-aws.bak` is version 2.0 built from commit `2288bd59` — the
*same commit* as the installed 2.1, which is another reason to trust `Implementation-Build` over
version numbers.

### 3. Settings to apply at install time

**Seven fields.** If you see *AWS profiles* or *Apply on nodes labelled*, or a text box under
*Attribute unprofiled calls as*, you are looking at an older build.

| Field | Value | Why |
|---|---|---|
| *Managed authentication* | **off** (the default) | Install and restart with it off; turn on afterwards with no restart. **Observe-only does nothing until this is on** — the master switch is checked first on both the Pipeline and Freestyle paths |
| *Apply to jobs matching* | blank | Observe-only makes full scope safe |
| *Except jobs matching* | blank | Reserved as the incident switch |
| *Attribute unprofiled calls as the node's own instance role* | **ticked** | This is what audits ~98% of calls |
| *Agent base identity* | `Ec2InstanceMetadata` | Infra's agents are EC2 |
| *Observe only* | **ticked — the shipped default** | So the moment someone turns the master switch on, the safe mode is already selected rather than every in-scope build changing at once. Enforcing is then a second, deliberate click |
| *Diagnostics* | ticked | Turn off once the rollout is settled |

### 4. Rollout order, and the one restart

Install with the master switch **off** → restart (the only restart) → switch **on** with
observe-only → read a day of console evidence → untick observe-only.

The restart is the only irreversible-feeling step and the only one that touches running work:
it interrupts in-flight builds and disconnects agents. Everything after it is a checkbox.

**Rollback:** put the 2.1 `.jpi` back and restart. Back up both files first —
`plugins/ck-aws.jpi` and the config XML above. Once running, rollback is usually cheaper than
that: unticking *Managed authentication* takes effect without a restart.

### 4b. Pick the window — restart risk, measured

Jenkins was last restarted **2026-08-11 06:27 UTC**, seven days ago, so a restart is a normal
operation on this controller and not a once-a-year event.

What a restart does to work in flight:

- **Pipeline builds resume.** `workflow-durable-task-step 1452` persists step state; a build
  mid-`sh` picks up where it left off.
- **Freestyle builds are killed.** 39 job configs are `<project>` (Freestyle) and they do not
  resume. This is the only thing a restart actually destroys.
- **Queued items survive** — `queue.xml` is persisted and replayed.

Build finish times per UTC hour over 14 days, from the build records themselves:

```
  07h  50 |####################################
  10h  40 |#############################
  09h  39 |############################
  11h  30 |######################
  06h  29 |#####################
  05h  19 |##############
  08h  18 |#############
  13h  15 |###########
  12h  12 |#########
  14h   8 |######
  04h   7 |#####
  15h   6 |####          <- 20:30 IST
  18h   6 |####
  16h   5 |####          <- 21:30 IST
  03h   5 |####
  02h   3 |##
  23h   3 |##
  19h   2 |#
  17h   1 |#             <- 22:30 IST, the quietest hour on the controller
  20h,21h,22h,00h,01h: no builds at all in 14 days
```

**Recommended window: 17:00–18:00 UTC = 22:30–23:30 IST.** One build finished in that hour in
the last fortnight, and no dated cron fires in it — the scheduled jobs sit at 00:00, 01:05,
03:30, 06:30, 08:00, 11:30, 12:05, 13:00, 14:30 and 23:00 UTC. Two crons are hash-spread
(`H */1 * * *` hourly, `H */6 * * *`) so they can land anywhere; check the queue is empty
immediately before restarting rather than assuming.

**Do not restart during 05:00–11:00 UTC** (10:30–16:30 IST) — that is the working peak, and it
is when `prod/*`, `uat/*` and `qa1/*` deploys run.

**How to tell what is in flight** — `build.xml` is written when a build *finishes*, so looking
for builds without a `<result>` gives a false all-clear. Use the live log instead:

```
find /var/lib/jenkins/jobs -path '*/builds/*' -name log -newermt '-3 minutes' | wc -l
grep -c 'BuildableItem\|WaitingItem\|BlockedItem' /var/lib/jenkins/queue.xml
```

Both must read 0. At 11:22 UTC on 2026-08-18 they read 5 and 2 — `prod/fe #1566`,
`uat/batchprocessor #152`, `ecr-replication #485`, `qa1/ckauto #273`,
`dev3/Stormus-Build-Publish-NexusJob #142` running, with `nonprod-ck-drupal-deploy-app #1598`
queued for an executor.

### 4c. The restart activates more than ck-aws — known as of 2026-08-18

The upload landed cleanly at 11:41 UTC: `plugins/ck-aws.jpi` is 95,749 bytes,
`Plugin-Version 2.2.0`, `Implementation-Build 1bf157ee…`, sha256
`f2d3a59eb808ccf4ffb0a9166f21eef43edf9d95b0b6b6ce72691ec46dbbbaa6` — byte-identical to the
artifact the canaries ran against. It was the only thing written; nothing else in `plugins/`
changed in that hour, and no `.pending`/`.tmp` was left behind.

But Jenkins' *Download progress* page lists the whole update-centre queue, not just the current
action, and it showed two more entries. Both were staged on **17 Aug 05:53**, a day earlier and
unrelated to this work, and both have been waiting for a boot ever since:

| Plugin | Staged | Note |
|---|---|---|
| `role-strategy` | 17 Aug | **This is the live authorization strategy** — `config.xml` names `RoleBasedAuthorizationStrategy`. Its `.bak` is from Nov 2025 |
| `commons-lang3-api` | 17 Aug | Library, almost certainly role-strategy's dependency. `.bak` from Sep 2024 |

**So the next restart activates three plugin upgrades, only one of which is ours**, and one of
the other two governs who can do what in Jenkins. ck-aws lands inert; role-strategy does not.
If permissions misbehave after the restart, look there first — and do not let a role-strategy
regression get attributed to ck-aws, or vice versa.

Two clean options, both the operator's call: restart and accept all three, or restore
`role-strategy.bak` and `commons-lang3-api.bak` over their `.jpi` first so the restart carries
only ck-aws. The second buys unambiguous attribution at the cost of one extra write.

**Provenance, from the Jenkins log — not inferred:**

```
17 Aug 05:53:20  Adding dependent install of commons-lang3-api for plugin role-strategy
17 Aug 05:53:20  Starting the installation of commons-lang3-api on behalf of randeep.arora@cloudkeeper.com
17 Aug 05:53:20  Starting the installation of role-strategy    on behalf of randeep.arora@cloudkeeper.com
```

So `role-strategy` was a deliberate single action by Randeep Arora, `commons-lang3-api` came
with it as a dependency, and those were the only two plugins touched that day. Not caused by
the ck-aws upload — they would have activated at the next boot regardless. We inherit them only
because we are the reason a restart is being scheduled, so **tell Randeep before restarting**:
their change goes live at the same moment as ours.

### 5. `numExecutors` — a POC artefact, NOT an infra concern

Executors reset to 0 on every restart **of the clone**, which stalled a real queued build
during testing. The cause is POC-only: `init.groovy.d/pocInit06KillResumedBuilds.groovy` calls
`setNumExecutors(0)` on every start, to stop resumed production builds running on a clone.
Infra has no `pocInit*` hooks — they were pushed onto the clone after it was built, never part
of the AMI. **Do not carry this step into the infra runbook.**

### 6. Accept these two known limits before starting

- **`qa-virtuoso-resource-creation`** (daily) assumes a role explicitly and exports the
  credentials as environment variables, which outrank `AWS_CONFIG_FILE` in every AWS SDK.
  Calls before the assume are attributed and the `AssumeRole` itself is attributed; calls
  after it are not, though they stay one join away. One line in that repo fixes it; no plugin
  can. This is the only live gap — everything else once listed is disabled, dormant 1100+
  days, or has never run.
- **Terraform's provider-level `assume_role`** (3 of 802 jobs) makes a second hop whose calls
  carry `aws-go-sdk-<nanotime>`. Those calls *are* audited — the first hop is `jk-`, so the
  chain is traceable — they are just not labelled with the build. A `*_override.tf` fix was
  built and then **deliberately removed** before release: it was wired into the Pipeline path
  only, it could not replace an override deleted mid-build, and it was not worth the surface
  for a labelling improvement. Do not reintroduce it without reading MEMORY.md addendum 6.
- A node whose role AWS will not let self-assume stays **unattributed but working**.

### 7. Set up gap detection after the install

Log recorder on `io.github.rads4.ckaws` at WARNING, plus the CloudTrail session-name buckets.
See *Detecting unaudited calls automatically* below. Deferred by decision, but without it
nothing reports centrally.
---

# Decision record — moved out of this file

The full rationale for **v2.0 (2026-08-07)**, **M12 Universal Attribution** and
**M11 Managed Authentication** now lives in [`docs/DECISION-LOG.md`](docs/DECISION-LOG.md).
Read it before reopening any of those decisions — they are settled, and the log records why.

# Architecture

## The layered architecture

The system is four layers. Each layer is independently adoptable, independently
revertable, and depends only on the layer below it.

```
Layer 3   IAM trust policy: "sts:RoleSessionName" StringLike "jk-*"
          (AWS-side, non-bypassable, outside Jenkins entirely)
              ^ made possible by Layer 1's session naming

Layer 2   OPTIONAL execution conveniences
          ckAws.run([...]) — retry/timeout/typed result for callers that
          genuinely are a plain AWS CLI invocation. Never required.

Layer 1   THE CONTRACT (mandatory, the only thing consumers must adopt)
          ckAwsWithProfile('non_prod') { ... }
          AssumeRole with jk-<job>-<build>, executed on the agent,
          credentials exported into the block as AWS_* environment variables,
          masked in the console, expiring at block exit.

Layer 0   CONFIGURATION
          profile name -> role ARN (+ region) in JCasC.
          Jenkins-admin-owned, version-controlled, reviewable.
```

**Layer 1 is the product.** Layers 0 and 2 exist to serve it. Layer 3 is the
only real enforcement and is not ours to deploy.

> **Superseded as the rollout mechanism (2026-08-06).** Everything in this layered
> model is implemented and validated end to end, and Layer 0 is unchanged and still
> required. But Layer 1 is opt-in, so it can be forgotten — and the requirement is
> now that repositories stay unaware of the plugin entirely. `ckAwsWithProfile` is
> retained as **Layer 1B, the explicit override**, not as the way pipelines are
> expected to authenticate; **Layer 1A (managed injection) becomes the default
> path.** See "M11 — Managed Authentication" above. The reasoning recorded in this
> section remains accurate and is kept deliberately; only the conclusion about
> *which mechanism carries the rollout* has changed.

## Principle 1 — the plugin owns identity, and only identity

The plugin's entire responsibility is answering *"who is this build allowed to
be, and for how long?"*:

- Resolve a profile name to a role ARN through Jenkins-owned configuration.
- Generate the `jk-<job>-<build>` session name.
- Perform STS AssumeRole, on the machine where the work will run.
- Publish the resulting credentials into a bounded scope.
- Withdraw them when that scope ends.

Everything else is out of scope, permanently and by design.

## Principle 2 — execution stays outside the plugin

The plugin does not execute AWS commands on behalf of consumers. Consumers keep
running whatever they already run — `sh "aws ..."`, `terraform apply`,
`python3 script.py`, `docker login` — inside the authenticated block.

This is not a concession. It is the property that makes the plugin reusable.
There are at least four distinct ways an AWS consumer obtains credentials, and
only one of them is "hand an argument list to something":

| Consumer shape | How it takes credentials |
|---|---|
| `sh "aws ecs update-service ..."` | env vars, or `--profile` |
| `aws ecr get-login-password \| docker login ...` | env vars (it is a **pipeline**, not an argv) |
| boto3 `Session()` | env vars, or `profile_name` |
| `terraform apply` | env vars only |

The exported-environment contract serves all four. An argument-list executor
serves one. Since the plugin's goal is to be consumable by *any* Jenkins shared
library or Jenkinsfile, the contract must be the environment.

## Principle 3 — profile → role resolution is Jenkins-owned

Consumers name an environment; they never name an ARN.

```groovy
ckAwsWithProfile('non_prod') { ... }
```

The `profile → roleArn (+ region)` mapping lives in **Jenkins Configuration as
Code**, under `unclassified.ckAws.profiles`. Consequences, all intended:

- Changing which role an environment maps to requires Jenkins admin permission.
- The mapping is version-controlled and reviewable like the rest of platform
  config.
- Consumers are portable: the same Jenkinsfile works in another Jenkins with a
  different account, because it contains no ARNs.
- Unknown profile names **fail closed**, with an error listing the configured
  profiles.

The plugin must **never read `~/.aws/config`**. That file lives on the agent
filesystem, outside Jenkins' permission model, editable by anyone with agent or
SSH access. Depending on it would place the identity decision outside the system
that is supposed to be making it. This rule predates M6 and is unchanged.

Role ARNs are **not secrets**. They must not be routed through the Jenkins
Credentials plugin, which is the wrong tool for non-secret configuration.

An explicit `roleArn:` parameter exists as a documented escape hatch for
pipelines whose profile has not been added to JCasC yet (e.g. an initial
Terraform-repo migration). It is not a security boundary — a pipeline author who
wants an arbitrary ARN can already call `sh "aws sts assume-role"` directly. The
security boundary is Layer 3.

## Principle 4 — the block-scoped authentication wrapper

Layer 1's shape is a **block**, not a value-returning step:

```groovy
node {
    ckAwsWithProfile('non_prod') {
        sh "aws ecs update-service ..."          // inherits the session
        sh "terraform apply -auto-approve"       // inherits the session
        sh "python3 code/dr_sync.py"             // inherits the session
    }
}
```

Why a block and not a returned value:

1. **Authentication is inherently scoped.** "For the duration of this work, be
   this identity" is a region of a program, not a point in it. A returned value
   cannot express a scope, cannot clean up, and cannot host credential refresh.
2. **Credentials must not enter CPS program state.** A value returned to a
   pipeline variable is serialized into the flow's `program.dat` on disk and is
   trivially printable from a pipeline. Block-scoped credentials live in the
   body's `EnvVars` and in an `EnvironmentExpander` that stores them as
   `hudson.util.Secret`, never as plaintext in program state.
3. **The block is the only place refresh can live** when the 1-hour chained
   session cap becomes a problem.
4. **It matches what Jenkins users already know** — `withCredentials`,
   `withAWS`, `withEnv`. Adoption cost is one wrapping line.

Exported into the block:

| Variable | Source |
|---|---|
| `AWS_ACCESS_KEY_ID` | AssumeRole result (masked) |
| `AWS_SECRET_ACCESS_KEY` | AssumeRole result (masked) |
| `AWS_SESSION_TOKEN` | AssumeRole result (masked) |
| `AWS_REGION`, `AWS_DEFAULT_REGION` | profile mapping or step parameter, if configured |
| `CK_AWS_SESSION_NAME` | the `jk-<job>-<build>` session name — non-secret, for logging and debugging |

All three credential variables are declared sensitive and are masked in the
console for the duration of the block.

## Principle 5 — execution happens where the work happens

The AssumeRole subprocess runs **on the agent selected by the enclosing `node`
block**, via the step context's `Launcher` — not on the Jenkins controller.

This is a correctness requirement, not a preference:

- The base identity the plugin chains from (EC2 instance role) is the **agent's**
  identity. Assuming from the controller would use the wrong base identity and,
  in most topologies, would simply be denied.
- Credentials produced on the controller are useless to `sh` steps that execute
  on an agent.
- The controller's identity is broader than an agent's. Authenticating there
  would be a privilege escalation relative to the status quo.

Consequently `ckAwsWithProfile` requires `Launcher` and `FilePath` context and
therefore must be used inside a `node { }` block. Outside one, it fails closed
with an actionable message.

## Principle 6 — the plugin is generic; genericity is a hard constraint

No layer of the plugin may contain:

- Per-AWS-service logic. The process execution primitive must never branch on
  `args[0]`. If you find yourself writing `if (args[0] == "ecs")`, stop.
- CloudKeeper-specific names, conventions, account IDs, or role names. `prod`
  and `non_prod` are *data in someone's JCasC file*, not identifiers in the
  source tree.
- Deployment-specific logic. The plugin has no concept of a deployment, a task
  definition, a cluster, or a rollback.

The plugin is validated against the CloudKeeper deployment library because that
is the first consumer available — not because it is the only intended consumer.
Terraform pipelines, standalone Jenkinsfiles, and future shared libraries are
first-class consumers of the same contract.

---

## Rejected designs

See [`docs/DECISION-LOG.md`](docs/DECISION-LOG.md) — do not re-propose these.

# The session-naming convention (unchanged, load-bearing)

```
jk-${JOB_NAME}-${BUILD_NUMBER}
```

produces CloudTrail sessions like `jk-myjob-123` instead of generic
auto-generated names. **Do not change this shape without discussion.** It is the
basis for the Layer 3 IAM trust-policy condition:

```json
"Condition": {
  "StringLike": { "sts:RoleSessionName": "jk-*" }
}
```

Any AssumeRole with a non-conforming session name is denied by AWS itself. This
is the only truly non-bypassable control in the system; nothing inside Jenkins
can be, because it all runs inside the same trust boundary as an unrestricted
shell.

**Why this now matters more than it did at M0.** When the AWS CLI resolves a
profile that has a `role_arn`, it performs the AssumeRole itself and generates
its *own* session name (`botocore-session-<epoch>`) unless `role_session_name`
is pinned in the config file — and even pinned, it is static per profile and can
never carry a job name or build number. So any consumer authenticating via
`--profile` today has **zero build attribution in CloudTrail**, and would be
**denied outright** the day the Layer 3 trust policy is applied. Migrating
consumers to Layer 1 is a prerequisite for Layer 3, not an optional tidy-up.

# Known constraint: chained AssumeRole session duration

EC2 instance role → AssumeRole into a target role is role chaining, which caps
the session at **1 hour regardless of the target role's configured maximum**.
Builds longer than an hour need a credential refresh path. The block scope is
where that will live. Until it exists, a build whose block runs past the hour
will fail on credential expiry — this must be measured against real deployment
job durations before wide adoption.

> **Resolved by M11 for the managed path.** When the AWS tool performs its own
> AssumeRole from a generated config profile, refresh is native: botocore returns
> `RefreshableCredentials` and re-assumes on expiry under the same session name.
> The plugin does not have to implement refresh at all. The constraint above still
> applies inside an explicit `ckAwsWithProfile` block (Layer 1B), which holds a
> fixed set of credentials for the life of the block.

---

# Configuration reference

```yaml
unclassified:
  ckAws:
    profiles:
      - name: "non_prod"
        roleArn: "arn:aws:iam::123456789012:role/non_prod"
        region: "us-east-1"
      - name: "prod"
        roleArn: "arn:aws:iam::210987654321:role/prod"
        region: "us-east-1"
```

`region` is optional. Profile names are free-form strings chosen by the Jenkins
admin; the plugin attaches no meaning to any particular value.

M11 gives each profile an explicit **authentication mode**, so the same-account
case is a documented configuration rather than an inference from a blank field:

```yaml
unclassified:
  ckAws:
    managedAuthentication: true
    profiles:
      - name: "non_prod"
        mode: "AssumeRole"          # cross-account; carries jk-<job>-<build>
        roleArn: "arn:aws:iam::123456789012:role/non_prod"
        region: "us-east-1"
      - name: "ops"
        mode: "InstanceProfile"     # same-account; the agent's own identity
```

| Mode | Role ARN | Rendered as | CloudTrail |
|---|---|---|---|
| `AssumeRole` | required | a profile resolving through the plugin's helper | `jk-<job>-<build>` |
| `InstanceProfile` | not used | a profile section with no credential keys | the agent's instance-role session, as today |

A profile section carrying no credential keys makes the AWS SDKs fall *through* to
the agent's identity; an **unknown** profile is a hard error. That asymmetry —
verified against botocore 1.42.65 — is what lets a same-account profile keep
working without the plugin pretending to authenticate it.

M11 also adds three global settings alongside the mapping, all optional except the
first, and all on the same screen:

| Setting | Default | Purpose |
|---|---|---|
| `managedAuthentication` | `false` | Master switch. Ships off, so the upgrade is inert. Toggling it is a restart-free rollback |
| `jobRules` (M12) | empty | Ordered `pattern -> profile` rules, matched against a job's full name; first match wins. Selects the `[default]` identity for calls that name no profile. **No match means the plugin does nothing for that job** |
| `jobNamePattern` (M11) | empty (= all jobs) | Staged rollout by job full name. Superseded by `jobRules`, which both gates and selects |
| `credentialSource` | `Ec2InstanceMetadata` | Base identity of the agent. Accepts the three values botocore validates (`Ec2InstanceMetadata`, `EcsContainer`, `Environment`), so the plugin is not tied to EC2 agents |

A `defaultProfile` setting is deliberately **not** included initially — see M11's
limitation 4.

**Adding a profile is a row in System configuration. It must never require a
plugin release.** Nothing in the source tree may enumerate, default to, or branch
on a profile name.

---

## Migration strategy

See [`docs/DECISION-LOG.md`](docs/DECISION-LOG.md).

# Organizational context

- CloudKeeper's Platform/DevOps team owns Jenkins, shared Groovy libraries,
  deployment Groovy files, and Jenkins plugins. Application teams do not write
  deployment logic.
- The deployment library (`cln-deployment-scripts`) is `Utilities.groovy`,
  `Build.groovy`, `Deploy.groovy` and 12 `vars/*.groovy` entry points. It
  authenticates entirely through `aws ... --profile ${prof}`, where `prof` is a
  plain string (`envName == 'prod' ? 'prod' : 'non_prod'`) set in 12
  `vars/*.groovy` entry points and threaded through 9 function signatures. It
  performs no explicit STS call anywhere. Measured, not estimated: **13 AWS CLI
  invocations across 4 files** — 12 carrying `--profile`, 1 not (see below).
  - *Correction to earlier versions of this document:* there is no
    `AwsAuth.groovy` and no `Audit.groovy` in that repository. Earlier drafts
    described an explicit `sts assume-role` inside the shared library; that call
    site does not exist.
  - `Utilities.dockerLoginEcr(ecrUrl, regionName, prof)` accepts `prof` and
    **does not use it** — its `aws ecr get-login-password` call carries no
    `--profile` and therefore runs as the ambient instance role. Under Layer 1
    its identity silently becomes the assumed role, which must therefore hold
    `ecr:GetAuthorizationToken`. Verify before rollout.
- `code/dr_sync.py` uses boto3 `Session(profile_name=...)` from environment
  variables — a non-CLI consumer.
- The infrastructure repository (`cln-infra-terraform`) runs Terraform from
  `jenkins/*.groovy` pipelines with **no profile at all**: ambient instance role
  plus an `AWS_REGION` environment variable.
- Region is not a constant across the estate: the deployment library hardcodes
  `us-east-1`, DR sync targets `us-east-2`, Terraform reads `AWS_REGION`. Region
  must always be an input, never an assumption.

---

# Repository strategy

The plugin lives in its own independent repository (`ck-aws-plugin`), currently
on personal GitHub, kept independent of `cln-deployment-scripts` and
`cln-infra-terraform` — no dependency in either direction. That isolation is
deliberate and is reinforced, not weakened, by the layered architecture: the
plugin has no CloudKeeper-specific content, so it has no reason to live inside a
CloudKeeper application repository.

Long-term repository ownership remains an open decision, to be made on
maintainability, ownership, release process, and CK standards. Nothing in this
document favours an option.

---

## Definition of Done — M6 and M11

See [`docs/DECISION-LOG.md`](docs/DECISION-LOG.md).

# What NOT to do

- Do not add per-AWS-service logic anywhere.
- Do not read `~/.aws/config` from inside the plugin.
- Do not put CloudKeeper names, ARNs, account IDs, or deployment concepts in the
  source tree.
- Do not make Layer 2 (`ckAws.run`) a required interface, or a dependency of
  Layer 1.
- Do not return credentials from a step to the Pipeline DSL.
- Do not implement the IAM trust-policy enforcement as part of plugin work — it
  is an AWS-side change requiring access this project does not have, and it must
  come last.
- Do not modify the deployment library or the Terraform repository without
  explicit per-change approval.
- Do not swap to the AWS Java SDK without flagging it first.
- **Do not write credential material into the generated AWS config file.** Its
  security properties — best-effort cleanup, no masking, relaxed permissions
  tolerance — all depend on it containing nothing but a role ARN, a session name,
  a region and a `credential_source`.
- **Do not make Managed Authentication default to on.** The master switch ships off so
  that a plugin upgrade is behaviourally inert.
- **Do not put the generated file in the workspace, and never trust that a file in
  `@tmp` is still there.** `deleteDir()`, `git clean -fdx` and `cleanWs()` all
  reach those. This rule predates the implementation and the implementation
  drifted from it: 2.1 and every unreleased build before the fix wrote to
  `<workspace>@tmp` and memoized the
  path for the whole build, so a mid-build clean left every later step exporting
  `AWS_CONFIG_FILE=<deleted file>`. An AWS SDK reads a missing config file as an
  *empty* one, so `--profile x` then fails with "The config profile could not be
  found" — nothing thrown, nothing logged. Reproduced in
  `ProductionFailureModesTest`. 2.2 keeps the `@tmp` sibling (the only location a
  container mount is guaranteed to see) and **re-checks existence before reusing
  the memo**, regenerating when it is gone. Either invariant is acceptable;
  silently handing out a stale path is not.
- **Do not anchor the generated file to the current working directory.** A
  `DynamicContext` is handed the *current* `FilePath`, so inside
  `dir('application/service')` that is a directory in the middle of a checked-out
  source tree. 2.2 wrote `ck-aws/` there — once per `dir` block — and for a `dir`
  outside the workspace would leave a directory no cleanup reclaims. Anchor to
  `node.getWorkspaceFor(job)` when the current path is genuinely inside it, and
  fall back otherwise so `ws()` and concurrent `…@2` workspaces stay correct.
- **Do not return a bare value from a `DynamicContext` whose type another
  contributor also supplies.** `ContextVariableSet.get` scans the current level,
  consults every `DynamicContext`, and only *then* recurses to the parent. An
  unconditional non-null answer therefore **shadows the enclosing level**. For
  `EnvironmentExpander` that means every binding published by `withCredentials`,
  `withEnv`, `withAWS`, `withSonarQubeEnv` or `configFileProvider` vanishes the
  moment any inner block (`dir`, `ws`, `container`) adds a context level of its
  own. Always `merge(ours, context.get(EnvironmentExpander.class))` — and note
  `merge()` null-checks only its *first* argument, so a null second argument NPEs
  on expand. Ours must expand **first**, so a value the job set deliberately
  wins. Cost of getting this wrong, measured: `dev2/rivon` #942 and #944 lost
  their Nexus credentials and failed; #943, with the job out of scope, passed.
- **Do not memoize without a lock.** Parallel branches share a workspace and will
  both miss the memo and write the same file at once; a third branch can then
  read a half-written config. `putIfAbsent` does not prevent this — it
  deduplicates the entry after both writes have already happened.
- **Do not perform an STS call from a `DynamicContext`.** It is consulted for
  every step; the managed path must stay I/O-free after the first write. The
  AssumeRole belongs to the AWS tool, not to the plugin.
- Do not add a second configuration location (job property, folder property) —
  the global mapping is the source of truth.

---

# What 2.2 exports, and why each one

| variable | purpose | covers |
|---|---|---|
| `AWS_CONFIG_FILE` | points every AWS tool at the decorated copy of the node's own config | AWS CLI, boto3, Terraform's default credential resolution — **every** AWS call that goes through a shell |
| `CK_AWS_SESSION_NAME` | the build's session name, for a script that wants to log or tag with it | informational |
| `AWS_ROLE_SESSION_NAME` | names a session when a **tool assumes a role itself** | the Terraform "second hop" — see below |

## The second-hop problem

The generated config names the session for every role the *shared config* assumes. It
cannot name a hop the tool performs on its own: a Terraform provider carrying its own
`assume_role` block assumes a further role from the already-assumed session, and with no
`session_name` in that block the provider picks the name. The calls that actually change
infrastructure then carry a generated name, and CloudTrail ties them back only through
the AssumeRole event.

Measured: **3 of 21 Terraform jobs** have a provider-level `assume_role` and none sets
`session_name` — `cln-infra-terraform-pipelines/cln-app-terraform-pipeline`,
`ck-analytics-app-services-terraform`, `ck-ecs-terraform`. The other 18 use the profile
from `AWS_CONFIG_FILE` and are fully attributed already.

**The fix must be plugin-side, not repo-side.** The premise of this plugin is attribution
with no change to any Jenkinsfile, `.tf` file or shared library — a fix that requires
editing repositories is not a fix. Every AWS SDK reads `AWS_ROLE_SESSION_NAME`, so
exporting it names the second hop without touching anything.

**Status: MEASURED, and it does NOT work.** `AWS_ROLE_SESSION_NAME` was exported into a
build that ran `terraform plan` against a provider with its own `assume_role` block. The
provider ignored it:

```
CALLER:    .../ck-ops-jenkins-master-instance-iam-role/jk-poc-canary-terraform-secondhop-2
requested: roleSessionName = aws-go-sdk-1786899555220461151
RESULT:    .../terraform-assume-role/aws-go-sdk-1786899555220461151
```

The Terraform AWS provider builds the second AssumeRole from the `assume_role` block
alone and generates `aws-go-sdk-<nanotime>` when `session_name` is absent. No environment
variable reaches it.

**Superseded 2026-08-17.** A Terraform `*_override.tf` created in the working directory at build
time sets `session_name` on the provider's own `assume_role`, proven against the real cross-account
role with the real repo shape (MEMORY.md addendum 6). The override MUST copy `role_arn` verbatim —
omitting it silently drops the assume and Terraform runs as the raw instance role. The text below
records the original measurement, which remains accurate about environment variables. The only direct
fix is `session_name` in the provider block — a repository change, which the project's
premise excludes.

**What is true instead: attribution is transitive, and complete.** CloudTrail records the
*caller* of that AssumeRole as `jk-<job>-<build>`. So every Terraform API call can be
tied back with one join: `aws-go-sdk-<n>` → the AssumeRole event that created it →
`jk-<job>-<build>`. Direct labelling is missing for 3 of 802 jobs; traceability is not.

The export is retained because it is additive and free, and it may name a second hop for
any tool that *does* read it — but it must not be described as covering Terraform.

---

# Residual risk register (as of 2.2)

Everything below was established by reproduction or by census, not by argument.
Re-verify before widening scope.

## The safety net is not the master switch

The master switch is what you reach for *after* something breaks, and it needs a
human watching a queue. The real net has three layers:

1. **Structural.** The plugin can now only *add*, and **both** surfaces it touches
   are checked at runtime, not argued from construction:
   - config file — `AwsConfigOverlay.validate()`
   - environment — `ManagedAwsContext.wouldRemoveSomething()`, which expands the
     enclosing environment and the proposed merged one and compares them. If any
     variable the enclosing block set would be dropped or altered, the plugin
     contributes nothing and the build keeps its own environment.

   This closes the defect class that broke `dev2/rivon`: a contribution that
   succeeded and still took something away, which the exception guard could never
   have caught because nothing threw.

   The environment check earlier relied on merge ordering being correct by
   construction. That was replaced because "correct by construction" is exactly the
   claim that failed for rivon, and because the ordering argument depends on
   `DelegatedContext.get` returning the enclosing expander faithfully. Should it
   ever return null, a partial view, or a different level — in a nesting shape
   nobody has written yet — the merge is built from the wrong base and the argument
   silently stops holding. Comparing actual expansions needs no such assumption, so
   **an unimagined shape fails safe instead of failing silently.** That is what
   makes future jobs safe without enumerating them.
2. **Per-job, instant.** *Except jobs matching* is a kill switch for one job that
   needs no restart and no global toggle. Reach for it before the master switch.
3. **Observe only.** Prepare everything, export nothing. Widen the scope to every
   job, let real traffic run for a day, read the evidence, then enforce. This is
   how you survey 740 jobs — including the 637 whose Jenkinsfiles live in SCM —
   without any possibility of affecting one of them, and without watching a queue.

   **Precisely what it does:** the whole path runs — the node's config is read,
   decorated, validated, and the file *is* **written** to `<workspace>@tmp/ck-aws/
   config`. Only the export is withheld, so `AWS_CONFIG_FILE` and
   `CK_AWS_SESSION_NAME` stay unset, nothing reads the file, and behaviour is
   unchanged. Verified both halves on a canary: file present, variables unset.

   **The record lives only in each build's console log** — there is no aggregation
   and no central report. Observe-only answers "what would this build do", one
   build at a time. "Is anything slipping through" is a CloudTrail question.

Reach for them in that order. The master switch is the fourth thing, not the
first.

## Cannot fail — proven and test-locked

Context shadowing, source-tree pollution, stale memo after a mid-build clean, the
parallel write race, and a job's own `AWS_CONFIG_FILE` being overwritten. All in
`ProductionFailureModesTest`. The shadowing fix is at the context-resolution
layer, so it holds for **every** step — including ones nobody has written yet.

## Coverage is structurally complete

806 job configs: 740 Pipeline, 39 Freestyle, 27 folders. **No matrix, maven,
multibranch or external jobs.** Every buildable job is either a `WorkflowRun`
(hooked by `ManagedAwsContext`) or an `AbstractBuild` descendant (hooked by
`ManagedAwsFreestyleEnvironment`, which is additive and cannot shadow). Matrix and
Maven builds, should any appear, are `AbstractBuild` descendants and are covered
by the Freestyle path for free.

## Will not be audited today — by design, not by defect

- **Calls naming no profile on a node that cannot self-assume.** Bare `aws`
  commands resolve straight to IMDS, whose session name EC2 fixes to the instance
  ID. *Attribute unprofiled calls as the node's own instance role* closes this for
  every node whose role can assume itself — proven on all 7 slave types and the
  controller. A node where the probe fails is left exactly as it was and logs a
  warning; it stays unattributed rather than breaking.
- **Terraform's own `assume_role` block** (55 of 432 workspaces). `session_name`
  appears in no `.tf` file, so the second hop gets an SDK-generated name. Still
  traceable transitively — CloudTrail records the `jk-*` session that called
  `AssumeRole`. Closing it is a Terraform-repo change: set `session_name` from the
  `CK_AWS_SESSION_NAME` this plugin already exports.

  **Transitivity confirmed empirically 2026-08-25**, so this is cosmetic rather
  than a coverage gap: across 18h of full-traffic production, **all 12** second-hop
  events carried a `jk-` caller — `cleanup-275595855473` ←
  `jk-ck-iam-role-cleaner-*`, `jenkins-session` ←
  `jk-qa-virtuoso-resource-creation-484`, `Route53BackupSession` ←
  `jk-*-route53-backup-*`. Every one is recoverable with one join.

## Configuration surface — what is load-bearing and what is not

**2.3 removed two form entries.** *Attribute unprofiled calls as* (the static ARN) and *Apply on
nodes labelled* are gone from the UI; both properties are retained `@Deprecated` so existing XML
still loads and the tests keep a settable ARN and a way to pin to one node. The form is now **eight
fields**. What follows describes the full property set, including the two that are no longer typeable.

Two fields reviewed as "dead weight" were kept after tracing their callers, and the reasoning is
worth not repeating: **`profiles` is the configuration source for `CkAwsWithProfileStep`** — removing
it deletes a shipped feature, not clutter — and **`credentialSource` is written into every generated
`[default]`** (`credential_source = Ec2InstanceMetadata`), so it is functional output even though its
value has never varied.

**Load-bearing — do not remove:**

| Field | Why |
|---|---|
| *Managed authentication* | Master switch, and the restart-free rollback |
| *Except jobs matching* | The incident switch: one job excluded without a global toggle |
| *Attribute unprofiled calls as the node's own instance role* | Audits ~98% of calls. Without it coverage is ~2% |
| *Observe only* | The rollout mechanism; full scope at zero risk |
| *Diagnostics* | Every piece of POC evidence came from it |

**Removed from the form in 2.3:**

- ***Attribute unprofiled calls as* (the static role ARN) was a footgun.** It was
  used as a deliberate poison pill during testing — pointing it at a nonexistent
  ARN breaks every bare `aws` call in every build, which is exactly what a typo
  would do. Fully superseded by the per-node checkbox, which resolves each node's
  real role and *verifies the assume succeeds* before using it.
- ***Apply on nodes labelled*** — no demonstrated use case. Its original
  justification (agents that differ, or make no AWS calls) is handled
  structurally: a node with no config gets nothing contributed, and a node that
  cannot self-assume is detected and skipped. Never used in any test or run.

**Kept after tracing callers — these are NOT dead:**

- ***AWS profiles*** (the repeatable list) — every diagnostic block in the POC
  printed `sections appended: []`, so it has never appended a profile. But it is
  also the configuration source for `CkAwsWithProfileStep`, the M11 override layer.
  Removing it deletes a shipped feature.
- ***Agent base identity*** — only affects profiles the plugin writes, which in
  practice is just `[default]` — but it *is* written there, on every build. On EC2
  it is always `Ec2InstanceMetadata`; it would matter if agents moved to ECS or EKS.

*Apply to jobs matching* sits in between: largely redundant now that observe-only
surveys everything at zero risk, but it costs nothing and was used constantly.

## Detecting unaudited calls automatically

Neither the plugin nor observe-only reports centrally. Two mechanisms cover it,
and they catch different things — use both.

1. **Jenkins side — a log recorder.** No code, no build impact. *Manage Jenkins →
   System Log → Add recorder* on `io.github.rads4.ckaws` at WARNING collects every
   decline in one place: nodes with no resolvable role, contributions refused by
   the additions-only invariant, and anything the guard swallowed. **It only sees
   what the plugin knows** — it will never show the Terraform second hop, because
   there the plugin succeeded.

   **It also silently forgets.** A Jenkins log recorder is a
   `RingBufferLogHandler` capped at **256 entries** by default. Reads on 2026-08-21
   and 2026-08-25 both returned exactly 256 — the buffer is saturated and rolling,
   and at the observed peak (63 entries in one hour) it turns over in ~4 hours. It
   can only ever prove *"nothing bad in the newest 256 warnings"*, never *"no
   decline ever happened"*. Benign `no FlowNode in this context` noise is what
   fills it, which is why the `isBenignRace()` one-liner is a **detector fix, not a
   cosmetic one**. Raise the cap with
   `-Dhudson.util.RingBufferLogHandler.defaultSize=2000` at the next restart.
2. **AWS side — CloudTrail, and this is the real detector.** It is outcome-based:
   it measures what actually reached AWS. Bucket calls made by the Jenkins instance
   role by session-name shape — `jk-<job>-<build>` is audited; `i-0…` is an
   uncovered unprofiled call; `aws-go-sdk-…` / `botocore-session-…` / 32-hex is a
   tool that assumed a role itself. Anything not `jk-` is a gap, automatically and
   permanently, **including in jobs nobody has written yet.** Implement as a Logs
   Insights query, a metric filter with an alarm, or an EventBridge rule — all
   AWS-side only, structurally incapable of affecting a build.

Known blind spot in both: a job authenticating some entirely other way, e.g. static
keys in a Jenkins credential. The census found none, but neither mechanism would
see one appear.

## Session names: settled, do not re-litigate

All **27** role ARNs in the controller's configuration were probed with
`sts assume-role --role-session-name jk-probe-secops-1`. **26 succeeded.** The one
denial, `275595855473/SecOpsAdminRole`, was re-probed with an SDK-style name and a
neutral name and denied identically — so it is a pre-existing permission gap, not
a session-name restriction. **No trust policy anywhere constrains
`sts:RoleSessionName`.** This risk is closed; re-probe only if trust policies
change.

## No production job has ever been damaged

Scanning build history for AWS auth-failure signatures: `Unable to parse config
file` appears in **7 builds, all of them canaries** (`ckaws-canary*` #3-#5) — the
duplicate-key defect from the unprofiled-ARN experiment, contained entirely to the
canary set. Zero production jobs. `config profile could not be found`: zero.
`The source_profile`: zero. Two `ExpiredToken` and one `sts:AssumeRole` denial
exist in `prod/marketplace`, `qa1/marketplace` and `slack-messages-monitoring` —
all out of scope, all pre-existing, none plugin-related.

## May fail — ranked, with the trigger
2. **Malformed node configuration.** `ops_rds` on the controller carries a
   `role_arn` with no `credential_source`. The plugin adds a session name and
   leaves it no worse, but it was already broken.
3. **Windows agents.** `chmod` would throw, the guard would fail open, and the
   build would run unaudited rather than break. No Windows agents exist today.
4. **Container agents.** `container()`, `docker.inside` and `withKubeConfig` have
   **zero** executed history, so `@tmp` visibility inside a container mount is
   untested. Verify before adopting Kubernetes agents.
5. **Per-step cost.** Re-checking the file's existence is one remote stat per
   step (~1-2 ms). At the observed ~64 steps per build this is noise; a build with
   thousands of steps would notice. Throttle only with evidence.

## How this was measured, so it can be repeated

Build logs record every executed step, which is the only way to see inside the
637 jobs whose Jenkinsfiles live in SCM. Jenkins prefixes each console line with
a binary console note, so `grep '^\[Pipeline\]'` matches nothing — drop the
anchor. Last 3 builds of every job is 690 logs and ~230 MB; run it `nice -n19
ionice -c3` and off-hours.

---

# Working Principles

This project follows an architecture-first development approach. Every milestone
must follow this workflow:

1. Explain the implementation plan.
2. Wait for approval.
3. Implement only the agreed milestone.
4. Keep changes limited to the milestone scope.
5. Run relevant tests.
6. Explain the changes.
7. Stop for review.

Do not continue into the next milestone automatically. Do not redesign previous
decisions unless explicitly requested. Prefer standard Jenkins plugin development
practices over custom abstractions. When uncertain, explain assumptions instead
of making them silently.

Keep auth logic testable without a live Jenkins wherever possible — reserve
`JenkinsRule`-based tests for things that genuinely need a running Jenkins
(extension registration, config persistence, step context, agent launching).

---

# Documentation Maintenance

At the end of every completed implementation session:

1. Update `MEMORY.md` — milestone completed, architectural decisions, validation
   performed, issues encountered, deviations from plan.
2. Review `README.md` — update whenever user-facing behaviour changes.
3. Do **not** modify `CLAUDE.md` automatically.

`CLAUDE.md` is the project's architectural contract and should only be updated
when the architecture changes, new long-term design decisions are made, project
scope changes, or when explicitly requested.

If no architectural decisions changed during a session, leave `CLAUDE.md`
untouched and state that no update was necessary.
