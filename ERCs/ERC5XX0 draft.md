### eip: 5???
### title: Client Script URI for Token Contracts
### description: Add a scriptURI to point to point to an executable script associated with the functionality of the token.
### author: James (@JamesSmartCell), Weiwu (@weiwu-zhang)
### discussions-to:
### status: Draft
### type: Standards Track
### category: ERC
### created: 2022-05-03
### requires: 
### Abstract
This ERC is a contract interface that adds a `scriptURI()` function for locating executable scripts associated with the token.

### Motivation

Often, Smart Contract authors want to provide some user functionality to their tokens through client scripts. The idea is made popular with function-rich NFTs.

It's important that a token's client script is linked with the token's contract, since the client script carry out trusted tasks such as creating transactions for the user.

Users can be sure they are using the correct script through the contract author packaging a URI to an official script, made available with a call to the token contract itself.

This ERC proposes adding a `scriptURI`, a structure containing URIs to download the client script. It can be a link to IPFS, GitHub, a cloud provider, etc.

Each token contract has one `scriptURI`  function to return the download URI to a client script.

Concretely each element in the array contains a URI to download the script itself.

The script provides a client-side executable to the hosting token. Examples of such a script could be:

- A 'miniDapp', which is a cut-down dapp tailored for a single token.
- A 'TokenScript' which provides [T.I.P.S.](https://tokenscript.org/TIPS.html) from a browser wallet.
- An extension that is downloadable to the hardware wallet with an extension framework, such as Ledger.

#### Script location

While the most straightforward solution to facilitate specific script usage associated with NFTs, is clearly to store such a script on the smart contract. However, this has several disadvantages: 

1. The smart contract signing key is needed to make updates, causing the key to become more exposed, as it is used more often. 

2. Updates require smart contract interaction. If frequent updates are needed, smart contract calls can become an expensive hurdle.

3. Storage fee. If the script is large, updates to the script will be costly. A client script is typically much larger than a smart contract.

For these reasons, storing volatile data, such as token enhancing functionality, on an external resource makes sense. Such an external resource can be either be  hosted centrally, such as through a cloud provider, or privately hosted through a private server, or decentralized hosted, such as the interplanetary filesystem.

While centralized storage for a decentralized functionality goes against the ethos of web3, fully decentralized solutions may come with speed, price or space penalties. This ERC handles this by allowing the function `ScriptURI` to return multiple URIs, which could be a mix of centralized, individually hosted and decentralized locations.

While this ERC does not dictate the format of the stored script, the script itself could contain pointers to multiple other scripts and data sources, allowing for advanced ways to expand token scripts, such as lazy loading. 
The handling of the integrity of such secondary data sources is left dependent on the format of the script. For example, HTML format uses [the `integrity` property](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity), while [signed XML format has `<Manifest/>`](https://www.w3.org/TR/xmldsig-core2/#sec-Manifest).

#### Overview

With the discussion above in mind, we outline the solution proposed by this ERC. For this purpose, we consider the following variables:

- `SCPrivKey`: The private signing key to administrate a smart contract implementing this ERC. Note that this doesn't have to be a new key especially added for this ERC. Most smart contracts made today already have an administration key to manage the tokens issued. It can be used to update the `scriptURI`.

- `newScriptURI`: an array of URIs for different ways to find the client script.

We can describe the life cycle of the `scriptURI` functionality:

- Issuance

1. The token issuer issues the tokens and a smart contract implementing this ERC, with the admin key for the smart contract being `SCPrivKey`.
2. The token issuer calls `setScriptURI` with the `scriptURI`.

- Update `scriptURI`

1. The token issuer stores the desired `script` at all the new URI locations and constructs a new `scriptURI` structure based on this. 
2. The token issuer calls `setScriptURI` with the new `scriptURI` structure.

### Specification

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY” and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

We define a scriptURI element using the `string[]`.
Based on this, we define the smart contract interface below:
```
interface IERC5XX0 {
    /// @dev This event emits when the scriptURI is updated, 
    /// so wallets implementing this interface can update a cached script
    event ScriptUpdate(string[] memory newScriptURI);

    /// @notice Get the scriptURI for the contract
    /// @return The scriptURI
    function scriptURI() external view returns(string[] memory);

    /// @notice Update the scriptURI 
    /// emits event ScriptUpdate(scriptURI memory newScriptURI);
    function setScriptURI(string[] memory newScriptURI) external;
}
```
The interface MUST be implemented under the following constraints:

- The smart contract implementing `IERC5XX0` MUST store variables `address owner` in its state.

- The smart contract implementing `IERC5XX0` MUST set `owner=msg.sender` in its constructor.

- The `ScriptUpdate(...)` event MUST be emitted when the ```setScriptURI``` function updates the `scriptURI`.

- The `setScriptURI(...)` function MUST validate that `owner == msg.sender` *before* executing its logic and updating any state.

- The `setScriptURI(...)` function MUST update its internal state such that `currentScriptURI = newScriptURI`.

- The `scriptURI()` function MUST return the `currentScriptURI` state.

- The `scriptURI()` function MAY be implemented as pure or view.

- Any user of the script learned from `scriptURI` MUST validate the script is either at an immutable location, its URI contains its hash digest, or it implements ERC 5XX1, which asserts authenticity using signatures instead of a digest.

### Rationale

This method avoids the need for building secure and certified centralized hosting and allows scripts to be hosted anywhere: IPFS, GitHub or cloud storage.

### Backwards Compatibility

This standard is compatible with all Token standards (ERC20, 721, 777, 1155 etc.)

### Examples

We here go through a couple of examples of where an authenticated script is relevant for adding additional functionality for tokens. 

1. A Utility NFT is an event ticket and the authenticated script is a JavaScript 'minidapp' which asks the user to sign a challenge message that shows ownership of the key controlling the ticket. The dapp would then render the signature as a QR code which can be scanned by a ticketing app, which could then mark the ticket as used.

2. Smart Token Labs uses a framework called TokenScript; one element of which is a user interface description for contract interaction through tokens.
Consider a simple 'mint' verb associated with an already existing NFT. The associated script can for example allow the owner to mint a derivative through a contract already holding enough ether for the minting fee, without needing to connect their wallet.

3. An NFT Script which controls a Smartlock. For example consider the lock  being linked to a digital NFT twin and being controlled with the verbs "lock" and "unlock", each of which has an associated JavaScript. Each of these scripts could be executed after the user signs a challenge in a web-view. This is an off-chain example that uses on-chain assets for functionality.

### Tests
#### Test Contract
```
import "@openzeppelin/contracts/access/Ownable.sol";
import "./IERC5XX0.sol";
contract ERC5XX0 is IERC5XX0, Ownable {
    string[] private _scriptURI;
    function scriptURI() external view override returns(string[] memory) {
        return _scriptURI;
    }

    function setScriptURI(string[] memory newScriptURI) external onlyOwner override {
        _scriptURI = newScriptURI;

        emit ScriptUpdate(newScriptURI);
    }
}
```

#### Test case
```
const { expect } = require('chai');
const { BigNumber, Wallet } = require('ethers');
const { ethers, network, getChainId } = require('hardhat');

describe('ERC5XX0', function () {
  before(async function () {
    this.ERC5XX0 = await ethers.getContractFactory('ERC5XX0');
  });

  beforeEach(async function () {
    // targetNFT
    this.erc5XX0 = await this.ERC5XX0.deploy();
  });

  it('Should set script URI', async function () {
    const scriptURI = [
      'uri1', 'uri2', 'uri3'
    ];

    await expect(this.erc5XX0.setScriptURI(scriptURI))
      .emit(this.erc5XX0, 'ScriptUpdate')
      .withArgs(scriptURI);
    
    const currentScriptURI = await this.erc5XX0.scriptURI();

    expect(currentScriptURI.toString()).to.be.equal(scriptURI.toString());
  });
```

### Security Considerations

**When a server is involved**

When the client script does not purely rely on connection to a blockchain node, but also calls server APIs,  the trustworthiness of the server API is called into question. This ERC doesn't provide the mechanism to assert the authenticity of the API access point. Instead, as long as the client script is trusted, it's assumed that it can call any server API in order to carry out token functions. This means the client script can mistrust a server API access point.

**When the scriptURI doesn't contain integrity (hash) information**

We separately authored ERC5XX1 to guide on how to use digital signatures to efficiently and concisely to ensure authenticity and integrity of scripts not stored at an URI which is a digest of the script itself. 
