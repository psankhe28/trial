## Overview

This page introduces the Fabric adapter that utilizes the Common Connection Profile (CCP) feature of the Fabric SDK to provide compatibility and a unified programming model across different Fabric versions.

!!! note

    *The LTS versions of Hyperledger Fabric as well as the very latest 2.x release of Hyperledger Fabric are supported, all other versions are unsupported*

The adapter exposes many SDK features directly to the user callback modules, making it possible to implement complex scenarios.

!!! note

    *Some highlights of the provided features:*

    - supports multiple channels and chaincodes
    - supports multiple organizations
    - supports multiple identities
    - private data collection support
    - support for TLS and limited mutual TLS communication (identity certificates cannot have restrictions on them)
    - option to select the identity for submitting a TX/query
    - supporting multiple orderers

## Installing dependencies
You must bind Caliper to a specific Fabric SDK to target the corresponding (or compatible) SUT version. Refer to the [binding documentation](https://hyperledger.github.io/caliper/v0.5.0/overview/installing-caliper/#the-bind-command) for details. When you bind to an SUT, you are in fact selecting a specific Fabric SDK to use which could be used with different versions of Fabric SUTs.

!!! note

    - *None of the Fabric bindings support administration actions. It it not possible to create/join channels nor deploy a chaincode. Consequently running caliper only facilitate operations using the `--caliper-flow-only-test` flag*

### Binding with Fabric 1.4
It is confirmed that a 1.4 Fabric SDK is compatible with a Fabric 2.2 and later Fabric 2.x SUTs, therefore this binding can be used with later Fabric SUTs

Note that when using the binding target for the Fabric SDK 1.4 there are capability restrictions:

!!! note

    - *Currently setting `discover` to `true` in the network configuration file is not supported if you don’t enable the `gateway` option (eg specifying –caliper-Fabric-gateway-enabled as a command line option)*
    - *Detailed execution data for every transaction is only available if you don’t enable the `gateway` option*

### Binding with Fabric 2.2
It is confirmed that a 2.2 Fabric SDK is compatible with 2.2 and later Fabric SUTs, therefore this binding can be used with 2.2 and later Fabric SUTs

!!! note
    *The following further restrictions exist for this binding*:<br>
      - *Detailed execution data for every transaction is not available.*

### Binding with Fabric 2.4

Only Fabric 2.4 and later with the Peer Gateway capability enabled (which is the default setting for a Fabric peer) can be used.

!!! note
    *The following further restrictions exist for this binding*<br>
    - *Detailed execution data for every transaction is not available.<br>*
    - *mutual TLS is not supported<br>*
    - *peer and organization targeting is not supported so the options `targetPeers` and `targetOrganizations` in a request will throw an error.*

## Connection Profiles
Connection Profiles are a Fabric standard that provides connectivity information for your Fabric network. In the past (Hyperledger Fabric 1.1) you needed to describe all your endpoints in a connection profile, ie all the orderers and all the peers in order to be able to connect a client application to the network. This is referred to as a `static` connection profile and when you use this connection profile with Caliper you should set the `discover` property to false. The problem with static connection profiles is that if a network topology changes (eg add/remove orderer, peer, organisation etc) then every client needs to have an updated connection profile.

Hyperledger Fabric in 1.2 introduced the concept of discovery. This allowed you to ask a peer for the network topology. Your Fabric network has to be configured correctly for this to work (but all Fabric networks should be configured to allow for discovery now). Connection profiles that use this capability will only have a list of 1 or more peers for the specific organisation that connection profile applies to which will be used to discover the network. These connection profiles are referred to as `dynamic` connection profiles and when you use this connection profile with Caliper you should set the discover property to true.

Network builders and providers should generate connection profiles (for example test-network in fabric-samples does this), however if you don’t have a connection profile you will need to create one. Information about creating connection profiles can be found in Hyperledger Fabric documentation as well as the node-sdk documentation (the format changed between node sdks. The 1.4 version should work when binding to either Fabric 1.4 or 2.2 but the version documented by 2.2 may only work when binding to Fabric 2.2)

- [node sdk 2.2 documentation for connection profiles](https://hyperledger.github.io/fabric-sdk-node/release-2.2/tutorial-commonconnectionprofile.html)
- node sdk 1.4 documentation for connection profiles

Unfortunately the documentation provided by Hyperledger Fabric is more focused on static connection profiles rather than dynamic connection profiles and your aim should be to create the simpler and smaller dynamic connection profile.

With the introduction of using the Peer Gateway rather than the traditional node sdks (1.4 and 2.2) caliper has introduced the concept of declaring peers in an organization within the network configuration file as an alternative to connection profiles. This provides a simple way to describe either peers to discover from (when binding to Fabric 1.4 or 2.2, for Fabric 1.4 you must enable the gateway option as it won’t work otherwise as discovery is not supported with the Fabric 1.4 binding when the gateway option is not enabled) or the peer to be used as a gateway into the Fabric network (when binding to Fabric 2.4). An example of a peers section in the network configuration is

```sh
peers:
      - endpoint: peer0.org3.example.com:7051
        tlsCACerts:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
        grpcOptions:
          grpc.keepalive_timeout_ms: 20000
          grpc.keepalive_time_ms: 120000
          grpc.http2.min_time_between_pings_ms: 120000
          grpc.http2.max_pings_without_data: 0
          grpc.keepalive_permit_without_calls: 1
```

## Runtime settings

### Common settings
Some runtime properties of the adapter can be set through Caliper’s [runtime configuration mechanism](https://hyperledger.github.io/caliper/v0.5.0/reference/runtime-config/). For the available settings, see the `caliper.fabric` section of the [default configuration file](https://github.com/hyperledger/caliper/blob/v0.5.0/packages/caliper-core/lib/common/config/default.yaml) and its embedded documentation.

The above settings are processed when starting Caliper. Modifying them during testing will have no effect. However, you can override the default values *before Caliper* starts from the usual configuration sources. In the following example the `localhost` property applies only when binding with Fabric 2.2 or Fabric 1.4 (and only if the `gateway` option is enabled)

!!!note

    *An object hierarchy in a configuration file generates a setting entry for every leaf property. Consider the following configuration file:*
    ```sh
    caliper:
        fabric:
            gateway:
              localhost: false
    ```
    *After naming the [project settings](https://hyperledger.github.io/caliper/v0.5.0/reference/runtime-config/#project-level) file `caliper.yaml` and placing it in the root of your workspace directory, it will override the following two setting keys with the following values:*

    - *Setting `caliper-fabric-gateway-localhost` is set to false*

    ***The other settings remain unchanged.***

    Alternatively you can change this setting when you launch caliper with the CLI options of

    `--caliper-fabric-gateway-localhost false`

## The connector API
The [workload modules](https://hyperledger.github.io/caliper/v0.5.0/overview/workload-module/) interact with the adapter at three phases of the tests: during the initialization of the user module (in the `initializeWorkloadModule` callback), when submitting invoke or query transactions (in the `submitTransaction` callback), and at the optional cleanup of the user module (in the `cleanupWorkloadModule` callback).

### The `initializeWorkloadModule` function

See the [corresponding documentation](https://hyperledger.github.io/caliper/v0.5.0/overview/workload-module/#initializeworkloadmodule) of the function for the description of its parameters.

The last argument of the function is a `sutContext` object, which is a platform-specific object provided by the backend blockchain’s connector. The context object provided by this connector is a `FabricConnectorContext` instance but this doesn’t provide anything of use at this time.

For the current details/documentation of the API, refer to the [source code](https://github.com/hyperledger/caliper/blob/v0.5.0/packages/caliper-fabric/lib/FabricConnectorContext.js).

### The `submitTransaction` function

The `sutAdapter` object received (and saved) in the `initializeWorkloadModule` function is of type `[ConnectorInterface](https://github.com/hyperledger/caliper/blob/v0.5.0/packages/caliper-core/lib/common/core/connector-interface.js)`. Its `getType()` function returns the `fabric` string value.

The `sendRequests` method of the connector API allows the workload module to submit requests to the SUT. It takes a single parameter: an object or array of objects containing the settings of the requests.

The settings object has the following structure:

- `contractId`: string. Required. The ID of the contract to call. This is either the unique contractID specified in the network configuration file or the chaincode ID used to deploy the chaincode and must match the id field in the contacts section of channels in the network configuration file.
- `contractFunction`: string. Required. The name of the function to call in the contract.
- `contractArguments`: string[]. Optional. The list of **string** arguments to pass to the contract.
- `readOnly`: boolean. Optional. Indicates whether the request is a TX or a query. Defaults to `false`.
- `transientMap`: Map<string, byte[]>. Optional. The transient map to pass to the contract.
- `invokerIdentity`: string. Optional. The name of the user who should invoke the contract. If not provided, a user will be selected from the organization defined by `invokerMspId` or the first organization in the network configuration file if that property is not provided.
- `invokerMspId`: string. Optional. The mspid of the user organization who should invoke the contract. Defaults to the first organization in the network configuration file.
- `targetPeers`: string[]. Optional. An array of endorsing peer names as the targets of the transaction proposal. If omitted, the target list will be chosen for you and if discovery is used then the node SDK uses discovery to determine the correct peers.
- `targetOrganizations`: string[]. Optional. An array of endorsing organizations as the targets of the invoke. If both targetPeers and targetOrganizations are specified, then targetPeers will take precedence.
- `channel`: string. Optional. The name of the channel on which the contract to call resides.
- `timeout`: number. Optional. [**Only applies to 1.4 binding when not enabling gateway use**] The timeout in seconds to use for this request.
- `orderer`: string. Optional. [**Only applies to 1.4 binding when not enabling gateway use**] The name of the target orderer for the transaction broadcast. If omitted, then an orderer node of the channel will be automatically selected.

So invoking a contract looks like the following:

```sh
let requestSettings = {
    contractId: 'marbles',
    contractFunction: 'initMarble',
    contractArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com',
    timeout: 10
};

await this.sutAdapter.sendRequests(requestSettings);
```

!!! note

    *`sendRequests` also accepts an array of request settings. However, Fabric does not support submitting an atomic batch of transactions like Sawtooth, so there is no guarantee that the order of these transactions will remain the same, or whether they will reside in the same block.*

## Gathered TX data
The previously discussed `sendRequests` function returns the result (or an array of results) for the submitted request(s) with the type of [TxStatus](https://github.com/hyperledger/caliper/blob/v0.5.0/packages/caliper-core/lib/transaction-status.js). The class provides some standard and platform-specific information about its corresponding transaction.

The standard data provided are the following:
- `GetID():string` returns the transaction ID.
- `GetStatus():string` returns the final status of the transaction, either `success` or `failed`.
- `GetTimeCreate():number` returns the epoch when the transaction was submitted.
- `GetTimeFinal():number` return the epoch when the transaction was finished.
- `IsVerified():boolean` indicates whether we are sure about the final status of the transaction. Unverified (considered failed) transactions could occur, for example, if the adapter loses the connection with every Fabric event hub, missing the final status of the transaction.
- `GetResult():Buffer` returns one of the endorsement results returned by the chaincode as a `Buffer`. It is the responsibility of the user callback to decode it accordingly to the chaincode-side encoding.

The adapter also gathers the following platform-specific data (if observed) about each transaction, each exposed through a specific key name. The placeholders `<P>` and `<O>` in the key names are node names taking their values from the top-level peers and orderers sections from the network configuration file (e.g., `endorsement_result_peer0.org1.example.com`). The `Get(key:string):any` function returns the value of the observation corresponding to the given key. Alternatively, the `GetCustomData():Map<string,any>` returns the entire collection of gathered data as a `Map`.

### Available data keys for all Fabric SUTs
The adapter-specific data keys that are available when binding to any of the Fabric SUT versions are :

| Key name         | Data type   | Description                                                                                 |
|------------------|-------------|---------------------------------------------------------------------------------------------|
| `request_type`   | string      | Either the `transaction` or `query` string value for traditional transactions or queries, respectively. |


## Available data keys for the Fabric 1.4 SUT when gateway is not enabled
The adapter-specific data keys that only the v1.4 SUT when not enabling the gateway makes available are :

| Key name                           | Data type    | Description                                                                                          |
|------------------------------------|--------------|------------------------------------------------------------------------------------------------------|
| `time_endorse`                     | number       | The Unix epoch when the adapter received the proposal responses from the endorsers. Saved even in the case of endorsement errors.                                  |
| `proposal_error`                   | string       | The error message in case an error occurred during sending/waiting for the proposal responses from the endorsers.                                                  |
| `proposal_response_error_<P>`      | string       | The error message in case the endorser peer `<P>` returned an error as endorsement result.                                                                        |
| `endorsement_result_<P>`           | Buffer       | The encoded contract invocation result returned by the endorser peer `<P>`. It is the user callback’s responsibility to decode the result.                        |
| `endorsement_verify_error_<P>`     | string       | Has the value of 'INVALID' if the signature and identity of the endorser peer `<P>` couldn’t be verified. This verification step can be switched on/off through the [runtime configuration options](https://hyperledger.github.io/caliper/v0.5.0/connector-configuration/fabric-config/#runtime-settings). |
| `endorsement_result_error<P>`      | string       | If the transaction proposal or query execution at the endorser peer `<P>` results in an error, this field contains the error message.                             |
| `read_write_set_error`             | string       | Has the value of 'MISMATCH' if the sent transaction proposals resulted in different read/write sets.                                                              |
| `time_orderer_ack`                 | number       | The Unix epoch when the adapter received the confirmation from the orderer that it successfully received the transaction. Note, that this isn’t the actual ordering time of the transaction.                      |
| `broadcast_error_<O>`              | string       | The warning message in case the adapter did not receive a successful confirmation from the orderer node `<O>`.                                                   |
| `broadcast_response_error_<O>`     | string       | The error message in case the adapter received an explicit unsuccessful response from the orderer node `<O>`.                                                   |
| `unexpected_error`                 | string       | The error message in case some unexpected error occurred during the life-cycle of a transaction.                                                                 |
| `commit_timeout_<P>`               | string       | Has the value of `'TIMEOUT'` in case the event notification about the transaction did not arrive in time from the peer node `<P>`.                                |
| `commit_error_<P>`                 | string       | Contains the error code in case the transaction validation fails at the end of its life-cycle on peer node `<P>`.                                                |
| `commit_success_<P>`               | number       | The Unix epoch when the adapter received a successful commit event from the peer node `<P>`. Note, that transactions committed in the same block have nearly identical commit times, since the SDK receives them block-wise, i.e., at the same time. |
| `event_hub_error_<P>`              | string       | The error message in case some event hub connection-related error occurs with peer node `<P>`.                                                                   |

You can access these data in your workload module after calling `sendRequests`:

```sh
let requestSettings = {
    contractId: 'marbles',
    contractVersion: '0.1.0',
    contractFunction: 'initMarble',
    contractArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com',
    timeout: 10
};

// single argument, single return value
const result = await this.sutAdapter.sendRequests(requestSettings);

let shortID = result.GetID().substring(8);
let executionTime = result.GetTimeFinal() - result.GetTimeCreate();
console.log(`TX [${shortID}] took ${executionTime}ms to execute. Result: ${result.GetStatus()}`);
```

### The cleanupWorkloadModule function

The `cleanupWorkloadModule` function is called at the end of the round, and can be used to perform any resource cleanup required by your workload implementation.

## Network configuration file reference
The YAML network configuration file of the adapter mainly describes the organizations and the identities associated with those organizations, It also provides explicit information about the channels in your Fabric network and the chaincode (containing 1 or more smart contracts) deployed to those channels. It can reference Common Connection Profiles for each organization (as common connection profiles are specific to a single organization). These are the same connection profiles that would be consumed by the node-sdk. Whoever creates the Fabric network and channels would be able to provide appropriate profiles for each organization.

The following sections detail each part separately. For a complete example, please refer to the [example section](https://hyperledger.github.io/caliper/v0.5.0/connector-configuration/fabric-config/#network-configuration-example) or one of the files in the [Caliper repositor](https://github.com/hyperledger/caliper), such as the caliper-fabric test folder.

<details>
  <summary><b>name</b></summary>

  <i>Required. Non-empty string.</i>
   <br/>
   The name of the configuration file.
   ```sh
   name: Fabric
   ```
</details>

<details>
  <summary><b>version</b></summary>

  <i>Required. Non-empty string.</i>
   <br/>
    Specifies the YAML schema version that the Fabric SDK will use. Only the `'2.0.0'` string is allowed.
   ```sh
   version: '2.0.0'
   ```
</details>

<details>
  <summary><b>caliper</b></summary>

  <i>Required. Non-empty object.</i>
  <br/>

  Contains runtime information for Caliper. Can contain the following keys.

  <ul>
    <li>
      <details>
        <summary><b>blockchain</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        Only the <code>"fabric"</code> string is allowed for this adapter.

        ```sh
        caliper:
            blockchain: fabric
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>sutOptions</b></summary>

        <i>Required. Non-empty object.</i>
        <br/>
        These are sut specific options block, the following are specific to the Fabric implementation
        <ul>
        <li>
        <details>
        <summary><b>mutualTls</b></summary>

        <i>Optional. Boolean.</i>
        <br/>
        Indicates whether to use client-side TLS in addition to server-side TLS. Cannot be set to <pre><code>true</code></pre> without using server-side TLS. Defaults to <pre><code>false</code></pre>.
        ```sh
        caliper:
            blockchain: fabric
            sutOptions:
              mutualTls: true
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>command</b></summary>

        <i>Optional. Non-empty object.</i>
        <br/>
        Specifies the start and end scripts.
        <br/>
        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at least one</b> of the following keys.</p>
        </div>
        <ul>
          <li>
            <details>
              <summary><b>start</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              Contains the command to execute at startup time. The current working directory for the commands is set to the workspace.

              ```sh
              caliper:
                command:
                  start: my-startup-script.sh
              ```
            </details>
          </li>
          <li>
            <details>
              <summary><b>end</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              Contains the command to execute at exit time. The current working directory for the commands is set to the workspace.

              ```sh
              caliper:
                command:
                  end: my-cleanup-script.sh
              ```
            </details>
          </li>
        </ul>
      </details>
    </li>
  </ul>
</details>

<details>
  <summary><b>info</b></summary>

  <i>Optional. Object.</i>
   <br/>
    Specifies custom key-value pairs that will be included as-is in the generated report. The key-value pairs have no influence on the runtime behavior.

   ```sh
    info:
      Version: 1.1.0
      Size: 2 Orgs with 2 Peers
      Orderer: Solo
      Distribution: Single Host
      StateDB: CouchDB
   ```
</details>

<details>
  <summary><b>organizations</b></summary>

  <i>Required. Non-empty object.</i>
  <br/>
  Contains information about 1 or more organizations that will be used when running a workload. Even in a multi-organization Fabric network, workloads would usually only be run from a single organization so it would be common to only see 1 organization defined. However it does support defining multiple organizations for which a workload can explicitly declare which organization to use. The first Organization in the network configuration will be the default organization if no explicit organization is requested.

  ```sh
   organizations:
  - mspid: Org1MSP
    identities:
      wallet:
        path: './org1wallet'
        adminNames:
        - admin
      certificates:
      - name: 'User1'
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    connectionProfile:
      path: './Org1ConnectionProfile.yaml'
      discover: true
  - mspid: Org2MSP
    connectionProfile:
      path: './Org2ConnectionProfile.yaml'
      discover: false
    identities:
      wallet:
        path: './org2wallet'
        adminNames:
        - admin
  - mspid: Org3MSP
    peers:
      - endpoint: peer0.org3.example.com:7051
        tlsCACerts:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
        grpcOptions:
          grpc.keepalive_timeout_ms: 20000
          grpc.keepalive_time_ms: 120000
          grpc.http2.min_time_between_pings_ms: 120000
          grpc.http2.max_pings_without_data: 0
          grpc.keepalive_permit_without_calls: 1
  ```
  
  Each organization must have <code>mspid</code>, <code>identities</code> and either <code>connectionProfile</code> or <code>peers</code> provided and at least 1 certificate or wallet definition in the identities section so that at least 1 identity is defined
  <ul>
    <li>
      <details>
        <summary><b>mspid</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The unique MSP ID of the organization.

        ```sh
        organizations:
          - mspid: Org1MSP
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>connectionProfile</b></summary>

        <i>Required if <code>peers</code> not provided. Non-empty object.</i>
        <br/>
        Reference to a Fabric network Common Connection Profile. These profiles are the same profiles that the Fabric SDKs would consume in order to interact with a Fabric network. A Common Connection Profile is organization specific so you need to ensure you point to a Common Connection Profile that is representative of the organization it is being included under. Connection Profiles also can be in 2 forms. A static connection profile will contain a complete description of the Fabric network, ie all the peers and orderers as well as all the channels that the organization is part of. A dynamic connection profile will contain a minimal amount of information usually just a list of 1 or more peers belonging to the organization (or is allowed to access) in order to discover the Fabric network nodes and channels.

        ```sh
        organizations:
          - mspid: Org1MSP
            connectionProfile:
              path: './test/sample-configs/Org1ConnectionProfile.yaml'
              discover: true
        ```
        <li>
        <details>
        <summary><b>path</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The path to the connection profile file

        ```sh
        organizations:
          - mspid: Org1MSP
            connectionProfile:
              path: './test/sample-configs/Org1ConnectionProfile.yaml'
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>discover</b></summary>

        <i>Optional. Boolean.</i>
        <br/>
        A value of <pre><code>true</code></pre> indicates that the connection profile is a dynamic connection profile and discovery should be used. If not specified then it defaults to <pre><code>false</code></pre>. For a Fabric 1.4 binding you can only set this value to true if you plan to use the <pre><code>gateway</code></pre> option.

        ```sh
        organizations:
          - mspid: Org1MSP
            connectionProfile:
              path: './test/sample-configs/Org1ConnectionProfile.yaml'
              discover: true
        ```
        </details>
        </li>
      </details>
    </li>
    <li>
      <details>
        <summary><b>peers</b></summary>

        <i>Required if <code>connectionProfile</code> not provided. Non-empty object.</i>
        <br/>
        Reference to one or more peers that are either
        <ul>
        <li>a peer to discover the network from when bound to Fabric 2.2 or Fabric 1.4 in conjunction with using the gateway enabled option
        </li>
        <li>a gateway peer when bound to Fabric 2.4</li>
        </ul>

        This option removes the need for connection profiles but the Fabric network must be set up correctly to allow the network to be discovered. These entries are the equivalent of a dynamic connection profile but in a more compact and easier form.

        ```sh
        organizations:
          - mspid: Org3MSP
            peers:
              - endpoint: peer0.org3.example.com:7051
                tlsCACerts:
                  pem: |-
                    -----BEGIN CERTIFICATE-----
                    ...
                    -----END CERTIFICATE-----
                grpcOptions:
                  grpc.keepalive_timeout_ms: 20000
                  grpc.keepalive_time_ms: 120000
                  grpc.http2.min_time_between_pings_ms: 120000
                  grpc.http2.max_pings_without_data: 0
                  grpc.keepalive_permit_without_calls: 1
        ```
        <li>
        <details>
        <summary><b>endpoint</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        the end point of the peer in the form of <code>host:port</code> (note that you do not specify a schema such as grpc:// or grpcs://, in fact these schemas are not real and were invented purely for connection profiles). Whether the end point is secured by tls or not is determined by the presence of the <code>tlsCACerts</code> property

        ```sh
        peers:
          - endpoint: peer0.org3.example.com:7051
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>tlsCACerts</b></summary>
        <i>Optional. Non-empty object.</i>
        <br/>
        Specifies the tls root certificate chain to verify a TLS connection with the peer by the client
        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>
        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The path of the file containing the certificate chain.

        ```sh
        tlsCACerts:
          path: path/to/cert.pem
        ```

        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The content of the certificate file in exact PEM format (which must split into multiple lines for yaml or include escaped new lines for json).
        ```sh
        tlsCACerts:
           pem: |
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
        </details>
        </li>
        <li>
        <details>
        <summary><b>grpcOptions</b></summary>
        <i>Optional. Non-empty Object.</i>
        <br/>
        A set of grpc specific options when creating a grpc connection to a peer.
        ```sh
        peers:
          - endpoint: peer0.org3.example.com:7051
            grpcOptions:
              grpc.keepalive_timeout_ms: 20000
              grpc.keepalive_time_ms: 120000
              grpc.http2.min_time_between_pings_ms: 120000
              grpc.http2.max_pings_without_data: 0
              grpc.keepalive_permit_without_calls: 1
        ```
        </details>
        </li>
        
      </details>
    </li>
    <li>
      <details>
        <summary><b>identities</b></summary>

        <i>Required. Non-empty object.</i>
        <br/>
        Defines the location of 1 or more identities available for use. Currently only supports explicit identities by providing a certificate and private key as PEM or an SDK wallet that contains 1 or more identities on the file system. At least 1 identity must be provided via one of the child properties of identity.

        ```sh
        identities:
           wallet:
             path: './wallets/org1wallet'
             adminNames:
             - admin
           certificates:
           - name: 'User1'
             clientPrivateKey:
               pem: |-
                 -----BEGIN PRIVATE KEY-----
                 ...
                 -----END PRIVATE KEY-----
             clientSignedCert:
               pem: |-
                 -----BEGIN CERTIFICATE-----
                 ...
                 -----END CERTIFICATE-----
        ```
        <ul>
        <li>
        <details>
        <summary><b>certificates</b></summary>
        <i>Optional. A List of non-empty objects.</i>
        <br/>
        Defines 1 or more identities by providing the PEM information for the client certificate and client private key as either an embedded PEM, a base64 encoded string of the PEM file contents or a path to individual PEM files

        ```sh
        certificates:
        - name: 'User1'
          clientPrivateKey:
             path: path/to/privateKey.pem
          clientSignedCert:
             path: path/to/cert.pem
        - name: 'Admin'
          admin: true
          clientPrivateKey:
           pem: |-
            -----BEGIN PRIVATE KEY-----
            MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgIRZo3SAPXAJnGVOe
            jRALBJ208m+ojeCYCkmJQV2aBqahRANCAARnoGOEw1k+MtjHH4y2rTxRjtOaKWXn
            FGpsALLXfBkKZvxIhbr+mPOFZVZ8ztihIsZBaCuCIHjw1Tx65szJADcO
            -----END PRIVATE KEY-----
          clientSignedCert:
           pem: |-
             -----BEGIN CERTIFICATE-----
            MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
            CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
            YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
            Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
            NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
            Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
            VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
            AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
            rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
            JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
            xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
            fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
            H8Zua3x+ZQn/kqVv
            -----END CERTIFICATE-----
        - name: 'User3'
          clientPrivateKey:
           pem: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JR0hBZ0VBTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEJHMHdhd0lCQVFRZ0lSWm8zU0FQWEFKbkdWT2UKalJBTEJKMjA4bStvamVDWUNrbUpRVjJhQnFhaFJBTkNBQVJub0dPRXcxaytNdGpISDR5MnJUeFJqdE9hS1dYbgpGR3BzQUxMWGZCa0tadnhJaGJyK21QT0ZaVlo4enRpaElzWkJhQ3VDSUhqdzFUeDY1c3pKQURjTwotLS0tLUVORCBQUklWQVRFIEtFWS0tLS0tCg==
          clientSignedCert:
           pem: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNXRENDQWY2Z0F3SUJBZ0lSQU1wU2dXRmpESE9vaFhhMFI2ZTlUSGd3Q2dZSUtvWkl6ajBFQXdJd2RqRUwKTUFrR0ExVUVCaE1DVlZNeEV6QVJCZ05WQkFnVENrTmhiR2xtYjNKdWFXRXhGakFVQmdOVkJBY1REVk5oYmlCRwpjbUZ1WTJselkyOHhHVEFYQmdOVkJBb1RFRzl5WnpFdVpYaGhiWEJzWlM1amIyMHhIekFkQmdOVkJBTVRGblJzCmMyTmhMbTl5WnpFdVpYaGhiWEJzWlM1amIyMHdIaGNOTWpBd09UQTNNVEUwTWpBd1doY05NekF3T1RBMU1URTAKTWpBd1dqQjJNUXN3Q1FZRFZRUUdFd0pWVXpFVE1CRUdBMVVFQ0JNS1EyRnNhV1p2Y201cFlURVdNQlFHQTFVRQpCeE1OVTJGdUlFWnlZVzVqYVhOamJ6RVpNQmNHQTFVRUNoTVFiM0puTVM1bGVHRnRjR3hsTG1OdmJURWZNQjBHCkExVUVBeE1XZEd4elkyRXViM0puTVM1bGVHRnRjR3hsTG1OdmJUQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDkKQXdFSEEwSUFCTWRMdlNVRElqV1l1Qnc0WVZ2SkVXNmlmRkx5bU9BWDdHS1k2YnRWUERsa2RlSjh2WkVyWExNegpKV2ppdnIvTDVWMlluWnF2ME9XUE1NZlB2K3pIK1JHamJUQnJNQTRHQTFVZER3RUIvd1FFQXdJQnBqQWRCZ05WCiBIU1VFRmpBVUJnZ3JCZ0VGQlFjREFnWUlLd1lCQlFVSEF3RXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QXBCZ05WCkhRNEVJZ1FnNWZPaHl6d2FMS20zdDU0L0g0YjBhVGU3L25HUHlKWk5oOUlGUks2ZkRhQXdDZ1lJS29aSXpqMEUKQXdJRFNBQXdSUUloQUtFbnkvL0pZN0dYWi9USHNRSXZVVFltWHNqUC9iTFRJL1Z1TFg3VHpjZWZBaUJZb1N5WQp5OTByZHBySTZNcDZSUGlxalZmMDJQNVpDODZVa1AwVnc0cGZpUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
        ```
        <ul>
        <li>
        <details>
        <summary><b>name</b></summary>
        <i>Required. Non-empty string.</i>
        <br/>
        Specifies a name to associate with this identity. This name doesn’t have to match anything within the certificate itself but must be unique

        ```sh
        certificates:
          - name: 'User1'
        ```
        </li>
        <li>
        <details>
        <summary><b>admin</b></summary>
        <i>Optional. Boolean.</i>
        <br/>
        Indicates if this identity can be considered an admin identity for the organization. Defaults to false if not provided This only needs to be provided if you plan to create channels and/or install and instantiate contracts (chaincode)

        ```sh
        certificates:
          - name: 'User2'
            admin: true
        ```
        </li>
        <li>
        <details>
        <summary><b>clientPrivateKey</b></summary>
        <i>Required. Non-empty object.</i>
        <br/>
        Specifies the identity’s private key for the organization.
        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>
        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The path of the file containing the private key
        ```sh
         clientPrivateKey:
            path: path/to/cert.pem
        ```
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The content of the private key file either in exact PEM format (which must split into multiple lines for yaml, or contain newline characters for JSON), or it could be a base 64 encoded version of the PEM (which will also encode the required newlines) as a single string. This single string format makes it much easier to embed into the network configuration file especially for a JSON based file
        ```sh
        clientPrivateKey:
           pem: |
             -----BEGIN PRIVATE KEY-----
              MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgIRZo3SAPXAJnGVOe
              jRALBJ208m+ojeCYCkmJQV2aBqahRANCAARnoGOEw1k+MtjHH4y2rTxRjtOaKWXn
              FGpsALLXfBkKZvxIhbr+mPOFZVZ8ztihIsZBaCuCIHjw1Tx65szJADcO
              -----END PRIVATE KEY-----
        ```
        </li>
        </ul>
        </li>
        <li>
        <details>
        <summary><b>clientSignedCert</b></summary>
        <i>Required. Non-empty object.</i>
        <br/>
        Specifies the identity’s certificate for the organization.
        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>
        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The path of the file containing the certificate
        ```sh
         clientSignedCert:
            path: path/to/cert.pem
        ```
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>
        The content of the certificate file either in exact PEM format (which must split into multiple lines for yaml, or contain newline characters for JSON), or it could be a base 64 encoded version of the PEM (which will also encode the required newlines) as a single string. This single string format makes it much easier to embed into the network configuration file especially for a JSON based file

        ```sh
        clientSignedCert:
           pem: |
             -----BEGIN CERTIFICATE-----
              MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
              CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
              YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
              Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
              NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
              Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
              VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
              AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
              rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
              JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
              xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
              fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
              H8Zua3x+ZQn/kqVv
              -----END CERTIFICATE-----
        ```
        </li>
        </ul>
        </li>
        </ul>
        </details>
        </li>
        <li>
        <details>
        <summary><b>wallet</b></summary>
        <i>Optional. Non-empty object.</i>
        <br/>
        Provide the path to a file system wallet. Be aware that the persistence format used between v1.x and v2.x of the node sdks changed so make sure you provide a wallet created in the appropriate format for the version of SUT you bind to.
        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Required. Non-empty string.</i>
        <br/>
        The path to the file system wallet
        ```sh
        identities:
          wallet:
            path: './wallets/org1wallet'
        ```
        </li>
        <li>
        <details>
        <summary><b>adminNames</b></summary>
        <i>Optional. List of strings.</i>
        <br/>
        1 or more names in the wallet that are identified as organization administrators. This only needs to be provided if you plan to create channels and/or install and instantiate contracts (chaincode)
        ```sh
        identities:
          wallet:
            path: './wallets/org1wallet'
            adminNames:
            - admin
            - another_admin
        ```
        </li>
        </ul>
        </details>
        </li>
        </ul>
      </details>
    </li>
    </ul>
</details>

<details>
  <summary><b>channels</b></summary>

  <i>Required. A list of objects.</i>
  <br/>
  Contains one or more unique channels with associated information about the chaincode (contracts section) that will be available on the channel

  ```sh
  channels:
  - channelName: mychannel
    contracts:
    - id: marbles
      contractID: myMarbles

  - channelName: somechannel
    contracts:
    - id: basic
  ```
  <ul>
    <li>
      <details>
        <summary><b>channelName</b></summary>

        <i>Required. Non-empty String.</i>
        <br/>
        The name of the channel.

        ```sh
        channels:
          - channelName: mychannel
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>contracts</b></summary>
        <i>Required. Non-sparse array of objects.</i>
        <br/>
        Each array element contains information about a chaincode deployed to the channel.
        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>the <code>contractID</code> value of <b>every</b> contract in <b>every</b> channel must be unique on the configuration file level! If <code>contractID</code> is not specified for a contract then its default value is the <code>id</code> of the contract.</p>
        </div>

        ```sh
        channels:
          mychannel
            contracts:
            - id: simple
            - id: smallbank
        ```
        <li>
        <details>
        <summary><b>id</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The chaincode ID that was specified when the chaincode was deployed to the channel

        ```sh
        channels:
          mychannel
            contracts:
            - id: simple
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>contractID</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The Caliper-level unique ID of the contract. This ID will be referenced from the user callback modules. Can be an arbitrary name, it won’t effect the contract properties on the Fabric side.
        <br/>
        If omitted, it defaults to the <code>id</code> property value.

        ```sh
        channels:
          mychannel
            contracts:
            - id: simple
            - contractID: simpleContract
        ```
        </details>
        </li>
      </details>
    </li>
    </ul>
</details>

## Network Configuration Example
The following example is a Fabric network configuration for the following network topology and artifacts:

- two organizations `Org1MSP` and `Org2MSP` (Note that having 2 organizations is not common in a network configuration file);
- one channel named `mychannel`;
- `asset-transfer-basic` chaincode deployed to `mychannel` with a chaincode id of `basic`;
- the nodes of the network use TLS communication, but not mutual TLS;
- the Fabric samples test network is started and terminated automatically by Caliper;

```sh
name: Fabric
version: "2.0.0"

caliper:
  blockchain: fabric
  sutOptions:
    mutualTls: false
  command:
    start: ../fabric-samples/test-network/network.sh up createChannel && ../fabric-samples/test-network/network.sh deployCC -ccp ../fabric-samples/asset-transfer-basic/chaincode-javascript -ccn basic -ccl javascript
    end: ../fabric-samples/test-network/network.sh down

info:
  Version: 1.1.0
  Size: 2 Orgs
  Orderer: Raft
  Distribution: Single Host
  StateDB: GoLevelDB

channels:
  - channelName: mychannel
    contracts:
    - id: basic
      contractID: BasicOnMyChannel

organizations:
  - mspid: Org1MSP
    identities:
      certificates:
      - name: 'admin.org1.example.com'
        admin: true
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    connectionProfile:
      path: './Org1ConnectionProfile.yaml'
      discover: true
  - mspid: Org2MSP
    connectionProfile:
    identities:
      certificates:
      - name: 'admin.org2.example.com'
        admin: true
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
      path: './Org2ConnectionProfile.json'
      discover: true
```
Another example with only a single organization but using the peers property so everything required is contained in a single network configuration file:

```sh
name: Fabric
version: "2.0.0"

caliper:
  blockchain: fabric
  sutOptions:
    mutualTls: false

channels:
  - channelName: mychannel
    contracts:
    - id: basic

organizations:
  - mspid: Org1MSP
    identities:
      certificates:
      - name: 'admin.org1.example.com'
        admin: true
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    peers:
      - endpoint: peer0.org1.example.com:7051
        grpcOptions:
          ssl-target-name-override: peer0.org1.example.com
          grpc.keepalive_time_ms: 600000
        tlsCACerts:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
```            

## License

The Caliper codebase is released under the [Apache 2.0 license](https://hyperledger.github.io/caliper/v0.5.0/general/license/). Any documentation developed by the Caliper Project is licensed under the Creative Commons Attribution 4.0 International License. You may obtain a copy of the license, titled CC-BY-4.0, at [http://creativecommons.org/licenses/by/4.0/](http://creativecommons.org/licenses/by/4.0/).