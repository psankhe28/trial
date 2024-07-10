# Contributing to Hyperledger Caliper

Welcome to Hyperledger Caliper project, we are excited about the prospect of you contributing.

## How to contribute
We are using GitHub issues for bug reports and feature requests.

If you find any bug in the source code or have any trivial changes (such as typos fix, minor feature), you can raise an issue or delivery a fix via a pull request directly.

If you have any enhancement suggestions or want to help extend caliper with more DLTs or have any other major changes, please start by opening an issue first. That way, relevant parties (e.g. maintainers or main contributors of the relevant subsystem) can have a chance to look at it before you do any work.

All PRs must get at least one review, you can ask `hyperledger/caliper-committers` for review. Normally we will review your contribution in one week. If you haven’t heard from anyone in one week, feel free to @ or mail a maintainer to review it.

All PRs must be signed before be merged, be sure to use `git commit -s` to commit your changes.

If a PR is reviewed and changes are requested then please do not force push the changes, push the changes into a new commit, this makes it easier to see the changes between the previously reviewed code and the new changes.

We use Azure pipelines to test the build - please test on your local branch before raising a PR.

There is also [Discord](https://discord.com/channels/905194001349627914/941417677778473031) with a Caliper channel for communication, anybody is welcome to join.

## Installing the Caliper code base

!!! note

    *this section is intended only for developers who would like to modify the Caliper code-base and experiment with the changes locally before raising pull requests. You should perform the following steps every time you make a modification you want to test, to correctly propagate any changes.*

The workflow of modifying the Caliper code-base usually consists of the following steps:

1. [Bootstrapping the repository](https://hyperledger.github.io/caliper/v0.5.0/general/contributing/#bootstrapping-the-caliper-repository)
2. [Modifying and testing the code](https://hyperledger.github.io/caliper/v0.5.0/general/contributing/#testing-the-code)
3. [Publishing package changes locally](https://hyperledger.github.io/caliper/v0.5.0/general/contributing/#publishing-to-local-npm-repository)
4. [Building the Docker image](https://hyperledger.github.io/caliper/v0.5.0/general/contributing/#building-the-docker-image)

### Bootstrapping the Caliper repository

!!! note

    *If you are developing from the latest main branch, refer to the instructions in the [vNext contributing guide](https://hyperledger.github.io/caliper/vNext/general/contributing/). The following instructions apply only apply if you have checked out the v0.5.0 tag and are developing from there.*

To install the basic dependencies of the repository, and to resolve the cross-references between the different packages in the repository, you must execute the following commands from the root of the repository directory:

1. `npm i`: Installs development-time dependencies, such as [Lerna](https://github.com/lerna/lerna#readme) and the license checking package

2. `npm run repoclean`: Cleans up the `node_modules` directory of all packages in the repository. Not needed for a freshly cloned repository.

3. `npm run bootstrap`: Installs the dependencies of all packages in the repository and links any cross-dependencies between the packages. It will take some time to finish installation. If it is interrupted by `ctrl+c`, please recover the `package.json` file first and then run `npm run bootstrap` again.

Or as a one-liner:
```sh
user@ubuntu:~/caliper$ npm i && npm run repoclean -- --yes && npm run bootstrap
```
!!! note

    *do not run any of the above commands with `sudo`, as it will cause the bootstrap process to fail.*

### Testing the code

Caliper has both unit tests and integration tests.

Unit tests can be run using `npm test` either in the root of the caliper source tree (to run them all) or within the specific package (eg caliper-fabric) to run just the tests within that package.

To run the integration tests for a specific SUT, use the following script from the root directory of the repository, setting the `BENCHMARK` environment variable to the platform name:

```sh
user@ubuntu:~/caliper$ BENCHMARK=fabric ./.build/benchmark-integration-test-direct.sh
```

The following platform tests (i.e., valid `BENCHMARK` values) are available:
- besu
- ethereum
- fabric
- fisco-bcos

A PR must pass all unit and integration tests.

If you would like to run other examples, then you can directly access the CLI in the `packages/caliper-cli` directory, without publishing anything locally.

```sh
user@ubuntu:~/caliper$ node ./packages/caliper-cli/caliper.js launch manager \
    --caliper-workspace ~/caliper-benchmarks \
    --caliper-benchconfig benchmarks/scenario/simple/config.yaml \
    --caliper-networkconfig networks/fabric/test-network.yaml
```

### Publishing to local NPM repository

The NPM publishing and installing steps for the modified code-base can be tested through a local NPM proxy server, Verdaccio. The steps to perform are the following:

1. Start a local Verdaccio server to publish to
2. Publish the packages from the Caliper repository to the Verdaccio server
3. Install and bind the CLI from the Verdaccio server
4. Run the integration tests or any sample benchmark

The `packages/caliper-publish` directory contains an internal CLI for easily managing the following steps. So the commands of the following sections must be executed from the `packages/caliper-publish` directory:

```sh
user@ubuntu:~/caliper$ cd ./packages/caliper-publish
```

!!! note

    *use the `--help` flag for the following CLI commands and sub-commands to find out more details.*

#### Starting Verdaccio
To setup and start a local Verdaccio server, run the following npm command:

```sh
user@ubuntu:~/caliper/packages/caliper-tests-integration$ npm run start_verdaccio
...
[PM2] Spawning PM2 daemon with pm2_home=.pm2
[PM2] PM2 Successfully daemonized
[PM2] Starting /home/user/projects/caliper/packages/caliper-tests-integration/node_modules/.bin/verdaccio in fork_mode (1 instance)
[PM2] Done.
| App name  | id | mode | pid    | status | restart | uptime | cpu | mem       | user   | watching |
|-----------|----|------|--------|--------|---------|--------|-----|-----------|--------|----------|
| verdaccio | 0  | fork | 115203 | online | 0       | 0s     | 3%  | 25.8 MB   | user   | disabled |

Use `pm2 show <id|name>` to get more details about an app
```

The Verdaccio server is now listening on the following address: `http://localhost:4873`

#### Publishing the packages
Once Verdaccio is running, you can run the following command to publish every Caliper package locally:

```sh
user@ubuntu:~/caliper/packages/caliper-publish$ ./publish.js npm --registry "http://localhost:4873"
...
+ @hyperledger/caliper-core@0.5.0-unstable-20220206065953
[PUBLISH] Published package @hyperledger/caliper-core@0.5.0-unstable-20220206065953
...
+ @hyperledger/caliper-fabric@0.5.0-unstable-20220206065953
[PUBLISH] Published package @hyperledger/caliper-fabric@0.5.0-unstable-20220206065953
...
+ @hyperledger/caliper-cli@0.5.0-unstable-20220206065953
[PUBLISH] Published package @hyperledger/caliper-cli@0.5.0-unstable-20200206065953
```
Take note of the dynamic version number you see in the logs, you will need it to install you modified Caliper version from Verdaccio (the unstable tag is also present on NPM, so Verdaccio would probably pull that version instead of your local one).

Since the published packages include a second-precision timestamp in their versions, you can republish any changes immediately without restarting the Verdaccio server and without worrying about conflicting packages.

#### Running package-based tests
Once the packages are published to the local Verdaccio server, we can use the usual NPM install approach. The only difference is that now we specify the local Verdaccio registry as the install source instead of the default, public NPM registry:


```sh
user@ubuntu:~/caliper/packages/caliper-tests-integration$ npm install --registry=http://localhost:4873 --only=prod @hyperledger/caliper-cli
user@ubuntu:~/caliper/packages/caliper-tests-integration$ npx caliper bind --caliper-bind-sut fabric --caliper-bind-sdk 1.4.0
user@ubuntu:~/caliper/packages/caliper-tests-integration$ npx caliper benchmark run --caliper-workspace ../caliper-samples --caliper-benchconfig benchmark/simple/config.yaml --caliper-networkconfig network/fabric-v1.4/2org1peergoleveldb/fabric-node.yaml
```

!!! note

    *we used the local registry only for the Caliper packages. The binding happens through the public NPM registry. Additionally, we performed the commands through npx and the newly installed CLI binary (i.e., not directly calling the CLI code file).*

### Building the Docker image
Once the modified packages are published to the local Verdaccio server, you can rebuild the Docker image. The Dockerfile is located in the `packages/caliper-publish` directory.

To rebuild the Docker image, execute the following:

```sh
user@ubuntu:~/caliper/packages/caliper-publish$ ./publish.js docker
...
Successfully tagged hyperledger/caliper:manager-unstable-20220206065953
[BUILD] Built Docker image "hyperledger/caliper:manager-unstable-20220206065953"
```

Now you can proceed with the Docker-based benchmarking as described in the previous sections.

!!! note

    *once you are done with the locally published packages, you can clean them up the following way:*
    ```sh
    user@ubuntu:~/caliper/packages/caliper-publish$ ./publish.js verdaccio stop
    ```

## Caliper Structure
Caliper is modularised under `packages` into the following components:

**caliper-cli**
This is the Caliper CLI that enables the running of a benchmark

**caliper-core**
Contains all the Caliper core code.

**caliper-**
Each `caliper-<adapter>` is a separate package that contains a distinct adaptor implementation to interact with different blockchain technologies. Current adaptors include:

- caliper-ethereum
- caliper-fabric
- caliper-fisco-bcos

Each adapter extends the `ConnectorBase` from the core package, as well as exports a `ConnectorFactory` function.

**caliper-tests-integration**
This is the integration test suite used for caliper; it runs in the Azure pipelines build and can (should) be run locally when checking code changes. Please see the readme within the package for more details.


## Add an Adaptor for a New DLT
New adapters must be added within a new package, under `packages`, with the naming convention `caliper-<adapter_name>`. Each adapter must implement a new class extended from `ConnectorBase` as the adapter for the DLT, as well export a `ConnectorFactory` function. Please refer to the existing Connectors for examples and requirements for implementation.

## Inclusive language guidelines

Please adhere to the inclusive language guidelines that the project has adopted as you make documentation updates.

- Consider that users who will read the docs are from different backgrounds and cultures and that they have different preferences.
- Avoid potential offensive terms and, for instance, prefer “allow list and deny list” to “white list and black list”.
- We believe that we all have a role to play to improve our world, and even if writing inclusive documentation might not look like a huge improvement, it’s a first step in the right direction.
- We suggest to refer to [Microsoft bias free writing guidelines](https://docs.microsoft.com/en-us/style-guide/bias-free-communication) and [Google inclusive doc writing guide](https://developers.google.com/style/inclusive-documentation) as starting points.

## License
The Caliper codebase is released under the Apache 2.0 license. Any documentation developed by the Caliper Project is licensed under the Creative Commons Attribution 4.0 International License. You may obtain a copy of the license, titled CC-BY-4.0, at http://creativecommons.org/licenses/by/4.0/.