---
name: spark-pombump-testing
description: Test the pombump files in spark-base/spark-<major.minor>/ (pombump-properties.yaml and pombump-deps.yaml) against the upstream Apache Spark source tag for each major Spark version (3.2, 3.3, 3.4, 3.5, 4.0). Clones apache/spark at tag v<version>, applies the configured source patches, then runs pombump on pom.xml. Use when asked to test, validate, or dry-run pombump configuration for a Spark version.
user-invocable: true
argument-hint: "[spark-version, e.g. 4.0.5 or 3.5.6]"
---

# Spark pombump testing

Checks that the pombump configuration in `spark-base/spark-<major.minor>/` works on the matching upstream Apache Spark release. The steps mirror the CI action [patch-pombump](../../actions/patch-pombump/action.yml): clone the tag, apply the patches, run pombump on the root `pom.xml`.

Use the existing [spark-patch-testing](../spark-patch-testing/SKILL.md) skill if you only need to check that source patches apply. This skill also runs pombump.

## When to use

- The user asks whether pombump works for a Spark version, or asks to validate `pombump-*.yaml`.
- `pombump-properties.yaml` or `pombump-deps.yaml` changed in `spark-base/`.
- A CI run failed in the `Install and Run pombump` step.

## Versions to cover

Test one representative tag per major version. These are the latest tags configured in `.build/pre-build-patch-pombump.yml`:

| Major | Tag to test | Patch dir | Config check |
|---|---|---|---|
| 3.2 | `v3.2.4` | `spark-base/spark-3.2/` | Java 11, Python 3.9 |
| 3.3 | `v3.3.4` | `spark-base/spark-3.3/` | Java 17, Python 3.10 |
| 3.4 | `v3.4.2` | `spark-base/spark-3.4/` | Java 17, Python 3.11 |
| 3.5 | `v3.5.6` | `spark-base/spark-3.5/` | Java 17, Python 3.11 |
| 4.0 | `v4.0.5` | `spark-base/spark-4.0/` | Java 17, Python 3.11 |

Git tags use the `v` prefix (`v4.0.5`). The `<spark version>` you pass is `4.0.5`, and the tag is `v4.0.5`.

Do not use `spark-base/spark-4.0/other/`. CI does not read it.

## Prerequisites

- `git`, `bash` (Git Bash on Windows; put Git's `usr\bin` first on `PATH` so GNU `patch` is used), GNU `patch`, `yq` (mikefarah v4), and `go` 1.22 or later.
- Network access to `https://github.com/apache/spark.git` and `https://github.com/chainguard-dev/pombump.git`. These match CI.

## Step 1: Set variables

```bash
SPARK_VERSION=4.0.5                                    # the version to test
POMBUMP_VERSION=v0.0.16                                # same default as the CI action
REPO=$(git rev-parse --show-toplevel)
MINOR=$(echo "$SPARK_VERSION" | awk -F. '{print $1"."$2}')   # 4.0
PATCH_DIR="$REPO/spark-base/spark-$MINOR"
PROPERTIES_FILE="$PATCH_DIR/pombump-properties.yaml"
DEPS_FILE="$PATCH_DIR/pombump-deps.yaml"
WORK=$(mktemp -d)
```

Check that both files exist before going further:

```bash
[[ -f "$PROPERTIES_FILE" && -f "$DEPS_FILE" ]] || { echo "Missing pombump config in $PATCH_DIR"; exit 1; }
```

## Step 2: Install pombump

Clone the same release tag CI uses and build it outside the repo. This is the path that was verified:

```bash
export GOBIN="$WORK/bin"
export PATH="$GOBIN:$PATH"
git clone --depth 1 --branch "$POMBUMP_VERSION" https://github.com/chainguard-dev/pombump.git "$WORK/pombump-src"
(cd "$WORK/pombump-src" && go build -o "$GOBIN/pombump")
pombump --help >/dev/null 2>&1 && echo "pombump OK"
```

Alternative if the module resolves through the proxy: `go install github.com/chainguard-dev/pombump@"$POMBUMP_VERSION"`.

## Step 3: Clone the Spark tag

```bash
git clone --depth 1 --branch "v$SPARK_VERSION" https://github.com/apache/spark.git "$WORK/spark-source"
cd "$WORK/spark-source"
git describe --tags   # must print v4.0.5 (or the requested version)
```

## Step 4: Apply the source patches

Read the patch list for the exact version from `.build/pre-build-patch-pombump.yml`. Apply the patches in that order. If the list is empty (for example `patch_files: []`), skip this step.

```bash
PATCHES=$(yq -r ".controls[] | select(.spark_version == \"$SPARK_VERSION\") | .patch_files[]" \
  "$REPO/.build/pre-build-patch-pombump.yml")
for patch in $PATCHES; do
  PATCH_PATH="$PATCH_DIR/$patch"
  [[ -f "$PATCH_PATH" ]] || { echo "Patch file $PATCH_PATH not found!"; exit 1; }
  echo "Applying patch $patch"
  patch -p1 < "$PATCH_PATH" || exit 1
done
```

If the version is missing from the YAML, the command prints nothing. Tell the user that the version has no patch config, and stop.

## Step 5: Run pombump

Run from the Spark source root, where `pom.xml` is. The command is the one used by CI. pombump logs each property and dependency it checks to stderr, so keep that log for Step 6:

```bash
pombump pom.xml --properties-file "$PROPERTIES_FILE" --patch-file "$DEPS_FILE" > pom.xml.patched 2> "$WORK/pombump.log"
```

Do not overwrite `pom.xml` until pombump has succeeded. A failed run still creates an empty `pom.xml.patched`, and the CI `mv` would replace `pom.xml` with it. Use this guarded form:

```bash
if pombump pom.xml --properties-file "$PROPERTIES_FILE" --patch-file "$DEPS_FILE" > pom.xml.patched 2> "$WORK/pombump.log" \
   && [[ -s pom.xml.patched ]]; then
  mv pom.xml.patched pom.xml
  echo "pombump OK"
else
  echo "pombump FAILED"; exit 1
fi
```

## Step 6: Validate the result

1. The POM is well-formed XML:
   ```bash
   python -c "import xml.etree.ElementTree as ET; ET.parse('pom.xml'); print('pom.xml well-formed')"
   ```
2. Check the pombump log, not the diff:
   ```bash
   grep -E "Patching (property|DM dep|dep)" "$WORK/pombump.log"
   ```
   Each line has the form `Patching property: X from A to B`. `B` must equal the value in the YAML. `A == B` is normal when the Spark tag already uses that version. Compare the line count with the number of entries in the YAML files. A YAML entry with no matching log line means the name is wrong or the property does not exist in that Spark version.
   
   Do not use `git diff` to judge the result. pombump re-serializes the POM. It drops the license comment, reorders elements such as `<parent>`, `<scm>`, and `<organization>`, and changes indentation. So `git diff` shows almost every line as changed. A whitespace-insensitive diff (`git diff -w --ignore-blank-lines`) still shows reordered blocks.
3. Optional, only when the user asks. If Maven is installed, check that the POM resolves:
   ```bash
   mvn -q -N validate
   ```
   Dependency resolution is slow.

## Step 7: Report and clean up

For each major version tested, report:

| Field | Value |
|---|---|
| Spark tag | `v<version>` |
| Patches applied | list, or `none` |
| pombump | exit code and `pombump OK` / `FAILED` |
| Properties patched | count of `Patching property` lines in the log, and how many of them have `A != B` |
| Dependencies patched | count of `Patching DM dep` lines in the log, compared with the entries in `pombump-deps.yaml` |

If a step fails, show the relevant output and stop. Do not edit the YAML files in the repo unless the user asks.

Clean up:

```bash
rm -rf "$WORK"
```

## Gotchas

- `pombump` writes the new POM to stdout. Check the exit code and the file size before `mv`. This prevents an empty `pom.xml` from replacing the real one.
- Patches are applied before pombump, in the order from the `.build` YAML. A patch can fail only because an earlier patch changed the same lines.
- `patch` can print a warning and still exit 0. For example, `patch unexpectedly ends in middle of line` appeared during the v3.2.4 run, and the run still passed. Read the `patch` output, not only the exit code, when a result looks wrong.
- Dependencies in `pombump-deps.yaml` may use `${...}` references, such as `${commons-compress.version}`. Those must match a property in `pombump-properties.yaml` for the same major version. Otherwise pombump fails.
- Each Spark major has different property names, so use the YAML for that major only.
- `spark-base/spark-4.0/other/` has its own pombump files, but CI does not use them. Do not test them as part of this skill unless the user asks.
- pombump output is not a minimal edit. It re-serializes the whole POM, so the diff against the Apache tag touches most lines. Judge the result from the log (Step 6), not from `git diff`.
- The clone is a full network download of the tag, and it can take several minutes.
- On Windows, run the commands in Git Bash, not PowerShell. The variables and heredocs above are Bash syntax.
- The skill only tests configuration. It does not build the Spark distribution or the Docker image.

## Resources

- CI action: `.github/actions/patch-pombump/action.yml`
- Patch config: `.build/pre-build-patch-pombump.yml`
- pombump configs: `spark-base/spark-<major.minor>/pombump-properties.yaml`, `spark-base/spark-<major.minor>/pombump-deps.yaml`
- Existing patch-only check: `.github/skills/spark-patch-testing/SKILL.md`
- pombump project: https://github.com/chainguard-dev/pombump
