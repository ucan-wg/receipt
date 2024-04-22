# UCAN Invocation Receipt Specification v1.0.0

## Editors

- [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
- [Irakli Gozalishvili](https://github.com/Gozala), [Protocol Labs]

## Authors

- [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
- [Irakli Gozalishvili](https://github.com/Gozala), [Protocol Labs]

## Depends On

- [DAG-CBOR]
- [UCAN]
- [UCAN-IPLD]
- [Varsig]

# Abstract

UCAN Invocation defines a format for expressing the intention to execute delegated UCAN capabilities. UCAN Invocation `Receipt` is a signed assertion of the [Executor] state describing [Result] and [Effect]s of the invocation.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction


A `Receipt` is a Invocation of the `/ucan/assert` capability. It represents signed assertion from the [Executor] state describing [Result] and [Effect]s of some task invocation. Receipt is a signed commitment by [Executor] to a state, described by it, within the timeframe of the `Receipt`.

# High-Level Concepts

## Roles

### Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

## Components

### Task

A [Task] is like a deferred function application: a request to perform some action on a resource with specific input.

### Result

A [Result] is the output of a [Task].

### Effect

An [Effect] is a [Task] caused by the invoked [Task].

## IPLD Schema

```ipldsch
type UCAN union {
  | Invocation "ucan/i/1.0.0-rc.1"
  | Receipt    "ucan/o/1.0.0-rc.1"
} representation keyed

type Receipt struct {
  p SignaturePayload
  s Signature
}

type SignaturePayload struct {
  h VarsigHeader
  p ReceiptPayload
}

type ReceiptPayload union {
  | AssertPayload "/ucan/assert"
} representation inline {
  discriminantKey "cmd"
}

type AssertPayload struct {
  uiv SemVer

  iss DID
  sub DID
  aud DID

  args Assertion
  nonce String

  meta Meta
  prf [&Delegation]
  exp Expiry
}

type Assertion struct {
  about &Task
  facts InvocationFacts
}

type InvocationFacts struct {
  # Output of the invocation
  out     Result

  # Effects to be performed
  run     Effect[]
}

type Result union {
  | any    "ok"    # Success
  | any    "error" # Error
} representation keyed

# Represents a task caused by the invocation
type Effect struct {
  cmd Command
  sub  DID
  args {String : Any}
  nonce String

  # Optionally effect can specify other invocation details
  iss optional DID
  aud optional DID

  meta optional Meta
  prf optional [&Delegation]
  exp optional Integer
}

type Command String
type DID String
type Meta { String: Any }

type Expiry union {
  | Timestapm int
  | Never     null
} representation kinded

type Timestamp int
type Never null
```

### `ReceiptPayload`

The `ReceiptPayload` MUST be a `/ucan/assert` Invocation.

#### Receipt Issuer

The `iss` field of the `ReceiptPayload` MUST be DID of the Invocation [Executor] or a delegate authorized by it. If `iss` is not the Invocation [Executor], the `prf` field of the `AssertPayload` MUST provide a valid delegation chain from [Executor] to the `iss` DID authorizing it to issue this receipt.

#### Receipt Subject

The `sub` field of the `ReceiptPayload` MUST be the DID of the Invocation [Executor].

#### Receipt Audience

The `aud` field of the `ReceiptPayload` MUST be the DID of the Invoaction [Executor] and be equal to `sub` field. This creates an authorization loop that is rooted and terminated with an [Executor], which in turn can be used by anyone to prove that described state has been asserted by the [Executor].

#### Receipt Expiry

The `exp` field of the `ReceiptPayload` denotes time until which [Executor] is commited to uphold issued assertion. The `exp` field MUST be set to `null` for assertions that are permanent.

Receipts for the pure computation SHOULD set `exp` field to `null`. On the other hand when result of the task is imprmanent, the `exp` field MUST be set to the time reflecting it TTL.

### `Assertion`

The `args` field of the `ReceiptPayload` MUST be set to the value conforming to the `Assertion` schema.

#### Asserted Task

The `about` field of the `Assertion` MUST be set to the executed Task this receipt is for.

#### Asserted Facts

The `facts` field of the `Assertion` MUST be set to value that conforms `InvocationFacts` schema.

#### Asserted Result

The `facts.out` field of the `Assertion` MUST be set to the result of the invocation and conform to the `Result` schema.

##### Result Variants

###### Success

The success branch MUST contain the value returned from a successful [Task] wrapped in the `"ok"` tag. The exact shape of the returned data is left undefined to allow for flexibility in various Task types.

```json
{ "ok": 42 }
```

###### Failure

The failure branch MAY contain detail about why execution failed wrapped in the "error" tag. It is left undefined in this specification to allow for [Task] types to standardize the data that makes sense in their contexts.

#### Asserted Effects

The `facts.run` field of the `Assertion` MUST be set to the list of values conforming to `Effect` schema. It describes set of Tasks that are caused by the Invocation. 

Pure computational tasks MUST set `facts.run` to an empty list. Effectful tasks MUST populate `facts.run` with set of requested tasks to be performed.

Please note that issuing receipt cedes control to the runtime which can decide when and whether to run requested tasks.

### Proofs

The `prf` field defines all [Delegation]s required to prove that this Invocation has an unbroken authorization chain.

### Metadata

The OPTIONAL `meta` field MAY be omitted or used to contain additional data about the receipt. This field MAY be used for tags, commentary, trace information, and so on.


[dag-json]: https://ipld.io/docs/codecs/known/dag-json/
[varsig]: https://github.com/ChainAgnostic/varsig/
[ucan-ipld]: https://github.com/ucan-wg/ucan-ipld/
[ucan]: https://github.com/ucan-wg/spec/
[dag-cbor]: https://ipld.io/specs/codecs/dag-cbor/spec/
[ipld representation]: https://ipld.io/docs/schemas/features/representation-strategies/
[lazy-vs-eager]: #112-lazy-vs-eager-evaluation
[invoker]: #211-invoker
[executor]: #212-executor
[task]: #3-task
[authorization]: #4-authorization
[invocation]: #5-invocation
[result]: #6-result
[effect]: #7-effect
[receipt]: #8-receipt
[pipelines]: #9-pipelines
[await]: #await
[Protocol Labs]:https://protocol.ai/
