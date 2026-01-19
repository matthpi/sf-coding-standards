# Project Setup

Setting up a new Salesforce project following Enterprise Patterns.

## Prerequisites

Install these tools before starting:

- **Git** - Version control
- **Node.js** - For npm/Yarn package management
- **Salesforce CLI** - `npm install -g @salesforce/cli`
- **VS Code** - Recommended IDE with Salesforce extensions

## 1. Initialize Project

```bash
# Create project directory
mkdir my-salesforce-project
cd my-salesforce-project

# Initialize Salesforce project
sf project generate --name my-salesforce-project

# Initialize Git
git init
```

## 2. Add Enterprise Libraries

Add **fflib-apex-common** and dependencies as Git submodules:

```bash
# Add sf-apex-common (includes fflib submodules)
git submodule add https://github.com/matthpi/sf-apex-common.git libs/sf-apex-common

# Initialize and fetch all submodules (including fflib libs)
git submodule update --init --recursive
```

Your `libs/` structure will be:
```
libs/
└── sf-apex-common/
    ├── force-app/              # Your custom extensions
    └── libs/                   # fflib submodules
        ├── fflib-apex-common/
        └── fflib-apex-mocks/
```

## 3. Create Core Files

**README.md** - Project overview and setup instructions

**LICENSE** - Choose appropriate license (MIT recommended for internal tools)

**copilot-instructions.md** - Create using the template in the [main README](../README.md#using-these-standards-in-your-project) that references these standards and adds your project-specific context

## 4. Setup Code Quality (Husky)

Install Husky for pre-commit validation:

```bash
# Initialize package.json
npm init -y

# Install Husky
npm install --save-dev husky
npx husky init

# Install Code Analyzer
sf plugins install @salesforce/sfdx-scanner
```

### Add package.json scripts:

```json
{
  "scripts": {
    "validate:apex": "sf code-analyzer run --target \"force-app\" --severity-threshold 3 --format table"
  },
  "devDependencies": {
    "husky": "^9.0.0"
  }
}
```

### Create pre-commit hook:

```bash
npx husky add .husky/pre-commit "npm run validate:apex"
```

## 5. Team Onboarding

New team members clone and setup:

```bash
git clone <repo-url>
cd <project>

# Install submodules
git submodule update --init --recursive

# Install npm dependencies (includes Husky)
npm install

# Authorize org
sf org login web --alias DevHub
```

## Troubleshooting

**Submodules not loading:**
```bash
git submodule update --init --recursive
```

**Code Analyzer not found:**
```bash
sf plugins install @salesforce/sfdx-scanner
```

**Husky hooks not running:**
```bash
npm install
npx husky install
```
