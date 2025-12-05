# Snyk - Setup & Quick Scan
**Purpose:** Analyze vulnerabilities (SCA), SAST security, and OS recommendations using Snyk.

---

## Scan Procedure

### 1. Install Snyk CLI (one-time setup)
```bash
# Option A: Install via Homebrew (recommended for macOS)
brew install snyk/tap/snyk

# Option B: Install via npm
npm install -g snyk

# Verify installation
snyk --version
```

### 2. Clone repository and checkout branch
```bash
git clone git@github.com:keiken-digital-solution/keiken-dragon-ai.git
cd keiken-dragon-ai && git checkout develop
```

### 3. Authenticate Snyk (one-time setup)
```bash
snyk auth
```
> **Note:** This will open a browser window to authenticate with your Snyk account

### 4. Run comprehensive Snyk scan
> **Important:** Ensure you are in the `keiken-dragon-ai` directory before running these commands

```bash
# Scan dependencies for vulnerabilities (SCA)
snyk test --all-projects --json > ../Scanner/Snyk/scan_results/snyk-sca-results.json

# Scan source code for security issues (SAST) - requires Snyk Code enabled
snyk code test --json > ../Scanner/Snyk/scan_results/snyk-sast-results.json

# Scan Docker images for OS vulnerabilities and recommendations
# Example with Frontend image
snyk container test keiken-dragon-ai-frontend --json > ../Scanner/Snyk/scan_results/snyk-container-frontend-results.json
```
> **Note:** Results are saved in `../Scanner/Snyk/scan_results/` directory

### 5. Generate HTML report
```bash
snyk test --file=backend/requirements.txt > ../Scanner/Snyk/scan_results/snyk-report-backend.txt
snyk test --file=frontend/package.json > ../Scanner/Snyk/scan_results/snyk-report-frontend.txt
```