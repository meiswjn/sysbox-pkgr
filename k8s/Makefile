#
# Sysbox Deploy K8s Daemonset Makefile
#

.PHONY: sysbox-deploy-k8s-image \
	fetch_sysbox_ce_bins \
	fetch_sysbox_ee_bins \
	check-sysbox-artifacts \
	check-crio-artifacts \
	clean-sysbox-ce clean-crio clean


SHELL := /bin/bash

SYSBOX_BINS = sysbox-runc sysbox-mgr sysbox-fs

# List of sysbox binaries to overwrite from local source builds after extracting
# the released sysbox-ce .deb. Default preserves existing behavior.
OVERWRITE_SYSBOX_BINS_FROM_SOURCE ?= sysbox-runc

# Obtain the version to build from the Sysbox repo itself. If a full version string
# is not found in the corresponding VERSION file, append a dummy revision/patch ("0")
# to match the name of the sysbox's deb image to download.
# Examples:
# 1) if VERSION = 0.6.3, then
#     SYSBOX_CE_VER        = 0.6.3
#     SYSBOX_CE_VER_SEMVER = 0.6.3
#     SYSBOX_CE_VER_FULL   = 0.6.3-0
#
# 2) if VERSION = 0.6.3-1, then
#     SYSBOX_CE_VER        = 0.6.3-1
#     SYSBOX_CE_VER_SEMVER = 0.6.3
#     SYSBOX_CE_VER_FULL   = 0.6.3-1
#
SYSBOX_CE_VER = $(shell cat ../sources/sysbox/VERSION)
SYSBOX_CE_VER_SEMVER = $(shell echo $(SYSBOX_CE_VER) | cut -d"-" -f1)
SYSBOX_CE_VER_FULL = $(shell echo $(SYSBOX_CE_VER) | sed '/-[0-9]/!s/.*/&-0/')

# Optional fallback sysbox-ce version if the requested sysbox-ce .deb is not
# available upstream (e.g., when building from an unreleased Sysbox version).
#
# When building sysbox-pkgr as part of the full sysbox repo, we try to infer a
# reasonable fallback from the sysbox-install manifest image tag.
SYSBOX_CE_VER_FALLBACK ?= $(shell \
	if [ -f ../../sysbox-k8s-manifests/sysbox-install.yaml ]; then \
		grep -E 'image:.*sysbox-deploy-k8s:v' ../../sysbox-k8s-manifests/sysbox-install.yaml | head -1 | sed -E 's/.*:v([^[:space:]]+).*/\1/'; \
	fi)

# Last-resort fallback if no manifest is present.
SYSBOX_CE_VER_FALLBACK_DEFAULT ?= 0.6.7-0

# CRIO versions to build.
CRIO_VERSIONS = v1.29 v1.30 v1.31 v1.32 v1.33

# Patch version is used to track changes to the sysbox-deploy-k8s image not related to
# the Sysbox's version. For example, if we need to rebuild the sysbox-deploy-k8s image
# due to a change in any of the files in this directory, we bump this patch version.
# If there's no need for a 'patch' version, then the patch version is set to nil (i.e.,
# empty). This is useful to distinguish between sysbox-deploy-k8s images that have the
# same Sysbox version but differ in other components (e.g., crio versions, supported
# k8s releases, etc.).
SYSBOX_DEPLOY_K8S_IMAGE_PATCH := ""

# Obtain the current system architecture.
UNAME_M := $(shell uname -m)
ifeq ($(UNAME_M),x86_64)
	SYS_ARCH := amd64
else ifeq ($(UNAME_M),aarch64)
	SYS_ARCH := arm64
else ifeq ($(UNAME_M),arm64)
	SYS_ARCH := arm64
else ifeq ($(UNAME_M),arm)
	SYS_ARCH := armhf
else ifeq ($(UNAME_M),armel)
	SYS_ARCH := armel
endif

#
# Sysbox artifacts for Ubuntu distro
#

# The fetch_* targets download the Sysbox binaries from the Sysbox repo and
# place them in the "bin" sub-directory. The *_image targets then load those
# binaries into the sysbox-deploy-k8s image. Variable SYSBOX_CE_VER selects
# the version of the Sysbox binaries to download.

fetch-sysbox-ce-bins:
	@echo "Fetching Sysbox CE binaries ..."
	@tmpdir="$$(mktemp -d)"; \
	set -e; \
	echo "TMPDIR = $$tmpdir"; \
	primary_semver="$(SYSBOX_CE_VER_SEMVER)"; \
	primary_full="$(SYSBOX_CE_VER_FULL)"; \
	deb="sysbox-ce_$${primary_full}.linux_$(SYS_ARCH).deb"; \
	url="https://storage.googleapis.com/sysbox-releases/v$${primary_semver}/sysbox-ce/$${deb}"; \
	echo "Attempting download: $$url"; \
	if ! wget -q "$$url" -P "$$tmpdir"; then \
		fallback_ver="$(SYSBOX_CE_VER_FALLBACK)"; \
		if [ -z "$$fallback_ver" ]; then fallback_ver="$(SYSBOX_CE_VER_FALLBACK_DEFAULT)"; fi; \
		fallback_semver="$$(echo "$$fallback_ver" | cut -d"-" -f1)"; \
		fallback_full="$$(echo "$$fallback_ver" | sed '/-[0-9]/!s/.*/&-0/')"; \
		deb="sysbox-ce_$${fallback_full}.linux_$(SYS_ARCH).deb"; \
		url="https://storage.googleapis.com/sysbox-releases/v$${fallback_semver}/sysbox-ce/$${deb}"; \
		echo "Primary sysbox-ce .deb not found; falling back to $$fallback_ver"; \
		echo "Attempting download: $$url"; \
		wget -q "$$url" -P "$$tmpdir"; \
		primary_full="$$fallback_full"; \
	fi; \
	mkdir -p "$$tmpdir/sysbox-ce-generic"; \
	dpkg -x "$$tmpdir/sysbox-ce_$${primary_full}.linux_$(SYS_ARCH).deb" "$$tmpdir/sysbox-ce-generic"; \
	mkdir -p bin/sysbox-ce/generic; \
	rm -rf bin/sysbox-ce/generic/*; \
	cp "$$tmpdir/sysbox-ce-generic/usr/bin/sysbox-"* bin/sysbox-ce/generic/.; \
	rm -rf "$$tmpdir"
	@echo "Before Overwrite"
	sha1sum bin/sysbox-ce/generic/*
	ls -alh bin/sysbox-ce/generic/
	@for bin in $(OVERWRITE_SYSBOX_BINS_FROM_SOURCE); do \
		case $$bin in \
			sysbox-runc) src="../../sysbox-runc/build/$(SYS_ARCH)/sysbox-runc" ;; \
			sysbox-mgr)  src="../../sysbox-mgr/build/$(SYS_ARCH)/sysbox-mgr" ;; \
			sysbox-fs)   src="../../sysbox-fs/build/$(SYS_ARCH)/sysbox-fs" ;; \
			*) echo "ERROR: unknown sysbox binary '$$bin' in OVERWRITE_SYSBOX_BINS_FROM_SOURCE"; exit 1 ;; \
		esac; \
		if [ ! -f "$$src" ]; then \
			echo "ERROR: missing local source-built binary: $$src"; \
			echo "Hint: build it first (e.g., from repo root: make sysbox)"; \
			exit 1; \
		fi; \
		echo "Overwriting $$bin with $$src"; \
		cp "$$src" "bin/sysbox-ce/generic/$$bin"; \
	done
	@echo "After Overwrite"
	sha1sum bin/sysbox-ce/generic/*
	ls -alh bin/sysbox-ce/generic/

#
# CRI-O artifacts (only built if not already present at ./bin/crio)
#
CRIO_BINS_DIR := $(PWD)/bin/crio

ifeq ($(wildcard $(CRIO_BINS_DIR)),)
build-crio: crio-build-container
	@echo "NOTE: building CRI-O binaries at ${PWD}/bin/crio"
	docker run --rm -v $(shell pwd)/bin:/mnt/results crio-bld
else
build-crio:
	@echo "NOTE: Skipping CRI-O build (found binaries at ${PWD}/bin/crio)"
endif

crio-build-container:
	docker build -t crio-bld -f Dockerfile.crio --build-arg sys_arch=$(SYS_ARCH) \
		--build-arg CRIO_VERSIONS="${CRIO_VERSIONS}" .

#
# The check-* targets verify that CRI-O, Sysbox binaries and its dependencies are
# all in the "bin" directory:
#
# bin
# ├── crio
# │   └── v1.24
# │       └── crio
# │   └── v1.25
# │       └── crio
# │   ├── v1.26
# │   │   └── crio
# │   ├── v1.27
# │   │   └── crio
# ├── sysbox-ce
# │   ├── ubuntu-bionic
# │   │   ├── sysbox-fs
# │   │   ├── sysbox-mgr
# │   │   └── sysbox-runc
# │   └── ubuntu-focal
# │       ├── sysbox-fs
# │       ├── sysbox-mgr
# │       └── sysbox-runc
#

check-sysbox-artifacts:
	$(foreach file,$(SYSBOX_BINS),[ -f "bin/sysbox-ce/generic/$(file)" ] || "missing sysbox-ce binary: bin/sysbox-ce/generic/$(file)")

check-crio-artifacts:
	@$(foreach version,$(CRIO_VERSIONS),[ -f "bin/crio/$(version)/crio" ] || "missing CRI-O binary: bin/crio/$(version)/crio";)
	@$(foreach version,$(CRIO_VERSIONS),[ -f "bin/crio/$(version)/pinns" ] || "missing CRI-O binary: bin/crio/$(version)/pinns";)

#
# These targets build the sysbox-deploy-k8s images for sysbox-ce
#

all: sysbox-deploy-k8s-image

sysbox-deploy-k8s-image: build-crio check-crio-artifacts fetch-sysbox-ce-bins check-sysbox-artifacts
	docker build -t ghcr.io/nestybox/sysbox-deploy-k8s:v$(SYSBOX_CE_VER_FULL)$(SYSBOX_DEPLOY_K8S_IMAGE_PATCH) \
		--build-arg sys_arch=$(SYS_ARCH) \
		--build-arg sysbox_version=v$(SYSBOX_CE_VER_FULL) \
		-f Dockerfile.sysbox-ce .

#
# Cleanup targets
#

clean-sysbox-ce:
	-rm -rf bin/sysbox-ce

clean-crio:
	-rm -rf bin/crio

clean: clean-sysbox-ce clean-crio