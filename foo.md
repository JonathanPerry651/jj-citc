#!/usr/bin/env bash
set -euo pipefail

ARCH="${1:-amd64}" # amd64 (x64) or arm64
OUT_DIR="${2:-./antigravity-dist}"

mkdir -p "${OUT_DIR}"
cd "${OUT_DIR}"

APT_BASE="https://us-central1-apt.pkg.dev/projects/antigravity-auto-updater-dev"
INDEX_URL="${APT_BASE}/dists/antigravity-debian/main/binary-${ARCH}/Packages"

echo "[1/4] Fetching latest release metadata from APT index..."
INDEX_DATA=$(curl -fsSL "${INDEX_URL}")

# Parse latest Version and Package Path
LATEST_VERSION_LINE=$(echo "${INDEX_DATA}" | awk -F': ' '/^Version:/ {print $2}' | tail -n1)
LATEST_FILE_PATH=$(echo "${INDEX_DATA}" | awk -F': ' '/^Filename:/ {print $2}' | tail -n1)

VERSION=$(echo "${LATEST_VERSION_LINE}" | cut -d'-' -f1)
BUILD_ID=$(echo "${LATEST_VERSION_LINE}" | cut -d'-' -f2)
FOLDER_ARCH=$([ "$ARCH" = "amd64" ] && echo "x64" || echo "arm64")

echo "  -> Version: ${VERSION}, Build ID: ${BUILD_ID}"
echo "  -> Package: ${LATEST_FILE_PATH}"

# Download from Google Artifact Registry
DEB_URL="${APT_BASE}/${LATEST_FILE_PATH}"
DEB_FILENAME=$(basename "${LATEST_FILE_PATH}")

echo "[2/4] Downloading package from Google Artifact Registry..."
curl -fL --progress-bar -o "${DEB_FILENAME}" "${DEB_URL}"

# Extract application payload
echo "[3/4] Extracting Desktop UI files..."
TMP_UNPACK="_unpack_tmp"
rm -rf "${TMP_UNPACK}"
mkdir -p "${TMP_UNPACK}"

(
    cd "${TMP_UNPACK}"
    ar -x "../${DEB_FILENAME}"
    if [ -f data.tar.xz ]; then
        tar -xf data.tar.xz
    elif [ -f data.tar.gz ]; then
        tar -xzf data.tar.gz
    elif [ -f data.tar.zst ]; then
        tar --zstd -xf data.tar.zst
    fi
)

TARGET_FOLDER="Antigravity-${FOLDER_ARCH}"
rm -rf "${TARGET_FOLDER}"
mv "${TMP_UNPACK}/usr/share/antigravity" "${TARGET_FOLDER}"
rm -rf "${TMP_UNPACK}"

# Create standard tar.gz distribution archive
echo "[4/4] Generating Antigravity.tar.gz..."
TARBALL_NAME="Antigravity-${VERSION}-${BUILD_ID}-linux-${FOLDER_ARCH}.tar.gz"
tar -czf "${TARBALL_NAME}" "${TARGET_FOLDER}"
ln -sf "${TARBALL_NAME}" "Antigravity.tar.gz"

echo "✓ Done! Created ${OUT_DIR}/${TARBALL_NAME} and ${OUT_DIR}/${TARGET_FOLDER}"
