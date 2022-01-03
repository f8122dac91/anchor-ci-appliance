# Anchor CI Appliance

> Also hosted on [soceanfi/anchor-ci-appliance](https://hub.docker.com/repository/docker/soceanfi/anchor-ci-appliance) on docker hub

Docker image to generate deterministic, verifiable builds of Anchor programs.

This must be run *after* a given ANCHOR_CLI version is published and a git tag
is released on GitHub.


## Build Commands

```
docker build -t anchor-ci-appliance .
```

To set specific versions for solana-cli and anchor-cli (e.g. solana-cli v1.9.0, anchor-cli v0.19.0), run:

```
docker build -t anchor-ci-appliance:v1.9.0-v0.19.0 \
    --build-arg SOLANA_CLI=v1.9.0 \
    --build-arg ANCHOR_CLI=v0.19.0 .
```


## Local Usage with Docker

Build an anchor repo with:

```
docker run --rm -it \
    -v`pwd`:/workdir \
    anchor-ci-appliance:v1.9.0-v0.19.0 anchor build
```

Test it with:

```
docker run --rm -it \
    -v`pwd`:/workdir \
    anchor-ci-appliance:v1.9.0-v0.19.0 \
    /bin/bash -c "yarn install && solana-keygen new --no-passphrase -o ./.test-keypair.json && anchor test"
```

_the command assumes the project's `Anchor.toml` specifies `./.test-keypair.json` as the wallet; i.e. in `Anchor.toml`:_

```
[provider]
cluster = "localnet"
wallet = "./.test-keypair.json"
```

**NOTE**: To run js tests, as opposed to ts tests, that uses `mocha` instead of `ts-mocha`, use this command to
link the mocha binary before issuing anchor test:

```
docker run --rm -it \
    -v`pwd`:/workdir \
    anchor-ci-appliance:v1.9.0-v0.19.0 \
    /bin/bash -c "yarn install && cd node_modules/mocha/ && yarn link && cd ../.. && solana-keygen new --no-passphrase -o ./.test-keypair.json && anchor test"
```


## GitHub Action

### Example

[file](github-action.yml)

```yaml
name: CI

on:
  workflow_dispatch:
    inputs:
      semver:
        description: "Semantic version (vX.X.X) for this release"
        required: true

jobs:
  build-and-release:
    runs-on: ubuntu-latest
    env:
      ANCHOR_CI_APPLIANCE_IMAGE: "soceanfi/anchor-ci-appliance:v1.9.0-v0.19.0"
    steps:
      - uses: actions/checkout@v2
      - run: mkdir -p image-cache
      - id: load-image-cache
        name: load-image-cache
        uses: actions/cache@v2
        with:
          path: image-cache
          key: ${{ runner.os }}-image-cache
      - if: steps.load-image-cache.outputs.cache-hit == 'true'
        run: |
          docker load -i image-cache/ci-image.tar
      - name: save-image
        run: |
          docker pull ${{ env.ANCHOR_CI_APPLIANCE_IMAGE }} | grep "Image is up to date" || docker save -o image-cache/ci-image.tar ${{ env.ANCHOR_CI_APPLIANCE_IMAGE }}
      - name: load-build-cache
        uses: actions/cache@v2
        with:
          path: |
            target
            node_modules
          key: ${{ runner.os }}-${{ github.event.inputs.semver }}-anchor-build
      - name: load-ts-node-modules-cache
        uses: actions/cache@v2
        with:
          path: ts/node_modules
          key: ${{ runner.os }}-${{ github.event.inputs.semver }}-ts-node-modules
      - name: build
        # dont use `using:`, seems like some of github's env vars messes it up
        run: docker run -v `pwd`:/workdir ${{ env.ANCHOR_CI_APPLIANCE_IMAGE }} anchor build
      - name: give-read-permissions
        # for cache-actions, idk why
        run: sudo chmod -R a+r target
      - name: upload-binary-artifact
        uses: actions/upload-artifact@v2
        with:
          name: binary-artifact
          path: target/deploy/*.so
      - name: yarn-install-and-build
        run: |
          cd ts
          yarn install
          yarn build
      - name: prepare-ts-release
        # delete everything but cache dirs and move ts dist to top level
        run: |
          ls | grep -v ts | grep -v .gitignore | grep -v target | grep -v image-cache | grep -v node_modules | xargs rm -rf
          mv ts/dist .
          mv ts/.npmignore .
          mv ts/README.md .
          mv ts/package.json .
          mv ts/tsconfig.json .
          cd ts
          ls | grep -v node_modules | xargs rm -rf
      - name: ts-release
        run: |
          git config --global user.name "github-action-ci"
          git config --global user.email "none@users.noreply.github.com"
          git add -A
          PRE_COMMIT_ALLOW_NO_CONFIG=1 git commit -m "typescript release ${{ github.event.inputs.semver }}"
          git tag ${{ github.event.inputs.semver }}
          git push origin ${{ github.event.inputs.semver }}
```


## Gitlab CI

### Login to Gitlab Registry

```
docker login registry.gitlab.com
```


### Build for Gitlab Container Registry

Build with:

```
docker build -t registry.gitlab.com/socean-finance/util/anchor-ci-appliance .
```

tag with (solana-cli version followed by anchor-cli version):

```
docker tag \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance \
    registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0
```

and push with:

```
docker push registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0
```


### Example `.gitlab-ci.yml`

[file](gitlab-ci.yml)

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
  artifacts:
    expire_in: 24 hrs
    paths:
      - target/
  cache:
    key:
      files:
        - Cargo.lock
    paths:
      - target/

test:
  image: registry.gitlab.com/socean-finance/util/anchor-ci-appliance:v1.9.0-v0.19.0
  stage: test
  script:
    - rustc --version && cargo --version && solana --version && anchor --version
    - solana-keygen new --no-passphrase -o ./.test-keypair.json
    - yarn install
    - anchor test
  cache:
    key:
      files:
        - yarn.lock
    paths:
      - target/
```
