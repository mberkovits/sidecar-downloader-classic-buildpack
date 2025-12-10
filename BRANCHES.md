# Buildpack Branches

This repository contains two versions of the buildpack, each in its own branch:

## 📦 Main Branch: Download JAR from GitHub Releases

**Branch:** `main`

### What It Does
- Downloads a pre-built JAR file from a GitHub repository's releases
- Uses GitHub API to fetch release information
- Caches downloaded JARs for faster subsequent builds
- Makes JAR available in runtime classpath

### Use When
- You have pre-built JAR files in GitHub releases
- You want fast deployments (no build time)
- You need consistent, tested artifacts
- You're distributing compiled libraries

### Configuration Example
```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-app"
JAR_FILENAME="my-app-1.0.0.jar"
RELEASE_TAG="v1.0.0"  # or "latest"
AUTO_START="true"
```

### Pros
- ⚡ Very fast (no compilation)
- 🎯 Uses tested release artifacts
- 💾 Small cache size
- 🔄 Easy rollback (just change release tag)

### Cons
- ❌ Requires creating GitHub releases
- ❌ Can't use unreleased code
- ❌ Manual process to update

### Heroku Usage
```bash
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git
```

---

## 🔨 Build-from-Source Branch: Clone and Build

**Branch:** `build-from-source`

### What It Does
- Clones a Java project from GitHub
- Detects build tool (Maven or Gradle)
- Installs necessary build tools (JDK, Maven/Gradle)
- Compiles the project
- Extracts the built JAR
- Makes JAR available in runtime classpath

### Use When
- You want to build from source code
- You need the latest code from a branch
- You don't create GitHub releases
- You want continuous deployment from a branch

### Configuration Example
```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-app"
BRANCH="main"          # or any branch
BUILD_TOOL="gradle"    # or "maven" or "auto"
JAR_NAME_PATTERN="*-all.jar"
AUTO_START="true"
```

### Pros
- ✅ Always uses latest code from branch
- ✅ No need to create releases
- ✅ True continuous deployment
- ✅ Can build from feature branches

### Cons
- ⏱️ Slower (must compile every time)
- 💾 Larger cache (JDK, build tools)
- 🔧 Requires working build setup
- 📦 Build failures block deployment

### Heroku Usage
```bash
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git#build-from-source
```

---

## 🔀 Switching Between Branches

### Switch to Download from Releases
```bash
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git
# or explicitly:
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git#main
```

Update your `.buildpack-config`:
```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-app"
JAR_FILENAME="my-app-1.0.0.jar"
RELEASE_TAG="latest"
```

### Switch to Build from Source
```bash
heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git#build-from-source
```

Update your `.buildpack-config`:
```bash
GITHUB_REPO_OWNER="myorg"
GITHUB_REPO_NAME="my-app"
BRANCH="main"
BUILD_TOOL="gradle"
```

---

## 📊 Quick Comparison

| Feature | Main (Releases) | Build-from-Source |
|---------|----------------|-------------------|
| **Speed** | Very Fast ⚡⚡⚡ | Moderate ⚡ |
| **Build Time** | None | 2-5 minutes |
| **Cache Size** | ~10MB | ~200MB |
| **Latest Code** | Only releases | Any branch ✅ |
| **Dependencies** | Minimal | JDK, Maven/Gradle |
| **GitHub Releases** | Required | Not needed |
| **Rollback** | Easy (change tag) | Git revert |
| **CI/CD** | Manual release step | Automatic |
| **Private Repos** | Supported ✅ | Supported ✅ |
| **Auth Required** | GITHUB_TOKEN | GITHUB_TOKEN |

---

## 🎯 Decision Guide

### Choose **Main Branch** (Download from Releases) if:
- ✅ You create GitHub releases regularly
- ✅ You want predictable, tested artifacts
- ✅ You need fast deployments
- ✅ You want easy rollback to previous versions
- ✅ Your JAR is already built in CI/CD

### Choose **Build-from-Source Branch** if:
- ✅ You don't create GitHub releases
- ✅ You want to deploy directly from a branch
- ✅ You need the absolute latest code
- ✅ You're okay with longer build times
- ✅ You want true continuous deployment
- ✅ You're building from feature branches

---

## 🔧 Configuration File Format

Both branches use `.buildpack-config` but with different options:

### Main Branch Config
```bash
GITHUB_REPO_OWNER="..."      # Required
GITHUB_REPO_NAME="..."       # Required
JAR_FILENAME="..."           # Required
RELEASE_TAG="latest"         # Optional
AUTO_START="false"           # Optional
JVM_OPTIONS=""               # Optional
```

### Build-from-Source Config
```bash
GITHUB_REPO_OWNER="..."      # Required
GITHUB_REPO_NAME="..."       # Required
BRANCH="main"                # Optional
BUILD_TOOL="auto"            # Optional
JAR_NAME_PATTERN=""          # Optional
AUTO_START="false"           # Optional
JVM_OPTIONS=""               # Optional
```

---

## 📝 Example Workflows

### Workflow 1: Production with Releases (Main Branch)
1. Develop and test locally
2. Create GitHub release with built JAR
3. Deploy to Heroku (pulls from release)
4. Fast, predictable deployment

### Workflow 2: Dev Environment (Build-from-Source Branch)
1. Push code to `develop` branch
2. Deploy to Heroku dev environment
3. Buildpack clones and builds automatically
4. Test latest changes immediately

### Workflow 3: Hybrid Approach
- **Production**: Use `main` branch (releases)
- **Staging**: Use `build-from-source` branch (develop branch)
- Best of both worlds!

---

## 🚀 Getting Started

1. **Choose your branch** based on needs above
2. **Create `.buildpack-config`** with appropriate settings
3. **Set GITHUB_TOKEN** for private repos:
   ```bash
   heroku config:set GITHUB_TOKEN=ghp_your_token_here
   ```
4. **Set buildpack**:
   ```bash
   # For releases:
   heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git
   
   # For build-from-source:
   heroku buildpacks:set https://github.com/YOUR_USERNAME/classic-buildpack.git#build-from-source
   ```
5. **Deploy**:
   ```bash
   git push heroku main
   ```

---

## 📞 Need Help?

Check the README.md in each branch for detailed documentation:
- `main` branch: Release-based approach
- `build-from-source` branch: Compilation approach

Both support:
- Private repositories (with GITHUB_TOKEN)
- Auto-start as background process
- Custom JVM options
- Caching for performance

