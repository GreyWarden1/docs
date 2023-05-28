# Troubleshooting



## My contract is using OpenZeppelin. How do I add GSN support?

Here is a sample on how to use GSN with OpenZeppelin contracts.
The `_msgData` and `_msgSender` methods methods of OpenZeppelin's `Context` base-contract
are used by all other contracts, and thus the hooks below are enough to support tokens, Ownable, roles, etc.

(Note that OpenZeppelin's `GSNRecipient` is for GSN v1.0, and should not be used anymore)

```solidity
pragma solidity ^0.8;

import "@opengsn/contracts/src/BaseRelayRecipient.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyContract is BaseRelayRecipient, ERC20 {

  constructor(string memory name_, string memory symbol_, address forwarder_) 
    ERC20(name_, symbol_) {
    _setTrustedForwarder(forwarder_);
  }

  string public override versionRecipient = "2.2.0";

  function _msgSender() internal view override(Context, BaseRelayRecipient)
      returns (address sender) {
      sender = BaseRelayRecipient._msgSender();
  }

  function _msgData() internal view override(Context, BaseRelayRecipient)
      returns (bytes calldata) {
      return BaseRelayRecipient._msgData();
  }
}
```

You can use instead OpenZeppelin's `ERC2771Context`, which is an equivalent implementation of BaseRelayRecipient.

## My project is built using Ethers.js v6, and I am getting an exception `ethereum.request is not a function` when trying to create a `BrowserProvider` that wraps the GSN `RelayProvider`

Note: Native support for Ethers.js is added GSN v3 starting with build `3.0.0-beta.8`.

The Ethers.js v6 expects the external provider passed in the constructor of the `BrowserProvider` to be EIP-1193
compatible, but GSN v2 provider does not implement `request()` function.

In most cases all issues can be solved by adding the `request()` function to the GSN RelayProvider instance at runtime:

```typescript
import { JsonRpcResponse } from 'web3-core-helpers'
import { BrowserProvider } from 'ethers'

/* ******* */

// @ts-ignore
gsnProvider.request = function (payload: any) {
  return new Promise((resolve, reject) => {
    gsnProvider.send(payload, function (error: Error | null, result?: JsonRpcResponse) {
      if (error) {
        reject(error)
      } else {
        resolve(result?.result)
      }
    })
  })
}
const provider = new BrowserProvider(gsnProvider as any)
```

## "signature mismatch" when using Metamask with local ganache


This is an old [bug](https://github.com/trufflesuite/ganache-core/issues/515) in Ganache.
It was fixed in v6.11, but requires an extra command-line argument:
```bash
ganache-cli --chainId 1337
```

Next major release should work without it.

(if you're curious: ganache's RPC call reports that the chain-id is 1337, but the OPCODE `chainid` returns 1 - unless we explicitly add the above command-line argument)

## Error: Provided Hub version(2.0) is not supported by the current interactor(2.2.2)

The **Paymaster** version is wrong: The client checks that it is compatible with all the contracts.
This happens when you create a custom paymaster with a version string that is less than 2.2.0


## I get an error: paymaster rejected in local view call to 'relayCall()' : Forwarder is not trusted

The paymaster doesn't support the forwarder that is used by your target contract.
make sure to set the `trustedForwarder` on both the paymaster an recipient contract, and use the same forwarder.


## I have multiple tests, each run ok, but running them together, I get "Unacceptable relayMaxNonce"

There is an issue with the way a relayer currently work and Truffle's `contract()` test construct.
Please use `describe()` or `context` instead (and avoid to snapshot/revert while a relayer is running)

An alternative is to start relayer in the `begin()` block of the contact.
Just note that you need to recreate the provider, and update the forwarder for your contracts.
e.g:

```js
contract('mytest', ()=> {
  before(async()=> {
    await GsnTestEnvironment.startGsn('localhost')
    const { forwaderAddress, paymasterAddress } = GsnTestEnvironment.loadDeployment()
    provider = RelayProvider.newProvider({provider: web3.currentProvider, config: { paymasterAddress}})
  })

  after(async()=> {
    await GsnTestEnvironment.stopGsn()
  })
}

```
  
}



## My client fails with an error: Can't resolve 'fs' 

Depending on your UI framework, you need to configure your webpack:

### Gatsby

Add this configuration file `gatsby-node.js` in the root of your project:
```js
exports.onCreateWebpackConfig = ({
  stage,
  rules,
  loaders,
  plugins,
  actions,
}) => {
  actions.setWebpackConfig({
        node: {
          fs: 'empty'
        }
  })
}
```

### Next.JS

Add the following configuration to `next.config.js` at the root directory of the project:

```js
module.exports = {
  future: { webpack5: true }, //needed only by earlier next.js versions (<= 10)
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.fallback.fs = false
    }
    return config
  },
}
```
