# auto-updater-setup

## How to setup parity auto-updater

### 1. On-chain setup
 
 
 #### Create accounts
 `Master key`. Key that will own contracts. <br>
 `Server key`. This is the key which Release service will use to push stable, beta and nightly releases to the Operations contracts. However, stable and beta releases need a confirmation from the manual key. <br>
 `Manual key`. Can be used to confirm stable and beta releases. This is the key which generally stays offline. <br>
 
 Add some POA to the accounts.
 
 #### Deploy smart contracts:
 All contracts can be deployed from the Master key. <br>
 There are repositories with contracts:
 [Registry](https://github.com/parity-contracts/name-registry), [Operations and OperationsProxy](https://github.com/parity-contracts/auto-updater), [GitHubHint](https://github.com/parity-contracts/github-hint). 
 - Specify [network parameters](https://github.com/natlg/name-registry/blob/68e3a4e767e585b45f73cff358b38324abaacfe0/truffle-config.js#L5-L11).
 - Deployment parameters for the OperationsProxy contact: <br>
  `owner`: `master key`  <br>
  `stable`: `server key`  <br>
  `beta`: `server key` <br>
  `nightly`: `server key` <br>
  `stableConfirmer`: `manual key` <br>
  `betaConfirmer`: `manual key` <br>
  `nightlyConfirmer`: `<null>` <br>
  `_operations`: address of Operations contracts <br>
  
  You can find an example [here](https://github.com/natlg/auto-updater/blob/b905a52b556c4ebb6dca4ffb38c708295b72e9af/migrations/2_deploy_contracts.js#L10-L20). <br>
                              
  Confirmers can be leaved empty (null value), in this case manual confirmation will not be needed. 
  But it's safer to confirm at least stable and beta releases.   <br>

 - With truffle you can run `truffle migrate --network sokol --verbose-rpc` for the each contract. First deploy SimpleOperations, then OperationsProxy with address of SimpleOperations contract as parameter.<br> 
 - Save addresses of deployed contracts.


 #### Register contracts
 
 ###### OperationsProxy:
   - call `reserve`  function of `Registrar` contracts, pass `Keccak-256 hash` of `“parityoperations”` (0x760526ad131d94f10ee75b247d523cf79f647a5ca5f1c06321d4ff9595c289f7).
   It’s defined in the Release service https://github.com/paritytech/push-release/blob/master/server.js#L63 , but you can change this value.
   This function is payable, so send 1 POA.
   
   - call `setAddress` function of `Registrar` contracts, pass parameters: <br>
   `_name`:  0x760526ad131d94f10ee75b247d523cf79f647a5ca5f1c06321d4ff9595c289f7 (name hash) <br>
   `_key`: A <br>
   `_value`: OperationsProxy contract address <br>
   
 ###### Operations:
 - call `reserve` function of `Registrar` contracts, pass hash of `“operations”` (0x51ca38ad0d9c8193e2a97a5c1967a34e79112bdff5a5783aad187224fb70165e). Send 1 POA.
 
 - call `setAddress` function of `Registrar` contracts, pass parameters: <br>
 `_name`:  0x51ca38ad0d9c8193e2a97a5c1967a34e79112bdff5a5783aad187224fb70165e (name hash) <br>
 `_key`: A <br>
 `_value`: Operations contract address <br>
 
###### GitHubHint:
 - call `reserve`  function of `Registrar` contracts, pass hash of `“githubhint”` (0x058740ee9a5a3fb9f1cfa10752baec87e09cc45cd7027fd54708271aca300c75). Send 1 POA.
 - call `setAddress` function of Registrar contracts, pass parameters: <br>
 `_name`:  0x058740ee9a5a3fb9f1cfa10752baec87e09cc45cd7027fd54708271aca300c75 (name hash) <br>
 `_key`: A <br>
 `_value`: GitHubHint contract address <br>
 
 
 #### Set up OperationsProxy contract
 
 Configure Parity's OperationsProxy to be the maintainer of Parity client releases in Operations. <br>
 Call `setClientOwner` function of Operations contract from `Master key`  account, pass address of OperationsProxy contract.

 
### 2. Parity setup:
- Add address of the Registry contract to the [spec](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/ethcore/res/ethereum/sokol.json#L31) file of the new network (`registrar` parameter). 
This contract is used by Parity and Release service for getting addresses of all other contracts. 
- Update [util/version/Cargo.toml](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/util/version/Cargo.toml#L22) with information about the network. <br>
Release service checks this file after receiving details about new release. 
- Add Release service call as in these files: [gitlab-push-release.sh](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/scripts/gitlab-push-release.sh#L14-L15), 
 [gitlab-build.sh](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/scripts/gitlab-build.sh#L159-L160), just change domain name `update.parity.io` to the address of your Release service.
 If you don't own smart contracts on other networks, remove pushing release for them. 
 
### 3. GitLab setup
- Add `secret` environment variable. It's an authentication token for requests to the Release service
 
### 4. Release service setup
See deployment instructions and some more information about [Release service](https://github.com/paritytech/push-release#deployment).
- Add configuration for the new network (example files [config/sokol-sample.json](https://github.com/natlg/push-release/blob/04faa80b63b020649ba2d53047523ca3bed07e2d/config/sokol-sample.json) and [push-release.json](https://github.com/natlg/push-release/blob/04faa80b63b020649ba2d53047523ca3bed07e2d/push-release.json#L30-L43))
Specify parameters: <br>
`secretHash`:  a Keccak-256 hash of token stored in GitLab. <br>
account `address`: The address of the account that will be used to send transactions  <br>
`ACCOUNT_PASSWORD`:  The password of the account. If no password is supplied, it is assumed that the account is already unlocked. <br>
`http_port` The HTTP port the service will be running on. <br>
`rpc_port` The port of Parity's JSON-RPC HTTP interface. <br>
You can also override other parameters by environment variables, see `config/sokol-sample.json` for more options.


## Launch

Run Parity on the same machine with Release service: 

```
parity --jsonrpc-port 8545  --unlock "0x..Server key" --password ./pass --chain ../poa-chain-spec/spec.json --warp --reserved-peers ../poa-chain-spec/bootnodes.txt --jsonrpc-apis all
```

Run Release service: <br>
`pm2 start push-release.json --only push-release-sokol` <br>

Check Service logs: <br>
`pm2 show push-release-sokol` <br>
and <br>
`pm2 logs push-release-sokol --lines 1000` <br>

If need to stop server: <br>
`pm2 stop all` <br>

---

###### Quick check: <br>
- Update version, add new tag to launch pipeline.
- Run some old version of Parity with cli options
 `--auto-update=all --release-track=nightly  --auto-update-check-frequency=1 -l updater=trace`.
- Wait until it's fully synchronized and all jobs in GitLab are finished, then Parity will start updating. <br><br>

If updating didn't start, check logs on the Release service, if all requests were received, address of received contracts. Check all transactions in the Explorer. <br>

See if release is added to the Operations contract:<br>
Call `latestInTrack` function with parameters `_client`: `parity`, `_track`: `3` (for the `nightly` track). 
It must return the hash of commit where tag was added (`0x000000000000000000000000` + hash). <br>
Call `release` function, pass `parity` and hash returned from the `latestInTrack` function. See the result.

See if download url is added to the GutHubHint: <br>
Call `entries` function on the GutHubHint contract, pass hash of build (`sha3` parameter from the push-build request). Don't forget to add `0x` in the beginning.
Download url must be returned. <br>


If release exists on the chain, check settings of Parity then: address of Register contract in the spec file (make sure it's the same as for Parity running with Release service), 
version, track. Any problems with blocks synchronization can stop update too.

## How it works

First step of CI includes tests, then script from Parity sends information about new release version to the Release service (Push release step). After builds for each platforms are ready, Parity script sends information about builds (Push build step). Release service saves all received data to smart contracts, so during auto update Parity can get all necessary information about new version from the chain (including link for downloading).

### Push release 

It's second step of CI after tests. Information about new version will be saved to the smart contract during this step.
<br>
Example request to the Release service from Parity: <br>
`curl --data "secret=$SECRET" http://update-server.parity.io:1337/push-release/$BRANCH/$COMMIT`, <br>
where <br>
`SECRET` - an authentication token, should be passed as a 64-digit, hex-encoded POST parameter. <br>
The Keccak-256 hash of this token should be stored in config of the release service.<br>
`COMMIT` - 40-digit hex-encoded Git commit hash of this release. <br>
`BRANCH` - the release branch name, should be one of stable, beta, nightly (equivalent to master) or testing.
<br> <br>
Release service receives request and performs the following steps:
1. Parameters check (`BRANCH` is checked as tag, allowed only `v{number}.{number}.{number}` format or `nightly` value).
2. Parity metadata checking. Release service reads `/util/version/Cargo.toml` file from https://raw.githubusercontent.com on received commit. <br> This file contains information about track, fork block, semver (counts from version number as major * 65536 + minor * 256 + patch), `critical` setting.
3. Get Registry contract address using json rpc request `parity_registryAddress` to the Parity.
4. Get OperationsProxy contract address using `getAddress` function of Registry contract with parameters `_name`: `operationsproxy` and `_key`: `A`.
5. Call `addRelease` function of OperationsProxy contract and pass information about the release: `_release`: commit hash, `_forkBlock`: fork block, `_track`: track, `_semver`: semver, `_critical`: critical setting. <br>
OperationsProxy then either redirects call to the Operations contract or saves release until confirmation.  


### Push build
Execute after builds for each supported platform are ready and saved to the S3 BUCKET. Also builds require confirming (using `OperationsProxy` contract) prior to becoming active.
<br> <br>
Example request:

```
curl --data "commit=$COMMIT&sha3=$SHA3&filename=$FILENAME&secret=$SECRET" http://update-server.parity.io:1337/push-build/$BRANCH/$PLATFORM
```

Steps on the release service:<br/>
1, 2, 3 - Same as for Push release  step. <br>
4\. Get address of the GithubHint contract from the Registry contract.<br/>
5\. Call `hintURL` function of GithubHint contract with parameters `_content`: sha3 (hash of content) and `_url`: url. Format for url is `${baseUrl}/${tag}/${platform}/${filename}`. <br>
6\. Add checksum to the Operations contract (`addChecksum` function, send parameters: `0x000000000000000000000000${commit}`, `platform`, `0x${sha3}`). <br>

### Auto updating
Parity updater uses Operations contract for getting information about new releases and GithubHint contract for getting download url. 
Addresses of these contracts are stored in the Registry contract and address of Registry contract is supposed to be stored in the `spec` file
for every network. <br> <br>
Steps performed by Parity updater:
1. Get fork number of this release: call `release` function of Operations contract, pass parameters `_client` (parity) and `_release` (commit hash of current version), it returns forkBlock, track, semver, critical setting
2. Get the hash of the latest release in our track: call `latestInTrack` function of Operations contract, pass parameters `_client` and `_track`. It returns the latest saved release (commit hash of new release)
3. Get all release informatiom (fork, track, semver, is_critical) using `release` function of Operations contract
4. Get hash of release binary: call `checksum` function of Operations contract, pass  parameters `_client`, `_release`, `_platform` parameters.
5. Check if fork is supported.
6. Get download url: call `entries` of GithubHint contract, pass received hash.
7. Download binary (more likely to the folder `/home/user/.local/share/io.parity.ethereum-updates`) and upgrade.

