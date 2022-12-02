# 🦑 Verifier Server for ink!

Server for ink! source code verification.

Features:

- Reproducible builds for ink! source code verification
  - Uploading of verifiable source code packages
  - Tracking the status of the verification build process
  - Metadata is generated by the build
- Signed upload of contract metadata
- Access to the verified artifacts

For a high-level explanation of the ink! Verifier Server and how it integrates with the [Explorer UI](https://github.com/web3labs/epirus-substrate/tree/main/explorer-ui) and [ink! Verifier Image](https://github.com/web3labs/ink-verifier-image), please check out the [ink! Verifier Explainer](./docs/INK_VERIFIER_EXPLAINER.md)

For instructions on how to generate a verifiable source code package, please check out the ink! Verifier Image [documentation](https://github.com/web3labs/ink-verifier-image/blob/main/README.md#package-generation).

For instructions on how to carry out a full end-to-end test, please check out our [tutorial](./docs/TUTORIAL.md).

**Table of Contents**

- [Configuration](#configuration)
- [Running Locally](#running-locally)
- [Testing](#testing)
- [Linting](#linting)
- [Running in Production](#running-in-production)
- [Reproducible Builds Verification](#reproducible-builds-verification)
  - [Actors](#actors)
  - [Directories](#directories)
  - [Process Overview](#process-overview)
- [Unverified Metadata Upload](#unverified-metadata-upload)
- [Web API](#web-api)
  - [OpenAPI Documentation](#openapi-documentation)
  - [Postman Collection](#postman-collection)
- [Technical Notes](#technical-notes)
  - [Publish Directory](#publish-directory)
  - [Network Names](#network-names)
- [Additional Developer Tools](#additional-developer-tools)

## Configuration

The configuration uses the environment variables described in the table below.

|Name|Description|Defaults|
|----|-----------|--------|
|SERVER_HOST|The server host address. e.g. 0.0.0.0|`127.0.0.1`|
|SERVER_PORT|The server port to listen to|`3001`|
|OAS_URL|The external URL of the server for the Opean Api docs|`http://${SERVER_HOST}:${SERVER_PORT}`|
|BASE_DIR|Base directory for verification pipeline stages|`:project_root_dir/.tmp`|
|CACHES_DIR|The base directory for caches|`:project_root_dir/.tmp/caches`|
|PUBLISH_DIR|Base directory for long-term access to successfully verified artefacts|`:project_root_dir/.tmp/publish`|
|MAX_CONTAINERS|Maximum number of running containers|`5`|
|VERIFIER_IMAGE|The ink-verifier container image to be used for verification|`ink-verifier:develop`|
|CONTAINER_ENGINE|The container engine executable|`docker`|
|CONTAINER_RUN_PARAMS|Additional parameters for the conainter engine|n/a|

The server has support for `.env` files.

## Running Locally

After installing the project dependencies, only for the first time, using

```bash
npm i
```

you can start a development server

```bash
npm run start:dev
```

## Testing

To run the unit tests, use the command

```bash
npm test
```

To generate a test coverage report, execute

```bash
npm run test:coverage
```

## Linting

To apply the code linter and automatically fix issues

```bash
npm run lint
```

## Running in Production

The ink! Verification Server is meant to be run as a standalone OS process since it spawns container processes for
the reproducible builds and we want to avoid the nuances of Docker in Docker or similar solutions.

We recommend the usage of [PM2](https://pm2.keymetrics.io/) process manager, an `ecosystem.config.js` is provided for this purpose.

Example:

```bash
pm2 start ecosystem.config.js --env production
```

See [PM2 documentation](https://pm2.keymetrics.io/docs/usage/quick-start/).


## Reproducible Builds Verification

This section describes the source code verification process based on ink! reproducible builds.

### Actors

* Requestor: The user uploading the source code for verification
* Server: The verification server implemented in this repository
* Verifier Image: The container image with the verification logic for ink! source codes

### Directories

* Staging: `$BASE_DIR/staging/:network/:code-hash`
* Processing: `$BASE_DIR/processing/:network/:code-hash`
* Errors: `$BASE_DIR/errors/:network/:code-hash`
* Publish: `$PUBLISH_DIR/:code-hash`

### Process Overview

1. A requestor uploads the source packge archive for a network and code hash
2. The server checks that:
  * The source code for the network and code hash is not already verified or being verified
  * There is enough host resources to start a new verification

> **Staging** steps below happen in the staging directory

3. The server downloads the pristine WASM byte code correspondening to the provided network and code hash
4. The server streams the archive if is a compressed archive

> **Processing** in the processing directory

5. The server moves the staging files to the processing directory
6. The server runs a container process for the verifier image to verify the package in processing. See [source code verification workflow](./docs/INK_VERIFIER_EXPLAINER.md#source-code-verification-workflow) for details
7. On the event of container exit the server moves the verified artificats to the publish directory if the verification was successful, otherwise keeps a log in the errors directory

## Unverified Metadata Upload

> For verified source code and metadata use the [Reproducible Builds Verification](#reproducible-builds-verification) mechanism.

The service supports uploading **signed contract metadata as an additional alternative to reproducible builds generated metadata**.
Please note that the signed metadata is not verified and the owner of the code hash is trusted.

This feature responds to (1) the support for `build_info` data is only available from `cargo-contract 2.0.0-alpha.4`,
(2) there is no official image or procedure regarding reproducible builds yet, (3) we want to expand the service utility in the meantime.

Although it is a far from ideal way to bind the metadata to a given code hash it prevents trivial exploitation by:
- Verifying that the signature is from the owner account of the code hash.
- Verifying that the signed message matches the sha256 of the uploaded `metadata.json` + the `code hash` of the uploaded contract bytecode.

## Web API

### OpenAPI Documentation

OpenAPI schemas are automatically generated from the server route schemas. If you are running the server locally, you can find the schema at http://127.0.0.1:3001/oas.json.

For the public instance that we are running, the OpenAPI schema can be found at https://ink-verifier.sirato.xyz/api/oas.json.

We also serve a Swagger interface at https://ink-verifier.sirato.xyz/api/api-docs/

### Postman Collection

You can import the API collection and environments found [here](./it/postman/) into Postman for easy testing of the API. 

## Technical Notes

### Publish Directory

The publish directory is not segmented by network name because the code hashes are content-addressable, i.e. the same for all networks.
For source code verification the network name is required to download the uploaded pristine bytecode.

### Network Names

We are using [@polkadot/apps-config](https://github.com/polkadot-js/apps/tree/master/packages/apps-config) to resolve the network endpoints by name. You can find the available endpoints in the [endpoints directory](https://github.com/polkadot-js/apps/tree/master/packages/apps-config/src/endpoints).

## Additional Developer Tools

To make the verification process easier for ink! smart contract developers, we provide a command line tool to help with building smart contracts with ink! Verifier Image so that the contract can later be verified. At the same time, the tool does the packaging and compression of the source code and resulting `.contract` file in the directory structure that ink! Verifier Image expects during verification.

The tool can be installed from the github repository

```
❯ cargo install --git https://github.com/web3labs/ink-verifier-image.git
```

or built from source

```
❯ git clone git@github.com:web3labs/ink-verifier-image.git
❯ cd cli/
❯ cargo install --path .
```

The CLI can then be used as follows

```
❯ cd </path/to/contract>
❯ build-verifiable-ink -t develop .
```

The source code for the  tool can be found at https://github.com/web3labs/ink-verifier-image/tree/main/cli
