# HL7 AU JIRA Spec Artifacts — context

How the HL7 AU FHIR IG specification files used by HL7 JIRA (specification feedback / ballot
configuration) are generated and maintained.

## The two repos

| Path | Role |
| --- | --- |
| `C:\repos\HL7BallotTool` (this repo) | Generator. A Visual Studio T4 template turns published IG definitions into the JIRA `specification` XML files. |
| `C:\repos\JIRA-Spec-Artifacts` | The live content loaded into HL7 JIRA for **all** HL7 projects. Generated AU files are copied here and PR'd upstream. |

`JIRA-Spec-Artifacts` remotes:
- `origin` → `https://github.com/hl7au/JIRA-Spec-Artifacts.git` (HL7 AU fork — push here)
- `upstream` → `https://github.com/HL7/JIRA-Spec-Artifacts.git` (HL7 international — PR target)

Background: <https://confluence.hl7.org/display/HL7/Configuring+Specification+Feedback>

## Generator

[HL7BallotTool/ballot.tt](HL7BallotTool/ballot.tt) is a T4 (`TextTemplatingFileGenerator`) template —
it is **not** run by `dotnet build`. Regenerate from Visual Studio: right-click `ballot.tt` →
**Run Custom Tool** (or save the file). `Program.cs` is a stub; `ballot.txt` is the throwaway
T4 output — the real outputs are written by the `save()` helper directly into
`HL7BallotTool/`.

It can also be run headless, which is handy for scripted/agent regeneration. The
`<#@ assembly name="Newtonsoft.Json" #>` directive resolves inside the VS host but not from the
CLI, so pass `-P` with a Newtonsoft.Json directory:

```bash
cd /c/repos/HL7BallotTool/HL7BallotTool && "/c/Program Files/Microsoft Visual Studio/18/Professional/Common7/IDE/TextTransform.exe" -P "C:\Users\brett\.nuget\packages\newtonsoft.json\13.0.3\lib\net45" -out ballot.txt ballot.tt
```

The template loops over hard-coded IG/change pairs. Adding a new IG means editing the
`filenames` and `changes` arrays at the top of `ballot.tt`, adding the two input files, and —
on the deployment side — registering the spec key in `JIRA-Spec-Artifacts/xml/SPECS-FHIR.xml`
(`<specification key="…" name="…"/>`). JIRA will not offer the spec until that entry exists.

### Inputs (per IG, in `HL7BallotTool/`)

1. `ImplementationGuide-<package id>.json` (usually `hl7.fhir.au.<key>`, but see AU Terminology)
   — the **ImplementationGuide resource from the
   latest published version** of that IG (not the CI build, unless deliberately generating from
   CI — see commit `447b766 "AU Base ci generation"`). Supplies
   `definition.resource[]` → `<artifact>` and `definition.page` (recursed) → `<page>`.
2. `ImplementationGuide-hl7.fhir.au.<key>.change.json` — the hand-maintained overlay:

   | Field | Effect |
   | --- | --- |
   | `speckey` | spec key as registered in `SPECS-FHIR.xml`; sets the output filename (`FHIR-<speckey>.xml`). Optional — falls back to mangling the IG filename |
   | `workgroup` | `defaultWorkgroup`. Optional — defaults to `au-fhir`. Must be a key from `JIRA-Spec-Artifacts/xml/_workgroups.xml` |
   | `cibuild` | `org/repo` → `ciUrl` (`https://build.fhir.org/ig/…`) and `gitUrl` |
   | `canonical` | `url` on `<specification>`, and the base for each version URL |
   | `ballot` | `ballotUrl`. Optional — omit for a spec that has never been balloted and no `ballotUrl` attribute is emitted |
   | `defaultver` | `defaultVersion` |
   | `versions[]` | one `<version>` each, in order; prefix a value with `deprecated:` to emit `deprecated="true"` |
   | `deprecate[]` | `{id, key, name}` artifacts that no longer exist in the IG but must remain, emitted with `deprecated="true"` |
   | `pagedeprecate[]` | `{key, name}` same, for pages |
   | `exclude[]` | artifact `reference.reference` ids to leave out of the output entirely. For IG scaffolding only (e.g. the `StructureDefinition/myobservation` placeholder in AU Encounter) — never a way to drop a real artifact, which must be deprecated instead |
   | `change[]` | `{id, name}` renames — overrides the artifact/page display name coming from the IG json (matched on artifact `reference.reference` or page key) |

   `defaultWorkgroup` is hard-coded to `au-fhir` in the template — see manual deltas below.

### Outputs

`HL7BallotTool/FHIR-au-{base,core,pd,erequesting,ps}.xml`, named from each change.json's
`speckey` so the generated filename already matches the deployed one.

A `current` version pointing at the CI build is always emitted first, plus the fixed
`artifactPageExtension` values (`-definitions`, `-examples`, `-mapping`) and the standard
`(NA)` / `(many)` pages.

**`DUPLICATE` marker:** if two artifacts resolve to the same display name the template writes a
bare `DUPLICATE` token into the `<artifact>` element. That is deliberately invalid XML — it is a
tripwire. Fix it by adding a `change` entry to disambiguate the name, then regenerate. Never ship
a file containing `DUPLICATE`.

## Standing rules

- **Deprecated material stays deprecated, forever.** Once an artifact, page or version is dropped
  from the IG it moves into `deprecate` / `pagedeprecate` / `deprecated:` in the change.json and
  stays there through every subsequent regeneration. It is never deleted.
- **Keys are permanent.** Any `key` referenced by a JIRA tracker item must never be removed or
  changed. Removing an artifact from the IG means deprecating it, not dropping it.
- Artifact `key` is derived mechanically: `reference.reference` with `/` → `-`
  (`StructureDefinition/au-condition` → `StructureDefinition-au-condition`). Page `key` is the
  `nameUrl` minus `.html`.

## Spec keys, files and work groups

Registered in `JIRA-Spec-Artifacts/xml/SPECS-FHIR.xml`; work group keys in `xml/_workgroups.xml`:

| Spec key (`speckey`) | File | IG inputs stem | `workgroup` |
| --- | --- | --- | --- |
| `au-base` | `FHIR-au-base.xml` | `hl7.fhir.au.base` | `au-fhir` |
| `au-core` | `FHIR-au-core.xml` | `hl7.fhir.au.core` | `au-core` |
| `au-pd` | `FHIR-au-pd.xml` | `hl7.fhir.au.pd` | `au-fhir` |
| `au-ps` | `FHIR-au-ps.xml` | `hl7.fhir.au.ps` | `au-core` |
| `au-erequesting` | `FHIR-au-erequesting.xml` | `hl7.fhir.au.ereq` | `au-ereq` |
| `au-encounter` | `FHIR-au-encounter.xml` | `hl7.fhir.au.encounter` | `au-fhir` |
| `au-terminology` | `FHIR-au-terminology.xml` | `hl7.au.terminology` | `au-terminology` |

Note eRequesting: the IG input files and canonical use the `ereq` stem while the registered JIRA
spec key is `au-erequesting`. `speckey` bridges that, so no rename is needed on copy.

**AU Encounter Record is CI-only** — no publication, so no `package-list.json`, no `versions[]`,
no `ballot`, and `defaultver` is `current` (the CI `<version>` the template always emits). Its IG
resource comes straight from the CI build:
<https://build.fhir.org/ig/hl7au/au-fhir-encounter/ImplementationGuide-hl7.fhir.au.encounter.json>.
`ballotUrl` is omitted from the output whenever `ballot` is absent or empty.

Its only IG resource so far is the `StructureDefinition/myobservation` scaffold left over from the
IG template, listed in `exclude[]` so it never becomes a permanent JIRA key. Drop that entry once
the IG defines real profiles; the 17 pages are the useful content in the meantime.

**AU Terminology (THO-AU) is CI-only too**, and breaks two conventions the other AU IGs follow:

- its package id is `hl7.au.terminology`, with **no `.fhir`** segment, so the input files are
  `ImplementationGuide-hl7.au.terminology.json` / `.change.json`. The CI resource is at
  <https://build.fhir.org/ig/hl7au/au-terminology/ImplementationGuide-hl7.au.terminology.json>
  (the `hl7.fhir.au.terminology` spelling 404s). The CI build also exposes
  `package.manifest.json`, which is how to confirm the package id for a new IG.
- its canonical is `http://terminology.hl7.org.au` — not under `hl7.org.au/fhir`.

It currently defines **no artifacts at all**: `definition.resource` is absent from the IG
resource, not merely empty. The template treats a missing `definition.resource` (and a missing
`definition.page`) as empty rather than failing, so the output is the 13 pages and nothing else.
`https://hl7.org.au/fhir/terminology/package-list.json` returns a 200 "Content Not Found" HTML
page, which is how hl7.org.au signals an unpublished IG — check that a package-list actually
parses as JSON before trusting the status code.

`HL7BallotTool/manual updates/` holds copies of deployed files that were edited by hand (including
`FHIR-au-signup.xml`, which has no generator inputs and is not yet in `SPECS-FHIR.xml`). Treat it
as a scratch/reference area, not the source of truth.

## Known manual deltas (generated → deployed)

There are no substantive deltas left. Every generated file now matches what is deployed in
element and attribute content, so a refresh can be copied straight across.

`FHIR-au-pd.xml` is the one file that still differs as *text*: the deployed copy was hand-written
(tab-indented `<version>` lines, no trailing spaces, `name="…"/>` on one artifact) while the
generator emits its usual style (4-space `<version>` indent with trailing spaces, a space before
every `/>`, and a trailing whitespace line with no newline at EOF — all of which the other six
deployed files already have). Verified identical with `remove_blank_text` parsing: same 45
elements, same attributes. `pd.change.json` no longer sets `ballot`, since 2.0.1 was never
balloted, so no `ballotUrl` is emitted.

## Command: "refresh all igs"

When Brett says **"refresh all igs"** (or "refresh the IGs", "refresh au-core", etc.), run the
whole update for all five AU IGs — or just the named ones — end to end, without asking for each
IG in turn. Steps:

### 1. Discover published versions

Each IG's publication history is `<canonical>/package-list.json` on hl7.org.au:

| Spec | package-list |
| --- | --- |
| au-base | <https://hl7.org.au/fhir/package-list.json> |
| au-core | <https://hl7.org.au/fhir/core/package-list.json> |
| au-pd | <https://hl7.org.au/fhir/pd/package-list.json> |
| au-ps | <https://hl7.org.au/fhir/ps/package-list.json> |
| au-erequesting | <https://hl7.org.au/fhir/ereq/package-list.json> |
| au-encounter | *(none — CI only; re-pull the IG resource from the CI build instead)* |
| au-terminology | *(none — CI only; re-pull the IG resource from the CI build instead)* |

Each `list[]` entry has `version`, `path`, `status` (`ci-build` | `draft` | `preview` | `ballot` |
`trial-use`), `date`, and optionally `current: true`. Reading rules:

- **Ignore the `status: ci-build` entry** — the template already emits a `current` version pointing
  at the CI build. (It also carries `current: true`; that is not the release flag.)
- **Newest publication** = the non-ci-build entry with the latest `date`. This is the version whose
  ImplementationGuide resource gets downloaded — a ballot or draft counts, because feedback is
  raised against ballot content and its artifacts must exist in the JIRA file.
- **`defaultver`** = the entry flagged `current: true` (excluding the ci-build one) — the milestone
  release, not the newest pre-release.
- **`ballot`** = the `path` of the most recent `status: ballot` entry. (au-pd has none — see the
  manual-deltas note.)
- Compare against the IG's current `change.json` to see what is new since the last refresh.

### 2. Download the ImplementationGuide resource

`<path>/ImplementationGuide-<package-id>.json` from the newest publication, e.g.
`https://hl7.org.au/fhir/7.0.0-ballot1/ImplementationGuide-hl7.fhir.au.base.json`. Save over
`HL7BallotTool/ImplementationGuide-hl7.fhir.au.<stem>.json`, keeping the existing filename stem
(`base`, `core`, `pd`, `ereq`, `ps`) — the template's arrays reference those names.

### 3. Update the change.json

Before overwriting the IG json, capture the previous artifact and page keys (the existing
`FHIR-au-*.xml` in this folder is the easiest source), then:

- **New versions** — insert every newly published version into `versions[]`, newest first, matching
  package-list order. Prefix with `deprecated:` any pre-release (`draft` / `preview` / `ballot` /
  `snapshot`) that a later publication supersedes. Released (`trial-use`) versions stay active
  unless deliberately retired — see 4.1.0 in au-base for a retired release.
- **`defaultver`, `ballot`** — set per the rules above.
- **Vanished artifacts** — any artifact in the previous output that is no longer in
  `definition.resource[]` moves into `deprecate[]` with its **exact previous `id`, `key` and
  `name`**. Never invent or re-derive the key.
- **Vanished pages** — same, into `pagedeprecate[]` with `{key, name}`.
- **Renames** — an artifact whose name changed but which must keep its old JIRA label goes in
  `change[]` as `{id, name}`. Also the fix for a `DUPLICATE` marker.
- Never delete an existing `deprecate` / `pagedeprecate` / `versions` entry. **Deprecated stays
  deprecated**, and keys are permanent (see Standing rules).

### 4. Regenerate, verify, report

Run the template (headless command above, or VS **Run Custom Tool**), then:

- grep the outputs for `DUPLICATE` — resolve any hit via `change[]` and regenerate.
- **Check for duplicate `key` values within each output.** If an artifact was deprecated in a past
  refresh and the IG has since reinstated it, the template emits it twice — once live from the IG
  and once from `deprecate[]` — and the `DUPLICATE` tripwire does *not* catch this, because it only
  compares names within the IG loop. Fix by removing the resurrected entry from `deprecate[]`; the
  key survives (now as a live artifact), so the permanence rule is not breached.
- Confirm no key present in the deployed file is missing from the new output.
- diff each `FHIR-au-*.xml` against `C:\repos\JIRA-Spec-Artifacts\xml\` and confirm every change is
  explained by the refresh, allowing for the known manual deltas.
- XSD validation against `JIRA-Spec-Artifacts/schemas/specification.xsd` will report
  `defaultWorkgroup` enum failures for `au-core` / `au-ereq` locally. That is a stale
  `schemas/workgroups.xsd` build artifact (last regenerated Nov 2024, before the AU work groups
  were added to `xml/_workgroups.xml`); the ant `validate` target rebuilds it from
  `_workgroups.xml` first, so upstream CI passes. The already-deployed files fail the same way.
- Report per IG: previous → new version loaded, `defaultver` / `ballot` changes, versions added,
  artifacts and pages newly deprecated, and anything that needed a judgement call.

Stop and ask before copying into `JIRA-Spec-Artifacts` or pushing — "refresh all igs" covers
steps 1–4 only. The copy/PR is the separate workflow below.

## Update workflow (end to end)

Steps 1–4 are what "refresh all igs" automates; 5–6 are the deployment half.

1. Replace `ImplementationGuide-hl7.fhir.au.<key>.json` with the IG resource from the newly
   published version.
2. Update that IG's `change.json`: add the new version to `versions[]` (marking superseded
   pre-release versions `deprecated:`), bump `defaultver`, adjust `ballot`, and move any artifacts
   or pages that disappeared from the IG into `deprecate` / `pagedeprecate`.
3. Run the T4 template in Visual Studio.
4. Diff the regenerated `FHIR-au-*.xml` against `C:\repos\JIRA-Spec-Artifacts\xml\` — confirm the
   only unexpected changes are the intended ones, and check for `DUPLICATE`.
5. Copy into `JIRA-Spec-Artifacts/xml/` (filenames already match), re-applying the manual deltas
   above where they still apply.
6. Commit both repos; branch in the `hl7au` fork and raise a PR against `HL7/JIRA-Spec-Artifacts`
   `master`. CI in that repo validates against `schemas/specification.xsd` and builds the JSON the
   Jira Elements Connect dropdowns consume — never edit `JIRA-Spec-Artifacts/json/` by hand.

The IG publisher itself flags drift ("The jira specification file appears to be out of date with
the versions, artifacts and pages currently defined in the IG") in `qa.html`, and can emit a
proposed XML file in the build directory — useful as a cross-check against what `ballot.tt`
produces.
