# GitHub JAR Downloader Buildpack

A Heroku buildpack that downloads a Java JAR file from a GitHub repository's releases and makes it available in your application's runtime classpath.

## Features

- 📦 Downloads JAR files from GitHub releases
- 🎯 Configurable via simple config file
- 🚀 Supports latest release or specific release tags
- 💾 Caches downloaded JARs for faster subsequent builds
- ♻️ Automatically adds JAR to runtime classpath

## Usage

### 1. Create Configuration File

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

### 2. Add Buildpack to Your Heroku App

```bash
# Using the buildpack from GitHub
heroku buildpacks:add https://github.com/YOUR_USERNAME/classic-buildpack.git

# Or set it as the primary buildpack
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git

# If you also need Java runtime, add it as well
heroku buildpacks:add heroku/java
```

### 3. Deploy

```bash
git push heroku main
```

## How It Works

1. **Detection Phase**: The buildpack checks for the presence of `.buildpack-config` in your app root
2. **Compile Phase**: 
   - Reads configuration from `.buildpack-config`
   - Downloads the specified JAR file from GitHub releases
   - Caches the JAR for faster subsequent builds
   - Places the JAR in `vendor/jars/` directory
3. **Runtime Phase**: The JAR is automatically added to the `CLASSPATH` environment variable

## Configuration Options

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `GITHUB_REPO_OWNER` | Yes | GitHub repository owner/organization | `apache` |
| `GITHUB_REPO_NAME` | Yes | GitHub repository name | `commons-lang` |
| `JAR_FILENAME` | Yes | Exact filename of the JAR in the release | `commons-lang3-3.12.0.jar` |
| `RELEASE_TAG` | No | Release tag or "latest" (default: "latest") | `v3.12.0` or `latest` |

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

### JAR file not found in release

**Error:** `Could not find JAR file 'xxx.jar' in latest release`

**Solution:** 
- Double-check the exact filename in the GitHub release
- Verify the release actually contains the JAR file as an asset
- Check the buildpack logs for available assets

### Authentication Issues

If downloading from a private repository:

```bash
# Set GitHub token as Heroku config var
heroku config:set GITHUB_TOKEN=your_personal_access_token
```

Then modify the `bin/compile` script to use authentication (add `-H "Authorization: token $GITHUB_TOKEN"` to curl commands).

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

