---
title: " WebAssembly Framework for Network Management in Heterogeneous Edge Networks"
abbrev: "Intent-Driven WebAssembly Framework"
category: info

docname: draft-chen-opsawg-intent-webassembly-framework-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Operations and Management Area Working Group"
keyword:
 - intent
 - webassembly
 - edge
venue:
  group: "Operations and Management Area Working Group"
  type: "Working Group"
  mail: "opsawg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/opsawg/"
  github: "billwuqin/webassembly-for-the-edge"
  latest: "https://billwuqin.github.io/webassembly-for-the-edge/draft-chen-opsawg-intent-webassembly-framework.html"

author:
 -
    fullname: Zonghua Cheng
    organization: China Telecom
    email: "abc@chinatelecom.cn"
 -
    fullname: Jianping Sun
    organization: China Telecom
    email: "ab@chinatelecom.cn"
 -
    fullname: Yuanyuan Yang
    organization: Huawei
    email: "yangyuanyuan55@huawei.com"

normative:
  RFC2119:
  RFC8174:

informative:
  RFC7252:
  RFC7959:
  RFC9019:
  RFC9052:
  RFC9110:
  RFC9114:
  RFC9124:
  I-D.ietf-opsawg-rfc5706bis:

  WASM:
    title: "WebAssembly Core Specification"
    target: "https://webassembly.github.io/spec/core/"
    author:
      - org: "WebAssembly Community Group"

  WIT:
    title: "The WIT text format"
    target: "https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md"
    author:
      - org: "WebAssembly Community Group"

--- abstract

   This document describes an architecture for executing WebAssembly (WASM) modules for network management on heterogeneous edge devices. It describes dynamic bytecode dispatching, the Universal Bytecode Interface (UBI) for abstracting heterogeneous network element capabilities, and metadata for capability exposure, intent-driven discovery, and security audits. The framework decouples management scripts from platform-specific interfaces. It reuses existing mechanisms for software update, payload manifests, integrity protection, and transport where applicable.

--- middle

# Introduction

## Background and Problem statement

   As automated network management evolves, network operators increasingly rely on dynamic, closed-loop control mechanisms. Structured tool interfaces can expose network node capabilities for automated orchestration. Large language models can generate execution logic for management tasks. However, executing dynamic operational scripts or telemetry-collection logic directly on heterogeneous and constrained network devices poses security risks and resource challenges.
Container-based execution may introduce startup and memory overhead that is unsuitable for some constrained nodes. WebAssembly {{?WASM}} provides an alternative for lightweight management tasks, subject to the capabilities and limits of the host runtime.


   

## Use Cases

   WebAssembly-based sandboxing, Universal Bytecode Interfaces, and automated orchestration support the following network management use cases:
   a) Telemetry Collection: Distributing lightweight bytecode to edge nodes to collect and process local telemetry, returning the results needed for network management.
   b) Policy Adjustment: Running short-lived tasks on edge nodes or compute boards for traffic inspection and service protection. Each sandbox executes a task with its own permissions and resource limits. Concurrent tasks support different user or service policies, subject to the node’s available capacity. Packet inspection uses authorized host capabilities.
   c) Device Management: Executing device management operations through authorized host functions. Tasks can access and process local resources without uploading all raw data, supporting deployments with data-transfer or privacy constraints.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

   * WASM Sandbox: A lightweight, isolated virtual machine environment
     running inside a host process, executing WebAssembly bytecode with
     linear memory isolation.
   * UBI (Universal Bytecode Interface): An abstraction layer of host functions injected into the WASM runtime, shielding management bytecode from hardware platform-specific details.
   * Node Profile: A structured metadata manifest that declares a network
     element's exposed capabilities, supported operations, and required
     host function privileges. Capability advertisement does not itself authorize task execution.
   * UBI-DP (Universal Bytecode Distribution Protocol): The encapsulation
     and messaging mechanism used to dispatch compiled WASM bytecode
     binaries along with metadata across network management channels.
   * WIT (WebAssembly Interface Types): The interface definition language
  used by the WebAssembly Component Model to define types and functions
  crossing component boundaries {{?WIT}}.

#  Architectural Framework Overview

   The proposed framework consists of three logical planes. They separate task orchestration, policy coordination, and execution on edge devices:

~~~~
   +--------------------------------------------------------+
   |        Intent & Orchestration Plane (Management)       |
   | [Intent Parsing] -> [Workflow Compile] -> [Node Profile]
   +--------------------------------------------------------+
                                |
                                v (Task and constraints)
   +--------------------------------------------------------+
   |        Coordination & Policy Plane (Control Layer)     |
   | [Static Audit Matrix] -> [CaaS Pool] -> [Hash and Sign]|
   +--------------------------------------------------------+
                                |
                                v(UBI-DP Dispatching)
   +----------------------------------------------------------------------+
   |             Embodied Execution Plane (Device Layer)                  |
   | [WASM Sandbox] -> [UBI Host Functions] -> [Platform Implementations] |
   +----------------------------------------------------------------------+
~~~~

 {:#arch title="Intent based Framework" artwork-align="center"}

   1. Intent & Orchestration Plane: Translates network operator intents into structured workflow models and maps required management tools. It identifies target nodes using Node Profiles and passes task constraints to the Coordination & Policy Plane.
   2.  Coordination & Policy Plane: Acts as the central management controller. It audits requested tasks against security policies, compiles modules via Compilation-as-a-Service (CaaS), and issues cryptographically signed bytecode envelopes. The signed metadata binds the module digest to its approved host functions and resource limits.
   3.  Embodied Execution Plane: The physical edge node running embedded WASM runtimes. It validates the module and its associated policy, enforces resource limits, and restricts device access through authorized UBI host functions. A warm process pool retains initialized runtime processes to reduce sandbox setup time; task state and permissions are reset before reuse.
  Host functions expose the sandbox host’s basic operations, such as reading a counter or updating a permitted policy. The UBI defines the interface visible to the module; platform-specific implementations access local APIs, drivers, or management services. Each invocation is checked against the task’s permissions.

# Core Operational Mechanisms

## Dynamic Resource and Capability Discovery Mechanism

The discovery mechanism maps task requirements to available node capabilities:

   1. Capability Advertisement: An edge node registers or updates its Node Profile using an existing management channel, such as CoAP {{?RFC7252}} or HTTP/3 {{?RFC9114}}.

   2. Semantic Feature Graph: The controller maintains a capability catalog, optionally represented as a graph, including supported operations and resource budgets.

   3. Intent-Driven Matching Filter: The controller selects nodes supporting the required UBI primitives and resource limits.

## Bytecode Dispatching Lifecycle

   Bytecode distribution MUST be atomic, secure, and self-contained. The edge node MUST NOT execute an incompletely received or unverified module. The network controller utilizes the UBI-DP format to encapsulate the
   binary and its metadata payload including the module digest, task permissions, and resource limits, through the following lifecycle:
   Step 1: Discovery. Edge Node registers its profile, declaring local
     UBI primitives.
   Step 2: Audit. Controller compiles the code and checks the resulting module’s imports against supported UBI primitives and task permissions via the Static Symbol Audit Matrix.
   Step 3: Transport. Controller computes the module’s SHA-256 digest, signs the metadata containing this digest, task permissions, and resource limits, and transmits the module and signed metadata via HTTP {{?RFC9110}} or CoAP {{?RFC7252}}.
   Step 4: Execution. Edge Node validates signatures and the module digest, checks local policy, provisions execution threads, isolates environments, and executes the authorized module with its approved permissions and resource limits.
   
## WASM Bytecode Semantic Properties and Core Capabilities

   WASM bytecode uses features supported by the target runtime and executes within the following limits:

   a) Thread Restrictions: The runtime restricts thread creation and execution time according to node policy.
   b) Memory Limits: The runtime enforces a maximum linear-memory size for each module.
   c) AOT Support: Ahead-of-time compilation may reduce runtime compilation overhead where supported.
   
## Universal Bytecode Interface (UBI) Specifications

## Telemetry and Sensing Domain

~~~~
   interface telemetry {
       record metric-data {
           metric-name: string,
           timestamp: u64,
           payload: list<u8>
       }
       read-metric: func(name: string) -> result<metric-data, string>
   }
~~~~

## Actuation and Control Domain

~~~~
   interface actuation {
       enum pin-value { low, high }
       gpio-write: func(pin: u32, value: pin-value) -> result<_, string>
       i2c-transfer: func(
           bus-id: u32,
           addr: u32,
           tx-data: list<u8>,
           rx-len: u32
       ) -> result<list<u8>, string>
   }
~~~~

## Media Processing Domain

~~~~
   interface media {
       enum resolution-mode {
           res-640x480,
           res-1280x720,
           res-1920x1080
       }
       audio-start: func(mode: u32, sample-rate: u32) -> result<_, string>
       camera-capture: func(res: resolution-mode) -> result<list<u8>, string>
   }
~~~~

# Metadata Definitions and Schemas

## Capability Profiling Schema (Node Profile)

The Node Profile describes runtime features, supported UBI interfaces, and resource limits. The detailed schema is to be defined.

## SUIT Manifest and Bytecode Distribution Envelope

   To securely deliver WebAssembly (WASM) bytecode components to constrained edge nodes, the distribution framework adopts the Software Updates for Internet of Things (SUIT) architecture {{?RFC9019}} and manifest information model {{?RFC9124}}. The management controller describes the compiled WASM binary in a SUIT manifest, protected via CBOR Object Signing and Encryption (COSE) {{?RFC9052}}. The mapping of WASM execution policy parameters to SUIT metadata remains to be defined.

   The framework uses the following workflow before executing the WASM module:

   1. Manifest Authentication: The recipient edge node validates the
      COSE signature of the SUIT manifest using pre-configured trust
      anchors.
   2. Applicability Verification: The device checks whether the manifest's target identifiers, including vendor IDs and class IDs, match its local device profile.
   3. Dependency Resolution: The node verifies that the UBI interfaces required by the module are available.
   4. Payload Fetching: The standalone core .wasm binary payload is retrieved and its digest is verified against the authenticated manifest. CoAP block-wise transfer {{?RFC7959}} may be used with appropriate transport or object security.
   5. Installation & Sandbox Initialization: The verified WASM module is loaded into the isolated runtime environment with approved memory page constraints and authorized UBI host symbols. The execution policy is authenticated, bound to the module digest, and checked against local authorization policy before these permissions are granted.

   The following JSON conceptually illustrates the payload metadata and WASM execution parameters. It is not a valid SUIT manifest encoding. The runtime parameters are illustrative extensions, and the digest is a placeholder.

~~~~
   {
     "suit-manifest-version": 1,
     "suit-sequence-number": 42,
     "suit-common": {
       "suit-components": [ "wasm-telemetry-module" ]
     },
     "suit-install": [
       {
         "suit-condition-vendor-id": "example-vendor-id",
         "suit-condition-class-id": "example-edge-class-id"
       },
       {
         "suit-directive-fetch": {
           "uri": "coap://management-server.local/wasm/module.wasm",
           "digest": {
             "algorithm": "SHA-256",
             "value":
             "<SHA-256 digest of the WASM payload>"
           }
         }
       }
     ],
     "suit-execute": [
       {
         "runtime-parameters": {
           "allowed_memory_pages_max": 256,
           "execution_timeout_ms": 10000,
           "allowed_host_symbols": [
             "telemetry.read-metric",
             "actuation.gpio-write"
           ]
         }
       }
     ]
   }
~~~~

# Security Considerations

   This section describes controls for executing bytecode within managed network environments.

## Static Symbol Audit Matrix and Link-Time Verification

   To prevent malicious or corrupted management code from executing unauthorized operations, the Management Controller MUST execute a static symbol audit before bytecode encapsulation. The node MUST also check module imports against its available interfaces and the task's approved permissions before instantiation.

   The audit pipeline enforces the following validation steps for WebAssembly core modules:

   1. Import Table Extraction: The audit engine parses the WASM import section to identify imported functions and other imported entities, including their names and types.
   2. Capability-Matrix Intersection: The extracted import symbols are cross-checked against the node's supported capabilities and the permissions approved for the task.
   3. Link-Time Rejection: Unresolved, type-incompatible, or unauthorized imports SHALL result in rejection of the module before instantiation. Modules that fail the controller's audit SHALL NOT be distributed. Import checks alone do not establish functional correctness or enforce restrictions on runtime argument values.

## Sandbox Boundary Enforcement

   The execution of dynamically provisioned bytecode on remote physical devices requires isolation and access controls. The framework applies the following controls:

   1. Linear Memory Boundary Isolation: Embedded runtimes MUST enforce WebAssembly memory bounds and per-task resource limits. Host functions MUST validate arguments and access permissions before accessing host resources.
   2. File System Namespacing: If file-system access is provided, runtimes and host functions MUST restrict access to explicitly authorized resources and prevent access outside that scope.
# Operational Considerations

TBC per {{?I-D.opsarea-rfc5706bis}}.


# IANA Considerations

   This document requests IANA to create a registry for Universal Bytecode
   Interface (UBI) Functional Symbols under the Operations and
   Management Area.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
