# Anchor CI Appliance

Docker image to generate deterministic, verifiable builds of Anchor programs.

This must be run *after* a given ANCHOR_CLI version is published and a git tag
is released on GitHub.


## Build Commands

```
docker build -t anchor-ci-appliance .
```

To set specific versions for solana-cli and anchor-cli (e.g. solana-cli v1.9.0, anchor-cli v0.19.0), run:

```
docker build -t anchor-ci-appliance \
    --build-arg SOLANA_CLI=v1.9.0 \
    --build-arg ANCHOR_CLI=v0.19.0 .
```


## Build for Gitlab Container Registry

Build with:

```
docker build -t registry.gitlab.com/socean-finance/util/anchor-ci-appliance .
```

tag with (solana-cli version followed by anchor-cli version):

```
docker tag \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v1.19.0
```

and push with:

```
docker push registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v1.19.0
```


## Local Usage with Docker

Build an anchor repo with:

```
docker run --rm -it \
    -v`pwd`:/workdir \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0 anchor build
```

Test it with:

```
docker run --rm -it \
    -v`pwd`:/workdir \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0 \
    /bin/bash -c "yarn install && solana-keygen new --no-passphrase -o ./.test-keypair.json && anchor test"
```

_assuming the project's `Anchor.toml` specifies `./.test-keypair.json` as the wallet; i.e. in `Anchor.toml`:_

```
[provider]
cluster = "localnet"
wallet = "./.test-keypair.json"
```


## Example `.gitlab-ci.yml`

[file](_gitlab-ci.yml)

```yaml
stages:
  - build
  - test

build:
  image: registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0
  stage: build
  script:
    - rustc --version && cargo --version && solana --version && anchor --version
    - anchor build

test:
  image: registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0
  stage: test
  script:
    - rustc --version && cargo --version && solana --version && anchor --version
    - solana-keygen new --no-passphrase -o ./test-keypair.json
    - yarn install
    - anchor test
```
