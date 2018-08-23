# auto-updater-setup

## How to setup parity auto-updater

1. Deploy smart contracts.
 TODO
2. Parity setup:
- Add address of the Registry contract to the [spec](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/ethcore/res/ethereum/sokol.json#L31) file of the new network (`registrar` parameter). 
This contract is used by Parity and Release service for getting addresses of all other contracts. 
- Update [util/version/Cargo.toml](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/util/version/Cargo.toml#L22) with information about the network.. <br>
Release service checks this file after receiving details about new release. 
- Add Release service call as in these files: [gitlab-push-release.sh](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/scripts/gitlab-push-release.sh#L14-L15), 
 [gitlab-build.sh](https://github.com/poanetwork/parity/blob/38892f969c3edfa81f15dcc84dc27b82e20b5b33/scripts/gitlab-build.sh#L159-L160)
 
3. Release service setup:
- Add configuration for the new network (files [config/sokol-sample.json](https://github.com/natlg/push-release/blob/04faa80b63b020649ba2d53047523ca3bed07e2d/config/sokol-sample.json) and [push-release.json](https://github.com/natlg/push-release/blob/04faa80b63b020649ba2d53047523ca3bed07e2d/push-release.json#L30-L43))


## Launch

Run Parity: 

```
parity --jsonrpc-port 8545  --unlock "0x.." --password ./pass --chain ../poa-chain-spec/spec.json --warp --reserved-peers ../poa-chain-spec/bootnodes.txt --jsonrpc-apis all
```

Run Release service: <br>
`pm2 start push-release.json --only push-release-sokol` <br>

Check Service logs: <br>
`pm2 show push-release-sokol` <br>


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
4. Get Operations contract address using `getAddress` function of Registry contract with parameters `_name`: `operations` and `_key`: `A`.
5. Call `addRelease` function of Operations contract and pass information about the release: `_release`: commit hash, `_forkBlock`: fork block, `_track`: track, `_semver`: semver, `_critical`: critical setting.


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
- Parity updater periodically checks the latest version for the specified track using Operations contract <br>
Updater uses Registry contract for getting Operations's address, so it's address must be specified in the `spec` file of current network as `registrar` parameter).
- If version matches, calculates it's hash and sends it to the GithubHint contract (it's address must be stored in the Registry contract too).<br> 
Download URL will be returned, then updater can upgrade Parity.
