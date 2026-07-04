---
# Machine-readable project descriptor — schema v1 (2026-05-05).
name: docker-vista-fork
kind: [docker, tool]
status: archived                           # retired 2026-07-04 — provisioning via vista-meta VEHU + m-test-engine
languages: [shell, mumps, dockerfile]

runtime:
  needs:
    - docker
    - linux                                # author tested on Linux Mint
  optional:
    - "iris (one of two M backend choices)"
    - "yottadb (the other M backend choice)"
  excludes: []

distribution:
  pypi: null
  github: rafael5/docker-vista
  upstream_origin: "WorldVistA/docker-vista"

location: ~/projects/docker-vista-fork

exposes:
  dockerfiles:
    - "Dockerfile (parameterised builder for VistA on IRIS or YDB/GT.M)"
  build_args:
    - "instance — VistA flavour (vehu, foia, …)"
    - "flags — passed to autoInstaller.sh"
  prebuilt_images:
    - "worldvista/foiavista (Docker Hub, IRIS-backed)"
    - "worldvista/osehravista (Docker Hub, YDB-backed)"
    - "worldvista/vehu (the VEHU image used by vista-meta lookups)"
  artifacts:
    - "INSTALL_GUIDE.md (Rafael's verified install procedure for FOIA-on-IRIS)"

consumes:
  formats: ["VistA-M zip archives", "*.KID for post-install"]
  services: ["docker daemon"]

companions:
  - project: irisctl
    relation: "irisctl wraps the IRIS-backed containers built from this Dockerfile (foia container at ~/data/foia-iris/)"
  - project: ydbctl
    relation: "ydbctl wraps the YDB-backed containers; `~/data/ydb-test/` is the test target"
  - project: vista-meta
    relation: "vista-meta has its own bespoke Dockerfile (not from this repo) but the VEHU image lookup matches"
  - project: fm-web
    relation: "fm-web targets either backend produced by this Dockerfile (IRIS- and YDB-portable RPC explorer)"
  - project: VistA-DataLoader-fork
    relation: "patches/data loaded into containers built from this Dockerfile"

incompatibilities:
  - "Pre-built Docker Hub images are no longer being updated (per upstream README)."
  - "FOIA on IRIS has a ~5-LU license cap (Community Edition limit); ydb path is unconstrained."
  - "Volume permissions: IRIS container expects uid 51773; YDB containers vary. See INSTALL_GUIDE.md."

docs:
  primary: README.md
  install_guide: INSTALL_GUIDE.md
---

# docker-vista-fork

Fork of `WorldVistA/docker-vista`. Builds VistA or RPMS instances on
either IRIS or YottaDB using a single parameterised Dockerfile.

This fork holds:

- The unmodified upstream Dockerfile + scripts (no Rafael-specific edits committed).
- `INSTALL_GUIDE.md` — Rafael's verified install procedure for FOIA VistA on IRIS Community Edition (live volume-backed at `~/data/foia-iris/`, verified 2026-05-03).
- Notes for the irisctl / ydbctl wrappers (see those projects).

## Live instances

- **FOIA on IRIS**: `~/data/foia-iris/` (volume-backed, zero-loss restore). Reach via `irisctl`.
- **YDB test**: `~/data/ydb-test/` (rXXX directory). Reach via `ydbctl`.

## Don't edit

Treat the Dockerfile as upstream-canonical. Rafael's notes go in
`INSTALL_GUIDE.md`; tooling that wraps the running container goes in the
sibling `irisctl` / `ydbctl` projects.
