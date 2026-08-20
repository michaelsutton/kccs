# KCC2 reference code

This document illustrates KCC2 authority checks in current Silverscript and
Argent syntax. It is non-normative; KCC2 Sections 2 through 5 define the required
semantics.

## 1. Silverscript P2PKH

The following code verifies `p2pkh-schnorr/v1`:

```js
function requireP2PKHSchnorr(
    byte[32] authority,
    pubkey publicKey,
    sig signature
) {
    byte[32] publicKeyHash = blake3WithKey(
        byte[](publicKey),
        byte[32](
            byte[]("PublicKeyHash") +
            byte[19](0x00000000000000000000000000000000000000)
        )
    );
    require(publicKeyHash == authority);
    require(checkSig(signature, publicKey));
}
```

## 2. Silverscript P2SH

The following code verifies a P2SH authority using an input index supplied by
the higher-level convention:

```js
function requireP2SH(byte[32] authority, int authorityInput) {
    byte[] expected = byte[](new ScriptPubKeyP2SH(authority));
    require(tx.inputs[authorityInput].scriptPubKey == expected);
}
```

## 3. Argent covenant-ID authority

Argent is a language and transpiler for multi-contract, multi-application
protocols built on Silverscript. It compiles Argent programs to Silverscript.
Cross-covenant introspection and state-transition validation are part of its
core domain, making Argent a natural higher-level example of KCC2 authority
schemes.

Argent's `cov_id.co_spent()` performs the minimum covenant-ID approval check. It
lowers to a requirement that `OpCovInputCount` for the given Covenant ID is
greater than zero:

```js
require(cov_id(authority).co_spent());
```

When an application must validate how the authority covenant participates, an
Argent `observes` clause can authenticate its program template and inspect its
input and output state. For example:

```js
entry authorize()
observes authority by self.authority_id {
    inputs {
        before: Authority,
    }

    outputs {
        after: Authority,
    }
} {
    AuthorityState before = authority.inputs.before.state;
    require(before.enabled);

    require authority.outputs become {
        after <- Authority(before),
    };
}
```

Here the `observes` declaration requires the named authority input and output,
authenticates their `Authority` templates, exposes the decoded input state, and
validates the declared successor state.
