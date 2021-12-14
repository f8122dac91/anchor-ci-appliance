# Anchor CI Appliance

Docker image to generate deterministic, verifiable builds of Anchor programs.

This must be run *after* a given ANCHOR_CLI version is published and a git tag
is released on GitHub.

## Build Commands

```
docker build -t anchor-ci-appliance
```

To set specific versions for solana-cli and anchor-cli (e.g. solana-cli v1.9.0, anchor-cli v0.19.0), run:

```
docker build -t anchor-ci-appliance --build-arg SOLANA_CLI=v1.9.0 --build-arg ANCHOR_CLI=v0.19.0 .
```

## Build for Gitlab Container Registry

Build with:

```
docker build -t registry.gitlab.com/socean-finance/util/anchor-ci-appliance .
```

and push with:

```
docker push registry.gitlab.com/socean-finance/util/anchor-ci-appliance
```
