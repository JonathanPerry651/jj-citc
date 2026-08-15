#!/usr/bin/env bash
set -euo pipefail

DEST_DIR="${1:-./antigravity-ui}"
ARCH="${2:-x64}" # x64 or arm64

mkdir -p "${DEST_DIR}"
cd "${DEST_DIR}"

RELEASES_API="https://antigravity-auto-updater-974169037036.us-central1.run.app/releases"

echo "==> [1/4] Fetching latest Antigravity Desktop UI release metadata..."
RELEASES_JSON=$(curl -fsSL "${RELEASES_API}")

# Automatically extract the latest version & build ID
LATEST_VERSION=$(echo "${RELEASES_JSON}" | python3 -c "import sys, json; data=json.load(sys.stdin); print(data[0]['version'])")
LATEST_BUILD_ID=$(echo "${RELEASES_JSON}" | python3 -c "import sys, json; data=json.load(sys.stdin); print(data[0]['execution_id'])")

echo "    Found Latest UI Release: ${LATEST_VERSION} (Build ID: ${LATEST_BUILD_ID})"

# Fetch release manifest
MANIFEST_URL="https://storage.googleapis.com/antigravity-public/antigravity-hub/${LATEST_VERSION}-${LATEST_BUILD_ID}/linux-${ARCH}/manifest.json"
echo "==> [2/4] Verifying release manifest (${MANIFEST_URL})..."
curl -fsSL "${MANIFEST_URL}" | python3 -m json.tool

TARBALL_URL="https://storage.googleapis.com/antigravity-public/antigravity-hub/${LATEST_VERSION}-${LATEST_BUILD_ID}/linux-${ARCH}/Antigravity.tar.gz"
TARBALL_FILE="Antigravity-UI-${LATEST_VERSION}-${LATEST_BUILD_ID}-linux-${ARCH}.tar.gz"

echo "==> [3/4] Downloading UI tarball: ${TARBALL_URL}..."
curl -fL --progress-bar -o "${TARBALL_FILE}" "${TARBALL_URL}"

echo "==> [4/4] Extracting tarball..."
mkdir -p "Antigravity-UI-${LATEST_VERSION}"
tar -xzf "${TARBALL_FILE}" -C "Antigravity-UI-${LATEST_VERSION}"

echo "✓ Successfully downloaded and extracted Antigravity UI!"
