# Attestation Service

Attestation Service (AS for short) is a general function set that can verify TEE evidence.
With Confidential Containers, the attestation service must run in an secure environment, outside of the guest node.

With remote attestation, Attesters (e.g. the [Attestation Agent](https://github.com/confidential-containers/attestation-agent)) running on the guest node will request a resource (e.g. a container image decryption key) from the [Key Broker Service (KBS)](https://github.com/confidential-containers/kbs)).
The KBS receives the attestation evidence from the attestation agent and forwards it to the Attestation Service (AS). The AS role is to verify the attestation evidence and provide Attestation Results back to the KBS. Verifying the evidence is a two steps process:

1. Verify the evidence signature, and assess that it's been signed with a trusted key of TEE.
2. Verify that the TCB described by that evidence (including hardware status and software measurements) meets the guest owner expectations.

Those two steps are accomplished by respectively one of the [Verifier Drivers](#verifier-drivers) and the AS [Policy Engine](#policy-engine). The policy engine can be customized with different policy configurations.

The AS can be built as a library (i.e. a Rust crate) by other confidential computing resources providers, like for example the KBS.
It can also run as a standalone binary, in which case its API is exposed through a set of [gRPC](https://grpc.io/) endpoints.

# Components

## Library

The AS can be built and imported as a Rust crate into any project providing attestation services.

As the AS API is not yet fully stable, the AS crate needs to be imported from GitHub directly:

```toml
attestation-service = { git = "https://github.com/confidential-containers/attestation-service" branch = "main" }
```

## Server

This project provides the Attestation Service binary program that can be run as an independent server:

- [`grpc-as`](bin/grpc-as/): Provide AS APIs based on gRPC protocol.

## Tools

- [`grpc-as-ctl`](bin/tools/grpc-as-ctl/): A simple tool to configure the policy and reference value of `grpc-as`.

# Usage

Build and install AS components:

```shell
git clone https://github.com/confidential-containers/attestation-service
cd attestation-service
make && make install
```

`grpc-as` and `grpc-as-ctl` will be installed into `/usr/local/bin`.

# Architecture

The main architecture of the Attestation Service is shown in the figure below:
```
┌──────────────────────┐      Evidence        ┌─────────────────────────────┐
│                      ├──────────────────────>     Attestation Service     │
│Verification Demander │                      │                             │
│    (Such as KBS)     <──────────────────────┤┌──────────┐┌───────────────┐│
│                      │  Attestation Result  ││  Policy  ││Reference Value││
└──────────────────────┘                      ││  Engine  ││    Povider    ││
                                              │└──────────┘└───────────────┘│
                                              │┌───────────────────────────┐│
                                              ││     Verifier Drivers      ││
                                              │└───────────────────────────┘│
                                              └─────────────────────────────┘
```

### Evidence format:

The attestation evidence is included in a [KBS defined Attestation payload](https://github.com/confidential-containers/kbs/blob/main/docs/kbs_attestation_protocol.md#attestation):

```json
{
    "tee-pubkey": $pubkey,
    "tee-evidence": {}
}
```

- `tee-pubkey`: A JWK-formatted public key, generated by the KBC running in the HW-TEE.
For more details on the `tee-pubkey` format, see the [KBS protocol](https://github.com/confidential-containers/kbs/blob/main/docs/kbs_attestation_protocol.md#key-format).

- `tee-evidence`: The attestation evidence generated by the HW-TEE platform software and hardware in the AA's execution environment.
The tee-evidence formats depend on the TEE and are typically defined by the each TEE verifier driver of AS.

**Note**: Verification Demander needs to specify the TEE type and pass `nonce` to Attestation-Service together with Evidence,
Hash of `nonce` and `tee-pubkey` should be embedded in report/quote in `tee-evidence`, so they can be signed by HW-TEE.
This mechanism ensures the freshness of Evidence and the authenticity of `tee-pubkey`.

### Attestation result format:

```json
{
    "tee": $tee_type,
    "allow": true,
    "output": {
        "verifier_output": $verifier_output,
        "policy_engine_output": $opa_output
    },
    "tcb_status": $tcb_status_claims
}
```

* `allow`: The verification results. `true` if verification succeeded, `false` otherwise.
* `output`: When verification fails, `output.verifier` describes the failure reasons. When verification succeeds, `output.policy_engine` may provide additional policy verification information, depending on the policy configuration.
* `tcb_status`: Contains HW-TEE informations and software measurements of AA's execution environment.

## Verifier Drivers

A verifier driver parse the HW-TEE specific `tee-evidence` data from the received attestation evidence, and performs the following tasks:

1. Verify HW-TEE hardware signature of the TEE quote/report in `tee-evidence`.

2. Resolve `tee-evidence`, and organize the TCB status into JSON claims to return.

Supported Verifier Drivers:

- `sample`: A dummy TEE verifier driver which is used to test/demo the AS's functionalities.
- `amd-sev-snp`: TODO.
- `intel-tdx`: TODO.

## Policy Engine

The AS uses the [Open Policy Agent (OPA)](https://www.openpolicyagent.org/docs/latest/) framework as its policy engine to verify the Attester TCB status.
OPA is a very flexible and powerful policy engine, AS allows users to define and upload their own policy when performing evidence verification.
If the user does not need to customize his own policy, AS will use the [default policy](default_policy.rego).

**Note**: Please refer to the [Policy Language](https://www.openpolicyagent.org/docs/latest/policy-language/) documentation for more information about the `.rego`.

## Reference Value Provider

In order to verify TCB status, AS needs to obtain the reference value for comparison in advance.
AS allows users to upload customized reference value sets when they need to perform evidence validation (just as they can upload customized policy).
If no customized reference value is uploaded, the AS will obtain the reference value from [Reference Value Provider Service](rvps/README.md) (RVPS for short).