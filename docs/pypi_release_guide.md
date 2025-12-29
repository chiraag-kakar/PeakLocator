# PyPI Release Guide

Complete guide for publishing Python packages to TestPyPI and production PyPI.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Build Package](#build-package)
- [TestPyPI Workflow](#testpypi-workflow)
- [Production PyPI Workflow](#production-pypi-workflow)
- [Verification](#verification)

---

## Prerequisites

### Required Tools
```bash
# Install build tools
python3 -m pip install build twine

# Verify installation
python3 -m build --version
python3 -m twine --version
```

### Create Accounts
1. **TestPyPI**: https://test.pypi.org/account/register/
2. **Production PyPI**: https://pypi.org/account/register/

---

## Setup

### Step 1: Generate API Tokens

**For TestPyPI:**
1. Go to: https://test.pypi.org/manage/account/tokens/
2. Create token named `your-package-testpypi` or similar
3. Copy the token (you won't be able to see it again)

**For Production PyPI:**
1. Go to: https://pypi.org/manage/account/tokens/
2. Create token named `your-package-pypi` or similar
3. Copy the token

### Step 2: Configure ~/.pypirc

Create or update `~/.pypirc` with both repositories:

```bash
cat >> ~/.pypirc << 'EOF'
[distutils]
  index-servers =
    testpypi
    pypi

[testpypi]
  repository = https://test.pypi.org/legacy/
  username = __token__
  password = pypi-YOUR_TESTPYPI_TOKEN_HERE

[pypi]
  repository = https://upload.pypi.org/legacy/
  username = __token__
  password = pypi-YOUR_PRODUCTION_PYPI_TOKEN_HERE
EOF
```

**Important:** Replace token placeholders with your actual tokens from the previous step.

### Step 3: Set Proper Permissions

```bash
chmod 600 ~/.pypirc
```

---

## Build Package

### Clean Previous Builds
```bash
rm -rf build/ dist/ *.egg-info
```

### Build Distribution Files
```bash
python3 -m build
```

This creates two distribution files:
- `dist/your-package-X.Y.Z.tar.gz` (source distribution)
- `dist/your-package-X.Y.Z-py3-none-any.whl` (wheel distribution)

### Validate Package
```bash
python3 -m twine check dist/*
```

Expected output:
```
Checking dist/your-package-X.Y.Z-py3-none-any.whl: PASSED
Checking dist/your-package-X.Y.Z.tar.gz: PASSED
```

---

## TestPyPI Workflow

### Step 1: Upload to TestPyPI
```bash
python3 -m twine upload --repository testpypi dist/*
```

Expected output:
```
Uploading distributions to https://test.pypi.org/legacy/
Uploading your-package-X.Y.Z-py3-none-any.whl
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ XX.X/XX.X kB
Uploading your-package-X.Y.Z.tar.gz
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ XX.X/XX.X kB

View at:
https://test.pypi.org/project/your-package/X.Y.Z/
```

### Step 2: Test Installation

**In a clean virtual environment:**
```bash
python3 -m venv test_env
source test_env/bin/activate

# Install from TestPyPI
pip install --index-url https://test.pypi.org/simple/ your-package

# Test import
python3 -c "import your_package; print('Success!')"

# Cleanup
deactivate
rm -rf test_env
```

### Step 3: Verify on TestPyPI
Visit: `https://test.pypi.org/project/your-package/`

Check:
- ✅ Correct version displayed
- ✅ README renders properly
- ✅ Package files available
- ✅ Installation works
- ✅ Dependencies are correct

---

## Production PyPI Workflow

### Prerequisites
- ✅ Tested on TestPyPI
- ✅ All tests passing locally
- ✅ Version bumped in `pyproject.toml`
- ✅ CHANGELOG.md or release notes updated
- ✅ Git tag created (optional but recommended)

### Step 1: Create Git Tag (Recommended)
```bash
git tag -a vX.Y.Z -m "Release version X.Y.Z"
git push origin vX.Y.Z
```

### Step 2: Upload to Production PyPI
```bash
python3 -m twine upload --repository pypi dist/*
```

Expected output:
```
Uploading distributions to https://upload.pypi.org/legacy/
Uploading your-package-X.Y.Z-py3-none-any.whl
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ XX.X/XX.X kB
Uploading your-package-X.Y.Z.tar.gz
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ XX.X/XX.X kB

View at:
https://pypi.org/project/your-package/X.Y.Z/
```

### Step 3: Test Production Installation

```bash
# Install from PyPI (official)
pip install your-package

# Verify version
pip show your-package

# Test import
python3 -c "import your_package; print('Success!')"
```

### Step 4: Verify on PyPI
Visit: `https://pypi.org/project/your-package/`

Check:
- ✅ Correct version displayed
- ✅ README renders properly
- ✅ Installation works globally
- ✅ Package discoverable on PyPI

---

## Using GitHub Actions for Automated Releases

### Workflow Setup

Create `.github/workflows/release.yml`:

```yaml
name: Release to PyPI

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., 1.0.0)'
        required: true
        type: string
      repository:
        description: 'Target repository'
        required: true
        type: choice
        options:
          - testpypi
          - pypi
        default: testpypi

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install build twine

      - name: Update version
        run: |
          # Update your version file/config
          # Example for pyproject.toml:
          sed -i "s/^version = \".*\"/version = \"${{ github.event.inputs.version }}\"/" pyproject.toml

      - name: Run tests
        run: |
          pip install -e ".[dev]"
          pytest

      - name: Build package
        run: python -m build

      - name: Validate package
        run: python -m twine check dist/*

      - name: Upload to TestPyPI
        if: github.event.inputs.repository == 'testpypi'
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.TESTPYPI_API_TOKEN }}
        run: python -m twine upload --repository testpypi dist/*

      - name: Upload to PyPI
        if: github.event.inputs.repository == 'pypi'
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
        run: python -m twine upload dist/*

      - name: Create Release
        if: github.event.inputs.repository == 'pypi'
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ github.event.inputs.version }}
          files: dist/*
```

### GitHub Secrets Setup

In your repository settings, add these secrets:
- `TESTPYPI_API_TOKEN` - Your TestPyPI token
- `PYPI_API_TOKEN` - Your production PyPI token

**Never commit tokens to version control.**

---

## Verification

### Check Package Contents
```bash
# List tar.gz contents
tar -tzf dist/your-package-X.Y.Z.tar.gz | head -20

# List wheel contents
unzip -l dist/your-package-X.Y.Z-py3-none-any.whl | head -20
```

### Verify Installation
```bash
python3 -c "
import your_package
print(f'Version: {your_package.__version__}')
print('Import successful!')
"
```

### Check PyPI Pages
- TestPyPI: `https://test.pypi.org/project/your-package/`
- PyPI: `https://pypi.org/project/your-package/`

---

## Troubleshooting

### Issue: `zsh: command not found: twine`
**Solution:** Use module syntax instead:
```bash
python3 -m twine upload --repository testpypi dist/*
```

### Issue: 403 Forbidden when uploading
**Solutions:**
- Verify token is correct in `~/.pypirc`
- Check token hasn't expired
- Ensure username is exactly `__token__` (not your PyPI username)
- Verify repository URL matches (testpypi vs pypi)

### Issue: Package version already exists
**Solution:** Increment version in your config and rebuild:
```bash
# Update version in pyproject.toml, setup.py, or __init__.py
# Then rebuild:
rm -rf build/ dist/ *.egg-info
python3 -m build
```

### Issue: Invalid package metadata
**Solution:** Validate before uploading:
```bash
python3 -m twine check dist/*
```

Fix any reported issues in your README, setup files, or dependencies, then rebuild.

### Issue: Dependencies not recognized
**Solution:** Ensure `pyproject.toml` or `setup.py` properly declares dependencies:
```toml
[project]
dependencies = [
    "requests>=2.28.0",
    "numpy>=1.20",
]
```

---

## Quick Reference Commands

### One-time Setup
```bash
# Install tools
python3 -m pip install build twine

# Configure credentials
cat >> ~/.pypirc << 'EOF'
[testpypi]
  repository = https://test.pypi.org/legacy/
  username = __token__
  password = pypi-YOUR_TOKEN_HERE

[pypi]
  repository = https://upload.pypi.org/legacy/
  username = __token__
  password = pypi-YOUR_TOKEN_HERE
EOF

chmod 600 ~/.pypirc
```

### Release Workflow
```bash
# 1. Build
rm -rf build/ dist/ *.egg-info
python3 -m build

# 2. Validate
python3 -m twine check dist/*

# 3. Test (TestPyPI)
python3 -m twine upload --repository testpypi dist/*

# 4. Verify on TestPyPI
# https://test.pypi.org/project/your-package/

# 5. Release (Production PyPI)
python3 -m twine upload --repository pypi dist/*
```

---

## Best Practices

- ✅ **Always test on TestPyPI first** before releasing to production
- ✅ **Version your releases** using semantic versioning (X.Y.Z)
- ✅ **Update CHANGELOG** for every release
- ✅ **Create Git tags** for release versions
- ✅ **Run full test suite** before building
- ✅ **Validate package metadata** with `twine check`
- ✅ **Use GitHub Actions** for automated, consistent releases
- ✅ **Keep tokens secure** - never commit them, use GitHub Secrets
- ❌ **Don't reuse version numbers** - increment for every release
- ❌ **Don't use plain text tokens** - always use secrets management

---

## References

- [PyPI Help](https://pypi.org/help/)
- [Twine Documentation](https://twine.readthedocs.io/)
- [Python Packaging Guide](https://packaging.python.org/)
- [PEP 517 - Build Backend](https://www.python.org/dev/peps/pep-0517/)
- [Semantic Versioning](https://semver.org/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
