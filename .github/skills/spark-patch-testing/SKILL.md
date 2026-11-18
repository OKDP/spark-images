---
name: spark-patch-testing
description: Test the Spark patch files in spark-base/ against the upstream Apache Spark source tag for a given version (3.2, 3.3, 3.4, 3.5, 4.0). Clones apache/spark at tag v<version>, then applies the patches listed for that version in .build/pre-build-patch-pombump.yml with `patch -p1 < "$PATCH_PATH" || exit 1`. Use when asked to test, validate, or check whether spark-base patches apply to a Spark version.
user-invocable: true
argument-hint: "[spark-version, e.g. 4.0.5 or 3.5.6]"
---

# Spark patch testing

Checks that the patches in `spark-base/spark-<major.minor>/` apply cleanly to the matching upstream Apache Spark release tag. This is the same clone-and-patch sequence the CI action [patch-pombump](../../actions/patch-pombump/action.yml) runs before pombump.


## When to use

- The user asks whether patches apply to a Spark version, or asks to validate `spark-base` patches.
- A new Spark version or patch was added and needs checking before a build.
- A patch failed in CI (`Patch and POMBump Spark Source` step).

## Prerequisites

- `git`, `bash` (not `sh`: the loop uses `[[`), GNU `patch`, and `yq`. `jq` is not needed.
- `yq`: either the mikefarah Go build (v4) or the Python `yq` wrapper works with the filter in step 1. The commands were tested with mikefarah v4.
- Network access to `https://github.com/apache/spark.git`.

Install on Debian/Ubuntu:
```bash
sudo apt-get install -y git patch
sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 && sudo chmod +x /usr/local/bin/yq
```
Use `yq_linux_arm64` on ARM. On Fedora/RHEL, use `dnf install git patch`. On Windows, use Git Bash, and make sure Git's `usr\bin` folder is first on `PATH` so `patch` is GNU patch.

## Step 1: Pick the version and the patch list

1. Take the version from the user. It must be a full version such as `4.0.5`. Derive the patch directory from the major.minor:
   ```bash
   SPARK_VERSION=4.0.5
   MINOR=$(echo "$SPARK_VERSION" | awk -F. '{print $1"."$2}')   # 4.0
   REPO=$(git rev-parse --show-toplevel)
   PATCH_DIR="$REPO/spark-base/spark-$MINOR"
   ```
2. Read the patch list for that exact version from `.build/pre-build-patch-pombump.yml`. Apply the patches in the order they are listed there.
   ```bash
   yq -r ".controls[] | select(.spark_version == \"$SPARK_VERSION\") | .patch_files[]" \
     "$REPO/.build/pre-build-patch-pombump.yml"
   ```
   If this prints nothing, the version has no patch config. Tell the user, and stop.

Versions currently configured: 3.2.1–3.2.4, 3.3.1–3.3.4, 3.4.1–3.4.2, 3.5.1, 3.5.6, 4.0.5. Always check the YAML, because it is the source of truth.

Do not use the `spark-base/spark-4.0/other/` folder. CI does not read it.

## Step 2: Clone the Spark tag

Use a shallow clone of the tag, as CI does. Work in a temporary directory, not in the repo.

```bash
WORK=$(mktemp -d)
git clone --depth 1 --branch "v$SPARK_VERSION" https://github.com/apache/spark.git "$WORK/spark-source"
cd "$WORK/spark-source"
```

Confirm the checkout: `git describe --tags` should print `v4.0.5` (or the requested version).

## Step 3: Apply the patches

Run the patches in the order from step 1. Each one uses the command below, which stops the run on the first failure:

```bash
for patch in $(yq -r ".controls[] | select(.spark_version == \"$SPARK_VERSION\") | .patch_files[]" \
      "$REPO/.build/pre-build-patch-pombump.yml"); do
  PATCH_PATH="$PATCH_DIR/$patch"
  if [[ ! -f "$PATCH_PATH" ]]; then
    echo "Patch file $PATCH_PATH not found!"; exit 1
  fi
  echo "Applying patch $patch"
  patch -p1 < "$PATCH_PATH" || exit 1
done
```

The missing-file check is an addition to the CI command. CI only prints "not found" and continues, so a typo in the YAML would pass there.

## Step 4: Check for fuzz and rejects

GNU `patch` applies a hunk even when its context has drifted, as long as it finds a close match. It reports this as `fuzz` or `offset`. The run still exits 0, so the apply can pass while the patch is fragile.

For an exact check of each patch, run this before applying it:

```bash
git apply --check --unidiff-zero "$PATCH_PATH"   # exit 0 = exact match
```

Report to the user, per patch:

| Result | Meaning |
|---|---|
| `Hunk #n succeeded` with no fuzz/offset | Exact. |
| `succeeded at N (offset X)` | Applies, but the line moved. Works, but fragile. |
| `with fuzz N` | Applies, but context changed. Recommend regenerating the patch. |
| `FAILED` / `.rej` file | Does not apply. The run stops with exit 1. |

## Step 5: Report and clean up

- Report the list of patches, each with its result, and the Spark tag tested.
- If a patch failed, show the failing hunk from the `.rej` file in the work directory. Do not change the patch in the repo unless the user asks.
- Remove the temporary directory: `rm -rf "$WORK"`.

## Gotchas

- `patch -p1` returns 0 for fuzzy applies. A passing run does not mean the patch is exact. Check the output.
- Patches are applied in sequence on the same tree. A patch can fail only because an earlier one changed the same lines. Check the order in the YAML.
- Spark 4.0.x has no `aws-sdk-bump.patch` in its config, and 3.5 and later have no `make-distribution.patch`. Use only the list from the YAML.
- Patch files in a Windows checkout may have CRLF line endings. If a patch fails with a format error, convert a temporary copy to LF before testing. Do not commit the conversion unless asked. On Linux or WSL, check the repo with `git status` first, because `core.autocrlf` can change line endings.
- On Linux, `REPO` must point to a checkout that the user can read. If the repo is on a Windows drive under WSL, use the `/mnt/<drive>/...` path.
- The clone is a full network download of the Spark tag. It can take several minutes on a slow link.
- This skill only checks that the patches apply. It does not run pombump or build the image.
- The clone uses GitHub (`apache/spark`), the same source as CI. This is intentional.

## Resources

- CI action: `.github/actions/patch-pombump/action.yml`
- Patch config: `.build/pre-build-patch-pombump.yml`
- Patch files: `spark-base/spark-<major.minor>/`
