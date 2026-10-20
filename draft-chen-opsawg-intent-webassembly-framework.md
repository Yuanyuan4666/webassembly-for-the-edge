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

informative:
  WASM:
    title: "WebAssembly Core Specification"
    target: "https://webassembly.github.io/spec/core/"
    author:
      - org: "WebAssembly Community Group"


--- abstract

   This document describes an architecture for executing WebAssembly (WASM) modules for network management on heterogeneous edge devices. It describes dynamic bytecode dispatching, the Universal Bytecode Interface (UBI) for abstracting heterogeneous network element capabilities, and metadata for capability exposure, intent-driven discovery, and security audits. The framework decouples management scripts from platform-specific interfaces. It reuses existing mechanisms for software update, payload manifests, integrity protection, and transport where applicable.

--- middle

# Introduction

## Background and Problem statement

   As automated network management evolves, network operators increasingly rely on dynamic, closed-loop control mechanisms. Structured tool interfaces can expose network node capabilities for automated orchestration. Large language models can generate execution logic for management tasks. However, executing dynamic operational scripts or telemetry-collection logic directly on heterogeneous and constrained network devices poses security risks and resource challenges.
Container-based execution may introduce startup and memory overhead that is unsuitable for some constrained nodes. WebAssembly [WASM] provides an alternative for lightweight management tasks, subject to the capabilities and limits of the host runtime.




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
   * WIT (WebAssembly Interface Types): The interface definition language used by the WebAssembly Component Model to define types and functions crossing component boundaries.

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
{: #arch title="Intent based Framework" artwork-align="center"}

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

   Bytecode distribution MUST be atomic, secure, and self-contained.
   The network controller utilizes the UBI-DP format to encapsulate the
   binary and its metadata payload through the following lifecycle:

   Step 1: Discovery. Edge Node registers its profile, declaring local
     UBI primitives.
   Step 2: Audit. Controller matches permissions via the Static Symbol
     Audit Matrix.
   Step 3: Transport. Controller compiles code, computes SHA-256
     fingerprints, signs payloads, and transmits via HTTP Semantics
     {{?RFC9110}} or CoAP {{?RFC7252}}.
   Step 4: Execution. Edge Node validates signatures, provisions execution
     threads, isolates environments, and executes the authorized module.

## WASM Bytecode Semantic Properties and Core Capabilities

   WASM bytecode dispatched via UBI-DP MUST conform to strict structural
   and behavioral boundaries:

   a) Single-Threaded Non-Blocking Loop: To prevent thread starvation on
      low-power nodes, bytecode SHALL NOT spawn guest-level OS threads.
   b) Virtual Memory Page Restriction: Memory configuration uses fixed
      WebAssembly Memory Pages (1 page = 64 KB), statically locked at
      compilation.
   c) AOT Compliant Structure: Modules MUST allow instantaneous
      translation into native machine instructions via AOT engines,
      bypassing JIT overhead.

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

JSON Schema and payload definitions preserved to define standard data
models for node inventory management

## SUIT Manifest and Bytecode Distribution Envelope

   To securely deliver WebAssembly (WASM) bytecode components to
   constrained edge nodes, the distribution framework adopts the
   Software Updates for Internet of Things (SUIT) manifest architecture.
   The management controller encapsulates the compiled WASM binary and
   its execution policy within a CBOR-based SUIT Manifest, protected
   via CBOR Object Signing and Encryption (COSE) {{?RFC9052}}.

   The SUIT manifest enforces the strict sequential workflow required
   for trusted edge execution:

   1. Manifest Authentication: The recipient edge node validates the
      COSE signature of the SUIT manifest using pre-configured trust
      anchors.
   2. Applicability Verification: The device checks whether the
      manifest's target hardware identifiers, vendor IDs, and class IDs
      match its local hardware profile.
   3. Dependency Resolution: The node verifies and resolves any required
      UBI interface dependencies declared in the manifest.
   4. Payload Fetching: The standalone core .wasm binary payload is
      retrieved via secure transport (e.g., CoAP block-wise transfer
      {{?RFC7252}}).
   5. Installation & Sandbox Initialization: The verified WASM module is
      loaded into the isolated runtime environment with static memory
      page constraints and authorized UBI host symbols.

   The structural representation of the SUIT manifest payload embedding
   the WASM execution parameters is illustrated conceptually below:

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
             "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
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

   This section details the security mechanisms, static analysis controls,
   and boundary enforcement strategies required to ensure safe bytecode
   execution within managed network environments.

## Static Symbol Audit Matrix and Link-Time Verification

   To prevent malicious or corrupted management code from executing
   unauthorized operations, the Management Controller MUST execute a
   mandatory Ahead-of-Time Static Symbol Audit before bytecode
   encapsulation.

   The audit pipeline enforces the following validation steps for
   self-contained core modules:

   1. Import Table Extraction: The audit engine parses the WASM binary
      structure, filtering all tokens located within the "Import Section".
      This maps every external host function identifier requested by the
      bytecode.
   2. Capability-Matrix Intersection: The extracted import symbols are
      cross-checked against the specific allowed_host_symbols array
      declared in the node's profile envelope.
   3. Link-Time Rejection: The binary is scanned to ensure no raw
      hardware address references bypass the UBI layer. Any mismatch or
      inclusion of unauthorized tokens SHALL result in an immediate
      Link-Time rejection, aborting the transmission and returning
      Error 400.

## Sandbox Boundary Enforcement

   The execution of dynamically provisioned bytecode on remote physical
   devices mandates severe security practices. This framework achieves
   endogenous security via layers of defensive validation:

   1. Linear Memory Boundary Isolation: Embedded runtimes MUST
      leverage virtual memory page capping to ensure that buffer
      overflows inside the sandboxed application cannot compromise
      host address spaces.
   2. File System Namespacing: Runtimes MUST map the sandbox internal
      root '/' directory to an ephemeral host path using chroot
      primitives to avoid directory traversal vectors.

# Operational Considerations

TBC per {{?I-D.opsarea-rfc5706bis}}.

# Security Considerations

TODO Security


# IANA Considerations

   This document requests IANA to create a registry for Universal Bytecode
   Interface (UBI) Functional Symbols under the Operations and
   Management Area.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
