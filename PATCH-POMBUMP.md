## Patching and Dependency Management System

This project uses a patching system to apply security fixes and dependency updates to Spark source code before building Docker images. The system supports both traditional source code patches and automated dependency version management via pombump.

### How the System Works

The patching system consists of three main components:

#### 1. Configuration File (`.build/pre-build-patch-pombump.yml`)

This file defines which Spark versions should receive processing and what type of modifications to apply:

```yaml
controls:
  # Spark version with source patches + pombump
  - spark_version: "3.2.4"
    python_version: "3.9"
    java_version: "11" 
    hadoop_version: "3.3.6"
    patch_files:
      - log4j-fix.patch
      
  # Spark version with pombump only (no source patches)
  - spark_version: "3.4.1"
    python_version: "3.11"
    java_version: "17"
    hadoop_version: "3.3.6"
    patch_files: []
```

**Key Behavior:**
- **Present in config**: Spark source is downloaded, patches applied (if any), pombump runs, patched source used for build
- **Not in config**: Original Spark distribution used without modifications

#### 2. Patch Files Structure

Patch files are organized by Spark minor version in the following structure:

```
spark-base/
  spark-3.2/
    ├── log4j-fix.patch           # Source code patches
    ├── pombump-properties.yaml   # Property version updates
    └── pombump-deps.yaml        # Dependency version updates
  spark-3.4/
    ├── pombump-properties.yaml   # Only pombump files (no source patches)
    └── pombump-deps.yaml
```

#### 3. Processing Workflow

The patching happens automatically during the Docker build process via the `.github/actions/patch-pombump/action.yml`:

1. **Configuration Check**: The system checks if the Spark version exists in `.build/pre-build-patch-pombump.yml`

2. **Source Download**: If found, the system downloads the corresponding Spark source code:
   ```bash
   git clone --depth 1 --branch v${SPARK_VERSION} https://github.com/apache/spark.git
   ```

3. **Source Patches** (if `patch_files` is not empty): Traditional patch files are applied using:
   ```bash
   patch -p1 < patch-file.patch
   ```

4. **Dependency Updates** (always runs if config exists): Uses `pombump` tool for safe Maven POM updates:
   ```bash
   pombump pom.xml --properties-file pombump-properties.yaml --patch-file pombump-deps.yaml
   ```

5. **Build Context**: The modified source files are copied to the Docker build context

### Processing Types

#### Source Code Patches + Pombump

For older Spark versions that need both source fixes and dependency updates:

```yaml
- spark_version: "3.2.4"
  patch_files:
    - log4j-fix.patch  # Source code fix
  # + pombump updates dependencies
```

#### Pombump Only

For newer Spark versions that only need dependency updates:

```yaml
- spark_version: "3.4.1"  
  patch_files: []  # No source patches needed
  # + pombump updates dependencies
```

#### No Processing

Spark versions not listed in the configuration file use the original distribution without modifications.

### POMBump Dependency Management

The system uses [pombump](https://github.com/chainguard-dev/pombump) to safely update Maven POM dependencies to secure versions without breaking builds.

**Why Pombump?**
- ✅ **Safe updates**: Validates dependency compatibility  
- ✅ **Security focused**: Updates to latest secure versions
- ✅ **Build reliability**: Prevents dependency conflicts
- ✅ **Automated**: No manual POM editing required

#### Pombump Configuration Files

**pombump-properties.yaml** - Updates Maven property versions:
```yaml
properties:
  - property: log4j.version
    value: "2.25.0"      # Security fix for Log4Shell
  - property: fasterxml.jackson.version
    value: "2.14.2"      # Jackson security updates  
  - property: netty.version
    value: "4.1.117.Final" # Network security fixes
  - property: guava.version
    value: "33.4.8-jre"   # Google Guava updates
```

**pombump-deps.yaml** - Updates specific dependency versions:
```yaml
patches:
  - groupId: org.apache.commons
    artifactId: commons-compress
    version: "1.27.1"     # Archive security fixes
    scope: import
    type: jar
  - groupId: io.netty
    artifactId: netty-all  
    version: "4.1.117.Final"
    scope: import
    type: jar
```

#### When Pombump Runs

Pombump automatically runs for **any Spark version present** in `.build/pre-build-patch-pombump.yml`, regardless of whether it has source patches:

- **With patches**: `patch_files: [log4j-fix.patch]` → Source patches + pombump
- **Without patches**: `patch_files: []` → Pombump only  
- **Not in config**: No processing (uses original Spark distribution)

This ensures all configured Spark versions get the latest secure dependency versions.

### Docker Build Integration

In the Dockerfile, patched sources are used when available:

```dockerfile
if [ -d "/tmp/build-context/patched-spark-files" ] && [ "$(ls -A /tmp/build-context/patched-spark-files 2>/dev/null)" ]; then
    echo "=== USING PATCHED SPARK SOURCE FILES ===";
    cp -r /tmp/build-context/patched-spark-files spark-source;
else
    echo "=== CLONING SPARK REPOSITORY ===";
    git clone --depth 1 --branch v${SPARK_VERSION} ${SPARK_REPO_URL} spark-source;
fi
```

### Benefits

- **Security**: Automatically applies security fixes and updates vulnerable dependencies
- **Compatibility**: Updates dependencies for better cloud and Kubernetes compatibility  
- **Flexibility**: Support for both source patches and dependency-only updates
- **Automation**: No manual intervention required during builds
- **Reliability**: POMBump ensures safe dependency updates without breaking builds
- **Selective**: Only processes Spark versions that need modifications