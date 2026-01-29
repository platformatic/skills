---
name: watt-status
description: Quick health check for Platformatic Watt configuration
allowed-tools: Read, Bash, Glob
---

# Watt Status Check

Perform a quick health check on the Platformatic Watt configuration for the current project.

## Checks to Perform

### 1. Node.js Version
```bash
node --version
```
**Expected**: v22.19.0 or higher
**Status**: OK if >= v22.19.0, otherwise UPGRADE NEEDED

### 2. watt.json Existence
```bash
ls watt.json 2>/dev/null && echo "exists" || echo "missing"
```
**Status**: Found or Missing

### 3. watt.json Validity
If watt.json exists:
```bash
node -e "try { JSON.parse(require('fs').readFileSync('watt.json')); console.log('valid'); } catch(e) { console.log('invalid: ' + e.message); }"
```
**Status**: Valid JSON or Invalid (with error)

### 4. wattpm Installation
```bash
npx wattpm --version 2>/dev/null || echo "not installed"
```
**Status**: Installed (with version) or Not installed

### 5. package.json Scripts
Read package.json and check for:
- `dev` script containing `wattpm dev`
- `build` script containing `wattpm build`
- `start` script containing `wattpm start`

**Status**: Configured (all present), Partial (some present), or Missing (none present)

### 6. Environment Configuration
```bash
ls .env 2>/dev/null && echo "exists" || echo "missing"
```
Check if `.env` contains:
- `PLT_SERVER_HOSTNAME`
- `PLT_SERVER_LOGGER_LEVEL`
- `PORT`

## Output Format

Present results clearly:

```
Watt Configuration Status
=========================

Node.js Version:  v22.19.0       [OK]
watt.json:        Found          [OK]
Configuration:    Valid JSON     [OK]
wattpm:           v3.2.0         [OK]
Scripts:          Configured     [OK]
Environment:      .env found     [OK]

Status: Ready to run

Commands:
  npm run dev    - Start development server
  npm run build  - Build for production
  npm run start  - Start production server
```

Or if issues are found:

```
Watt Configuration Status
=========================

Node.js Version:  v20.10.0       [UPGRADE NEEDED]
watt.json:        Missing        [MISSING]
Configuration:    N/A            [-]
wattpm:           Not installed  [MISSING]
Scripts:          Not configured [MISSING]
Environment:      .env missing   [OPTIONAL]

Issues Found:
1. Node.js version must be v22.19.0 or higher
2. watt.json configuration file not found
3. wattpm is not installed

Next Steps:
  1. Upgrade Node.js to v22.19.0+
  2. Run /watt to initialize Platformatic Watt
```

## Quick Fixes

If issues are simple, offer to fix them:

- **Missing wattpm**: Suggest `npm install wattpm`
- **Missing scripts**: Offer to add them to package.json
- **Missing .env**: Offer to create with defaults
- **Missing watt.json**: Suggest running `/watt init`
