# GitHub Build-from-Source Buildpack

A Heroku buildpack that clones a Java project from a GitHub repository, builds it, and makes the compiled JAR file available in your application's runtime classpath.

## Features

- 🔨 Clones and builds Java projects from GitHub
- 🏗️ Supports Maven and Gradle (auto-detection)
- 🔐 Works with private repositories
- 🎯 Configurable via simple config file
- 🌿 Clone any branch you specify
- 💾 Caches build tools (Maven, Gradle, JDK) for faster builds
- 🚀 Optionally runs JAR as background process
- ♻️ Automatically adds built JAR to runtime classpath

## Usage

### 1. Set GitHub Token (Required for Private Repositories)

If your GitHub repository is private, you need to provide a GitHub Personal Access Token:

```bash
# Create a GitHub token at: https://github.com/settings/tokens
# The token needs "repo" scope to access private repositories

# Set it as a Heroku config variable
heroku config:set GITHUB_TOKEN=ghp_your_token_here
```

**Note:** For public repositories, the token is optional but recommended.

### 2. Create Configuration File

In your application root, create a `.buildpack-config` file:

```bash
# GitHub repository owner (required)
GITHUB_REPO_OWNER="your-github-username"

# GitHub repository name (required)
GITHUB_REPO_NAME="your-repo-name"

# Branch to clone (optional, defaults to "main")
BRANCH="main"

# Build tool (optional, defaults to "auto")
BUILD_TOOL="auto"

# JAR name pattern (optional)
JAR_NAME_PATTERN="*-all.jar"

# Auto-start as background process (optional)
AUTO_START="false"

# JVM options if auto-starting (optional)
JVM_OPTIONS="-Xmx512m"
```

**Example for a Gradle project:**

```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-java-app"
BRANCH="main"
BUILD_TOOL="gradle"
JAR_NAME_PATTERN="my-java-app-*.jar"
AUTO_START="true"
JVM_OPTIONS="-Xmx512m -Dserver.port=8081"
```

**Example for a Maven project:**

```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-maven-app"
BRANCH="develop"
BUILD_TOOL="maven"
```

### 3. Add Buildpack to Your Heroku App

```bash
# Using the buildpack from GitHub
heroku buildpacks:add https://github.com/YOUR_USERNAME/classic-buildpack.git#build-from-source

# Or set it as the primary buildpack
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git#build-from-source

# If you also need Java runtime for your main app, add it as well
heroku buildpacks:add heroku/java
```

### 4. Deploy

```bash
git push heroku main
```

## How It Works

1. **Detection Phase**: The buildpack checks for the presence of `.buildpack-config` in your app root
2. **Compile Phase**: 
   - Reads the `GITHUB_TOKEN` from Heroku config vars (if set)
   - Reads configuration from `.buildpack-config`
   - Installs GitHub CLI for authentication
   - Clones the specified repository and branch
   - Detects the build tool (Maven or Gradle) if set to "auto"
   - Installs necessary build tools (cached for speed)
   - Builds the project (skips tests for faster builds)
   - Finds the compiled JAR file
   - Places the JAR in `vendor/jars/` directory
3. **Runtime Phase**: The JAR is automatically added to the `CLASSPATH` environment variable

## Authentication

### GitHub Personal Access Token

For **private repositories**, you must provide a GitHub Personal Access Token:

1. **Create a token**: Go to https://github.com/settings/tokens/new
   - Give it a descriptive name (e.g., "Heroku Buildpack")
   - Select the `repo` scope (Full control of private repositories)
   - Click "Generate token"
   - Copy the token (it starts with `ghp_`)

2. **Set it in Heroku**:
   ```bash
   heroku config:set GITHUB_TOKEN=ghp_your_token_here
   ```

For **public repositories**, authentication is optional but recommended.

## Configuration Options

| Variable | Required | Description | Default | Example |
|----------|----------|-------------|---------|---------|
| `GITHUB_REPO_OWNER` | Yes | GitHub repository owner/organization | - | `apache` |
| `GITHUB_REPO_NAME` | Yes | GitHub repository name | - | `commons-lang` |
| `BRANCH` | No | Branch to clone | `main` | `develop`, `feature/new-api` |
| `BUILD_TOOL` | No | Build tool to use | `auto` | `maven`, `gradle`, `auto` |
| `JAR_NAME_PATTERN` | No | Pattern to find JAR after build | (first JAR) | `*-all.jar`, `myapp-*.jar` |
| `AUTO_START` | No | Auto-start JAR as background process | `false` | `true` or `false` |
| `JVM_OPTIONS` | No | JVM options when auto-starting | - | `-Xmx512m -Dport=8081` |

## Build Tool Detection

When `BUILD_TOOL="auto"` (the default):
- Looks for `pom.xml` → Uses Maven
- Looks for `build.gradle` or `build.gradle.kts` → Uses Gradle
- If neither found → Fails with error

## JAR Detection After Build

After building, the buildpack looks for the compiled JAR:

### Maven
- Searches in `target/` directory
- Excludes `*-sources.jar` and `*-javadoc.jar`
- If `JAR_NAME_PATTERN` is set, matches that pattern
- Otherwise, takes the first JAR found

### Gradle
- Searches in `build/libs/` directory
- Excludes `*-plain.jar` and `*-sources.jar`
- If `JAR_NAME_PATTERN` is set, matches that pattern
- Otherwise, takes the first JAR found (usually the fat JAR)

## Running the Built JAR

### Option 1: As a Library (Default)

By default, the JAR is added to `CLASSPATH` and available for your application to import:

```bash
# .buildpack-config
AUTO_START="false"
```

Your application can then use classes from the JAR.

### Option 2: Auto-Start as Background Process

Set `AUTO_START="true"` to automatically run the JAR in the background:

```bash
# .buildpack-config
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="myrepo"
AUTO_START="true"
JVM_OPTIONS="-Xmx512m -Dserver.port=8081"
```

The JAR will automatically start in the background when your dyno boots. Logs are written to `$HOME/logs/github-jar.log`.

### Option 3: Worker Dyno

Create a `Procfile` to run the JAR as a separate worker process:

```
web: <your-main-app-command>
worker: java -jar $HOME/vendor/jars/*.jar
```

Then scale:
```bash
heroku ps:scale worker=1
```

### Option 4: Executable in Procfile

Run the JAR as your main web process:

```
web: java -jar $HOME/vendor/jars/*.jar
```

## JAR Location at Runtime

The built JAR will be available at:
- File path: `$HOME/vendor/jars/YOUR_JAR_FILENAME.jar`
- Automatically included in `$CLASSPATH`

## Caching

The buildpack caches the following to speed up subsequent builds:
- JDK (OpenJDK 11)
- Maven
- Gradle
- GitHub CLI

The repository is cloned fresh on each build to ensure you get the latest code from the branch.

## Build Process

### Maven Projects
Runs: `mvn clean package -DskipTests`
- Skips tests for faster builds
- Creates JAR in `target/` directory

### Gradle Projects
Runs: `./gradlew clean build -x test` (or `gradle` if wrapper not available)
- Skips tests for faster builds
- Creates JAR in `build/libs/` directory

## Examples

### Example 1: Build Gradle Project from Main Branch

```bash
GITHUB_REPO_OWNER="mycompany"
GITHUB_REPO_NAME="backend-service"
BRANCH="main"
BUILD_TOOL="gradle"
AUTO_START="true"
JVM_OPTIONS="-Xmx768m"
```

### Example 2: Build Maven Project from Develop Branch

```bash
GITHUB_REPO_OWNER="myteam"
GITHUB_REPO_NAME="api-gateway"
BRANCH="develop"
BUILD_TOOL="maven"
JAR_NAME_PATTERN="api-gateway-*.jar"
AUTO_START="false"
```

### Example 3: Build with Auto-Detection (Public Repo)

```bash
GITHUB_REPO_OWNER="apache"
GITHUB_REPO_NAME="commons-cli"
BRANCH="master"
BUILD_TOOL="auto"
```

## Troubleshooting

### Authentication Issues for Private Repositories

**Error:** `fatal: could not read Username` or `403` when cloning

**Solution:**
1. Make sure you've set the `GITHUB_TOKEN` config var:
   ```bash
   heroku config:set GITHUB_TOKEN=ghp_your_token_here
   ```
2. Verify your token has the `repo` scope
3. Check that the token hasn't expired

### Build Failures

**Error:** `Build failed` during Maven or Gradle build

**Solution:**
- Check the build logs for specific errors
- Ensure the project builds successfully locally
- Verify the branch name is correct
- Check if the project has dependencies on external systems
- Consider if tests are failing (they're skipped but some build configs still run them)

### JAR Not Found After Build

**Error:** `Could not find built JAR file`

**Solution:**
1. Check if the build actually produces a JAR (some projects only produce WARs or other artifacts)
2. Specify `JAR_NAME_PATTERN` to match your specific JAR name
3. Look at the "Available JARs" list in the build output

### Build Tool Not Detected

**Error:** `Could not detect build tool`

**Solution:**
Set `BUILD_TOOL` explicitly in `.buildpack-config`:
```bash
BUILD_TOOL="maven"  # or "gradle"
```

### Java Version Issues

The buildpack installs OpenJDK 11 by default. If your project requires a different Java version:
- Consider modifying the buildpack or using the official Heroku Java buildpack as a base
- Or ensure your project is compatible with Java 11

## Advanced: Multiple Repositories

To build JARs from multiple repositories, you can:
1. Use multiple buildpacks (if Heroku supports it for your stack)
2. Modify the config to support arrays
3. Or create separate builds and combine them

## Development

### Testing Locally

You can test the buildpack locally:

```bash
# Create a test app directory
mkdir test-app
cd test-app

# Create .buildpack-config
cat > .buildpack-config <<EOF
GITHUB_REPO_OWNER="your-org"
GITHUB_REPO_NAME="your-repo"
BRANCH="main"
BUILD_TOOL="gradle"
EOF

# Run buildpack
/path/to/classic-buildpack/bin/detect .
/path/to/classic-buildpack/bin/compile . /tmp/cache /tmp/env
```

## Comparison: Release vs Build-from-Source

This buildpack has two versions:

### Main Branch: Download from Release
- ✅ Faster (no build time)
- ✅ Less dependencies
- ✅ Consistent artifacts
- ❌ Requires manual release creation
- ❌ Can't use unreleased code

### Build-from-Source Branch: Clone and Build
- ✅ Always uses latest code from branch
- ✅ No need to create releases
- ✅ Can build from any branch
- ❌ Slower (build time)
- ❌ Requires build tools

## License

MIT

## Contributing

Pull requests are welcome! Please feel free to submit issues and enhancement requests.
