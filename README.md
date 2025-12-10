# GitHub JAR Downloader Buildpack

A Heroku buildpack that downloads a Java JAR file from a GitHub repository's releases and makes it available in your application's runtime classpath.

## Features

- 📦 Downloads JAR files from GitHub releases
- 🎯 Configurable via simple config file
- 🚀 Supports latest release or specific release tags
- 💾 Caches downloaded JARs for faster subsequent builds
- ♻️ Automatically adds JAR to runtime classpath

## Usage

### 1. Set GitHub Token (Required for Private Repositories)

If your GitHub repository is private, you need to provide a GitHub Personal Access Token:

```bash
# Create a GitHub token at: https://github.com/settings/tokens
# The token needs "repo" scope to access private repositories

# Set it as a Heroku config variable
heroku config:set GITHUB_TOKEN=ghp_your_token_here
```

**Note:** For public repositories, the token is optional but recommended to avoid rate limiting.

### 2. Create Configuration File

### 2. Create Configuration File

In your application root, create a `.buildpack-config` file:

```bash
# GitHub repository owner (required)
GITHUB_REPO_OWNER="your-github-username"

# GitHub repository name (required)
GITHUB_REPO_NAME="your-repo-name"

# JAR filename to download (required)
JAR_FILENAME="your-library.jar"

# Release tag to download from (optional, defaults to "latest")
RELEASE_TAG="latest"
```

**Example:**

```bash
GITHUB_REPO_OWNER="google"
GITHUB_REPO_NAME="gson"
JAR_FILENAME="gson-2.10.1.jar"
RELEASE_TAG="gson-parent-2.10.1"
```

### 3. Add Buildpack to Your Heroku App

### 3. Add Buildpack to Your Heroku App

```bash
# Using the buildpack from GitHub
heroku buildpacks:add https://github.com/YOUR_USERNAME/classic-buildpack.git

# Or set it as the primary buildpack
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git

# If you also need Java runtime, add it as well
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
   - Downloads the specified JAR file from GitHub releases using authentication
   - Caches the JAR for faster subsequent builds
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

For **public repositories**, authentication is optional but recommended to avoid GitHub API rate limits (60 requests/hour without auth vs 5000 with auth).

## Configuration Options

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `GITHUB_REPO_OWNER` | Yes | GitHub repository owner/organization | `apache` |
| `GITHUB_REPO_NAME` | Yes | GitHub repository name | `commons-lang` |
| `JAR_FILENAME` | Yes | Exact filename of the JAR in the release | `commons-lang3-3.12.0.jar` |
| `RELEASE_TAG` | No | Release tag or "latest" (default: "latest") | `v3.12.0` or `latest` |
| `AUTO_START` | No | Auto-start JAR as background process (default: "false") | `true` or `false` |
| `JVM_OPTIONS` | No | JVM options when auto-starting (requires AUTO_START=true) | `-Xmx512m -Dport=8081` |

## Running the Downloaded JAR

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
JAR_FILENAME="http-server-app-1.0.0.jar"
AUTO_START="true"
JVM_OPTIONS="-Xmx512m -Dserver.port=8081"
```

The JAR will automatically start in the background when your dyno boots. Logs are written to `$HOME/logs/github-jar.log`.

### Option 3: Worker Dyno

Create a `Procfile` to run the JAR as a separate worker process:

```
web: <your-main-app-command>
worker: java -jar $HOME/vendor/jars/your-app.jar
```

Then scale:
```bash
heroku ps:scale worker=1
```

### Option 4: Executable in Procfile

Run the JAR as your main web process:

```
web: java -jar $HOME/vendor/jars/your-app.jar
```

## JAR Location at Runtime

The downloaded JAR will be available at:
- File path: `$HOME/vendor/jars/YOUR_JAR_FILENAME.jar`
- Automatically included in `$CLASSPATH`

## Caching

The buildpack caches downloaded JARs to speed up subsequent builds. The cache is invalidated when:
- The release tag changes
- A new version is released (when using "latest")

## Finding the Correct JAR Filename

To find the correct JAR filename:

1. Go to your GitHub repository releases page: `https://github.com/OWNER/REPO/releases`
2. Find the release you want to use
3. Look at the "Assets" section
4. Copy the exact filename of the JAR file you want to download

## Examples

### Example 1: Using Latest Release

```bash
GITHUB_REPO_OWNER="FasterXML"
GITHUB_REPO_NAME="jackson-core"
JAR_FILENAME="jackson-core-2.16.0.jar"
RELEASE_TAG="latest"
```

### Example 2: Using Specific Release Tag

```bash
GITHUB_REPO_OWNER="junit-team"
GITHUB_REPO_NAME="junit5"
JAR_FILENAME="junit-jupiter-api-5.10.1.jar"
RELEASE_TAG="r5.10.1"
```

## Troubleshooting

### Authentication Issues for Private Repositories

**Error:** `404 Not Found` when trying to access a private repository

**Solution:**
1. Make sure you've set the `GITHUB_TOKEN` config var:
   ```bash
   heroku config:set GITHUB_TOKEN=ghp_your_token_here
   ```
2. Verify your token has the `repo` scope
3. Check that the token hasn't expired

### GitHub API Rate Limiting

**Error:** `API rate limit exceeded`

**Solution:**
Set a `GITHUB_TOKEN` to increase your rate limit from 60 to 5000 requests per hour:
```bash
heroku config:set GITHUB_TOKEN=ghp_your_token_here
```

### JAR file not found in release

**Error:** `Could not find JAR file 'xxx.jar' in latest release`

**Solution:** 
- Double-check the exact filename in the GitHub release
- Verify the release actually contains the JAR file as an asset
- For private repos, ensure `GITHUB_TOKEN` is set
- Check the buildpack logs for available assets

## Development

### Testing Locally

You can test the buildpack locally using Docker:

```bash
# Create a test app
mkdir test-app
cd test-app

# Create .buildpack-config
cat > .buildpack-config <<EOF
GITHUB_REPO_OWNER="google"
GITHUB_REPO_NAME="gson"
JAR_FILENAME="gson-2.10.1.jar"
RELEASE_TAG="latest"
EOF

# Run buildpack
/path/to/classic-buildpack/bin/detect .
/path/to/classic-buildpack/bin/compile . /tmp/cache /tmp/env
```

## License

MIT

## Contributing

Pull requests are welcome! Please feel free to submit issues and enhancement requests.

