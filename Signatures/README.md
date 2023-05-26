# Signing $ Verofying Message Process

## Requirements for EIP712

According to EIP712, for signing each message, you need to provide a Domain Separator specific to the dApp to ensure its uniqueness compared to similar contracts. This involves defining a name and version for the contract, as well as the deployed chainId and the address of the verifying contract. You can also provide a salt for added uniqueness. EIP712 explains:

> It is possible that two DApps come up with an identical structure like Transfer(address from,address to,uint256 amount) that should not be compatible. By introducing a domain separator the dApp developers are guaranteed that there can be no signature collision.

The following parameters are optional but enhance the security of your contract against replay attacks.

**name**: the dApp or protocol name, e.g. “Polytrade”

**version**: The current version of what the standard calls a “signing domain”. This can be the version number of your dApp or platform. It prevents signatures from one dApp version from working with those of others.

**chainId**: The [EIP-155](https://eips.ethereum.org/EIPS/eip-155) chain id. Prevents a signature meant for one network, such as a testnet, from working on another, such as the mainnet.The `block.chainid` keyword in Solidity returns the current chain id.

**verifyingContract**: The Ethereum address of the contract that will verify the resulting signature. The `address(this)` keyword in Solidity returns the contract’s own address, which it can use when verifying the signature.

Message Type for domain separator is:

    const domainType = {
        "EIP712Domain": [
            {
              "name": "name",
              "type": "string"
            },
            {
              "name": "version",
              "type": "string"
            },
            {
              "name": "chainId",
              "type": "uint256"
            },
            {
              "name":"verifyingContract",
              "type": "address"
            }
        ]
    }

After defining the TypedMessage, we will send the values in the same structure:

    const domainData = {
        name: "Polytrade",
        version: "1.0",
        chainId: chainId,
        verifyingContract:
        verifyingContractAddress
    };

EIP2612, which is the Permit extension for ERC20 tokens, allows approval by signing a message following the EIP712 standard. Therefore, it includes the domain separator along with a Permit structure:

    const permitType = {
      "Permit": [{
          "name": "owner",
          "type": "address"
        },
        {
          "name": "spender",
          "type": "address"
        },
        {
          "name": "value",
          "type": "uint256"
        },
        {
          "name": "nonce",
          "type": "uint256"
        },
        {
          "name": "deadline",
          "type": "uint256"
        }
      ]
    }

Similar to the Domain Separator, after defining the TypedMessage, we have the values for it:

    const permitData: {
        "owner": owner,
        "spender": spender,
        "value": value,
        "nonce": nonce,
        "deadline": deadline
    }

Now that we have all the necessary components, we can proceed with signing the prepared message and then verify if the owner's address is the signer of the message.

## 1. Signing Off-chain

### a. _Ethers_ `_signTypedData`

As explained in the [docs](https://docs.ethers.org/v5/api/signer/#Signer-signTypedData) you can use this method to sign a message using the EIP-712 specification and retrieve the signature.

> `signer._signTypedData( domain , types , value )`

    signature = await signer._signTypedData(
        domainData,
        permitType,
        permitData
    );

- Note that you only need to send the domain values and not the domain type, as the function sets it by default.

### b. _Web3_ `eth_signTypedData_v4`

According to the [Docs](https://docs.metamask.io/wallet/how-to/sign-data/#use-eth_signtypeddata_v4), when sending the data with the signer, you should use a V3 or V4 method.

> - **V1** is based upon an early version of EIP-712 that lacked some later security improvements, and should generally be neglected in favor of later versions.
> - **V3** is based on EIP-712, except that arrays and recursive data structures are not supported.
> - **V4** is based on EIP-712, and includes full support of arrays and recursive data structures.

    const data = JSON.stringify({
        types: {
            EIP712Domain: domainType,
            Permit: permitType,
        },
        domain: domainData,
        primaryType: "Permit",
        message: permitData
    });

    web3.currentProvider.send(
      {
        method: "eth_signTypedData_v4",
        params: [signer, data],
        from: signer
      },
      function(err, result) {
        if (err) {
          return console.error(err);
        }
      }
    )

### c. Metamask _eth-sig-util_ `signTypedData`

For the mentioned [function](https://metamask.github.io/eth-sig-util/latest/functions/signTypedData.html#signTypedData), you can specify the version and provide the private key of the signer along with the data.

> `signTypedData(privateKey, data, version)`

    ethSigUtil.signTypedData(privateKey, {
        data: {
          types: {
              EIP712Domain:domainTye,
              Permit: permitType
          },
          domain: domainData,
          primaryType: 'Permit',
          message: permitData
        },
        "V4"
    });

- Please note that the formatting of the message is highly sensitive, and it does not provide any errors if the types do not match, which can lead to an invalid signature.

### Split Signature

It is possible to send the signature along with the parameters to a smart contract for message verification. However, for EIP2612, you need to send the `r`, `s`, and `v` parameters of the signature to the permit function. Therefore, we need to split the hash to extract these parameters.

`splitSignature()` by ethers can do the job:

    const { r, s, v } = splitSignature(signature);

Alternatively, you can split the signature into its components as follows:

    const r = signature.slice(0, 66);
    const s = "0x" + signature.slice(66, 130);
    const v = parseInt(signature.slice(130, 132), 16);

## 2. Verifying on-chain

Signing on-chain with Solidity is straightforward. First, we need to encode the EIP712 domain type, and then we can obtain the keccak hash of it using the following steps:

    bytes32 internal constant _EIP_712_DOMAIN_TYPEHASH =
        keccak256(
            abi.encodePacked(
                "EIP712Domain(",
                "string name,",
                "string version,",
                "uint256 chainId,",
                "address verifyingContract",
                ")"
            )
        );

Similarly, for the `Permit` typehash, we can follow the same process of encoding and obtaining the keccak hash:

    bytes32 internal constant _PERMIT_TYPEHASH =
        keccak256(
            abi.encodePacked(
                "Permit(",
                "address owner,",
                "address spender,",
                "uint256 value,",
                "uint256 nonce,",
                "uint256 deadline",
                ")"
            )
        );

To accommodate the domain separator values when passing the **name** and **version**, and to dynamically recalculate the `_DOMAIN_SEPARATOR` in case of changes to the contract address or chain ID, we avoid hardcoding them. To calculate the domain separator, we first obtain the hash of the **name** and **version**. Subsequently, we hash the domain type along with all the values combined.

    _NAME_HASH = keccak256(bytes(name));
    _VERSION_HASH = keccak256(bytes(version));
    _DOMAIN_SEPARATOR = keccak256(
        abi.encode(
            _EIP_712_DOMAIN_TYPEHASH,
            _NAME_HASH,
            _VERSION_HASH,
            block.chainid,
            address(this)
        )
    );

For the `PERMIT`, we would follow the same process

    bytes32 PERMIT = keccak256(
        abi.encode(
            _PERMIT_TYPEHASH,
            owner,
            spender,
            value,
            nonce,
            deadline
        )
    );

To obtain the EIP712 digest for public address recovery, we concatenate the hashes of both `_DOMAIN_SEPARATOR` and `PERMIT` with the prefix `\x19\x01`. Then, we hash the resulting concatenation to obtain the digest required for address recovery. The prefix `0x19` is based on EIP191, which is used for standardized handling of signed data, and `01` signifies EIP712 structured data.

> `{0x1901}{_DOMAIN_SEPARATOR}{PERMIT}`

This assembly code ensures safe execution by first obtaining the memory pointer. It then writes the initial 2-byte prefix, followed by `_DOMAIN_SEPARATOR` and `PERMIT`. Finally, it computes the hash of the concatenated result and returns the resulting digest:

        assembly {
            let ptr := mload(0x40)
            mstore(ptr, "\x19\x01")
            mstore(add(ptr, 0x02), _DOMAIN_SEPARATOR)
            mstore(add(ptr, 0x22), PERMIT)
            digest := keccak256(ptr, 0x42)
        }

Here's the Solidity version of the code that performs the same operations:

    digest = keccak256(abi.encodePacked(uint16(0x1901), _DOMAIN_SEPARATOR, PERMIT));

Now that everything is prepared, we can obtain the public address by using the `ecrecover(digest, v, r, s)` function.

    ecrecover(digest, v, r, s);

By asserting the equivalence of the signer and owner addresses, we can ensure that they are identical, thereby enabling us to augment the spender's allowance.

- In the event that a signature has been sent, we can obtain the `v`, `r` and `s` parameters as follows:

  > If the signature length is 64 bytes (Compact Signature), we can adhere to the guidelines specified in EIP2098 to extract the corresponding parameters.

        bytes32 r;
        bytes32 s;
        uint8 v;
        if (signature.length == 64) {
            bytes32 vs;
            (r, vs) = abi.decode(signature, (bytes32, bytes32));
            s = vs & (0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff);
            v = uint8(uint256(vs >> 255)) + 27;
        } else if (signature.length == 65) {
            (r, s) = abi.decode(signature, (bytes32, bytes32));
            v = uint8(signature[64]);
        }

* Please take note that for off-chain verification purposes, we can utilize the same steps to generate domain and permit hashes (either manually or by utilizing the `TypedDataUtils.hashStruct` helper from _eth-sig-util_ library). Subsequently, we can create a digest from these hashes. Finally, we can use the `recoverAddress(digest, signature)` function from the _ethers_ library or the `recoverTypedSignature(data, signature, version)` function from the _eth-sig-util_ library to retrieve the signer's address.

* In cases where the message structure includes arrays or recursive structs, it is necessary to define their respective types, ensuring that they are declared in the correct order.

        messageType = {
            FirstStruct: [
                { name: "owner", type: "address" },
                { name: "customer", type: "Customer[]" },
                { name: "seller", type: "Seller" },
                { name: "price", type: "uint256" },
            ],
            Customer: [
                { name: "address", type: "address" },
                { name: "discount", type: "uint256" },
            ],
            Seller: [
                { name: "address", type: "address" },
                { name: "price", type: "uint256" },
            ],
        };
