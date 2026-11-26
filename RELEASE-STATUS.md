# Alpha distribution status

Version: 0.1.0-alpha.1. Application source commit: 8d9fe16c016385c2d0b9b3425be3dd00595d4767.

This repository contains installation files only. It does not grant access to the private application source. Evaluation is governed by LICENSE.

The backend and web images were built, validated and published by the guarded release workflow. This release supports linux/amd64 images only; native arm64 acceptance is not established.

The runtime packages are private. An invited account verified both pinned image pulls without access to the source repository. The pulls reused local layers. A local installation from the release images passed SDK evaluation and a backend interruption test. A cold install, offline operation, backup and restore, version upgrade, and independent human acceptance remain open. Use this candidate for non-production evaluation.

The original release archive SHA256 is 42a647ae6cbf267610122963399d10ac7660c8d99324cac571f7ce7be4fcfc4f. Use this repository's [updated installer guide](README.md) with that archive.

No SLA is offered. Do not post secrets or unredacted logs in public issues.
