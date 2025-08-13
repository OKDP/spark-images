## Patching System

This project uses a patching system to apply security fixes and dependency updates to Spark source code before building Docker images.

### How Patching Works

The patching system consists of three main components:

#### 1. Patch Configuration (`.build/pre-build-patch.yml`)

This file defines which patches to apply for specific Spark/Java/Python/Hadoop version combinations:

```yaml
controls:
  - spark_version: "3.2.4"
    python_version: "3.9"
    java_version: "11"
    hadoop_version: "3.3.6"
    patch_files:
      - log4j-fix.patch
```

#### 2. Patch Files Structure

Patch files are organized by Spark minor version in the following structure:

```
spark-base/
  spark-3.2/
    ├── log4j-fix.patch
    └── pombump-properties.yaml
```

#### 3. Patch Application Process

The patching happens automatically during the Docker build process via the `.github/actions/patch/action.yml`:

1. **Version Matching**: The system matches your build parameters against the controls in `.build/pre-build-patch.yml`

2. **Source Download**: If patches are found, the system downloads the corresponding Spark source code:
   ```bash
   git clone --depth 1 --branch v${SPARK_VERSION} https://github.com/apache/spark.git
   ```

3. **Patch Application**: Standard patch files are applied using:
   ```bash
   patch -p1 < patch-file.patch
   ```

4. **Dependency Updates**: For dependency version bumps, the system uses `pombump` tool:
   ```bash
   pombump pom.xml --properties-file pombump-properties.yaml --patch-file pombump-deps.yaml
   ```

5. **Build Context**: The patched source files are copied to the Docker build context for use during image building

### Patch Types

#### Security Patches
- **log4j-fix.patch**: Addresses Log4j vulnerabilities in older Spark versions

#### Dependency Updates  

#### Build Fixes

### POMBump Integration

For complex dependency updates, the system uses [pombump](https://github.com/chainguard-dev/pombump) to safely update Maven POM files:

- **pombump-properties.yaml**: Defines property version updates
- **pombump-deps.yaml**: Defines dependency version updates

Example pombump-properties.yaml:
```yaml
- property: guava.version
  value: "33.4.8-jre"
- property: netty.version  
  value: "4.1.117.Final"
```

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

### Adding New Patches

To add a new patch:

1. **Create the patch file** in the appropriate `spark-base/spark-X.Y/` directory
2. **Add the configuration** to `.build/pre-build-patch.yml`:
   ```yaml
   - spark_version: "3.5.1"
     python_version: "3.11" 
     java_version: "17"
     hadoop_version: "3.3.6"
     patch_files:
       - your-new-patch.patch
   ```
3. **Test the build** to ensure the patch applies correctly

### Benefits

- **Security**: Automatically applies security fixes to older Spark versions
- **Compatibility**: Updates dependencies for better cloud and Kubernetes compatibility  
- **Automation**: No manual intervention required during builds
- **Flexibility**: Different patches for different version combinations
- **Reliability**: POMBump ensures safe dependency updates without breaking builds

This patching system ensures that all built Spark images include the latest security fixes and compatibility improvements, even when building from older Spark source versions.