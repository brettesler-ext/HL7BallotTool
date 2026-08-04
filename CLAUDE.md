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

The template loops over five hard-coded IG/change pairs. Adding a new IG means editing the
`filenames` and `changes` arrays at the top of `ballot.tt`.

### Inputs (per IG, in `HL7BallotTool/`)

1. `ImplementationGuide-hl7.fhir.au.<key>.json` — the **ImplementationGuide resource from the
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
   | `ballot` | `ballotUrl` |
   | `defaultver` | `defaultVersion` |
   | `versions[]` | one `<version>` each, in order; prefix a value with `deprecated:` to emit `deprecated="true"` |
   | `deprecate[]` | `{id, key, name}` artifacts that no longer exist in the IG but must remain, emitted with `deprecated="true"` |
   | `pagedeprecate[]` | `{key, name}` same, for pages |
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

Note eRequesting: the IG input files and canonical use the `ereq` stem while the registered JIRA
spec key is `au-erequesting`. `speckey` bridges that, so no rename is needed on copy.

`HL7BallotTool/manual updates/` holds copies of deployed files that were edited by hand (including
`FHIR-au-signup.xml`, which has no generator inputs and is not yet in `SPECS-FHIR.xml`). Treat it
as a scratch/reference area, not the source of truth.

## Known manual deltas (generated → deployed)

As of the `speckey`/`workgroup` change, `FHIR-au-core.xml`, `FHIR-au-ps.xml` and
`FHIR-au-erequesting.xml` regenerate byte-identical to what is deployed. Two deltas remain — diff
before copying so they are not clobbered:

- `FHIR-au-pd.xml` — deployed has **no** `ballotUrl` attribute, and its `<version>` lines are
  formatted differently. `pd.change.json` sets `ballot` to the 2.0.1 release URL, which the
  template emits unconditionally. Strip `ballotUrl` on copy (or drop the `ballot` field and make
  the attribute conditional in the template).
- `FHIR-au-base.xml` — deployed lists `StructureDefinition/ethnicity` at the end of the artifact
  block rather than in IG order. Cosmetic; order is not significant.

## Update workflow

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
