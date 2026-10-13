---
title: "Intent-Driven WebAssembly Framework for Secure Management in Heterogeneous Edge Networks"
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
    email: "bill.wu@huawei.com"
 -
    fullname: Jianping Sun
    organization: China Telecom
    email: "bill.wu@huawei.com"

normative:

informative:


--- abstract

   This document specifies a standardized, lightweight, sandboxed
execution framework based on WebAssembly (WASM) for managed network
environments and constrained edge devices. It defines operational
mechanisms for dynamic bytecode dispatching, the Universal Bytecode
Interface (UBI) for abstracting heterogeneous network element
capabilities, and metadata models for capability exposure, bytecode
deployment, execution context, intent-driven discovery, and security
audits. This framework provides a systematic approach to decouple network management and operational scripts from specific hardware platforms, enabling secure, deterministic, and low-latency execution of management tasks.


--- middle

# Introduction

## Background and Problem statement

   As automated network management and intent-based networking (IBN)
   evolve, network operators increasingly rely on dynamic, closed-loop
   control mechanisms.  Frameworks utilizing structured tool interfaces
   (such as Model Context Protocol paradigms) are emerging to expose
   network node capabilities for automated orchestration.  Against this
   backdrop, the application of AI-generated code in network operations
   is becoming increasingly widespread, and automatically generating
   execution logic that meets management requirements via large
   language models has become an important means to enhance operational
   efficiency. However, executing dynamic operational scripts or telemetry-
   collection logic directly on heterogeneous and constrained network
   elements (e.g., edge gateways, IoT routers, MCU-based switches) poses
   severe security risks and resource challenges.

   Traditional management virtualization technologies like Linux
   Containers (Docker) introduce excessive boot latencies and high
   memory footprints, making them unsuitable for constrained nodes or
   microsecond-level telecommunication response loops.

## Deterministic Execution in Managed Edge Networks

   In modern distributed network architectures, edge nodes are no
   longer simple packet-forwarding entities but have evolved into
   compute-integrated network elements.  When network management
   systems orchestrate dynamic traffic engineering, local fault
   remediation, or real-time telemetry, they require on-demand
   execution of management scripts at the closest data-plane edge.

   However, managed networks enforce strict SLAs concerning jitter,
   packet loss, and power budgets.  Traditional monolithic containers
   fail to meet these operational constraints.  WASM sandboxing
   bridges this gap by decoupling management logic from the underlying
   hardware Board Support Package (BSP), transforming network-layer
   compute into an ephemeral, managed pipeline.

## Use Cases

   Looking ahead, the integration of WebAssembly-based sandboxing,
   Universal Bytecode Interfaces, and AI-driven orchestration is
   expected to redefine the operational boundaries of autonomous
   networks.  As 6G architectures, deterministic edge computing, and
   large-scale high-density network deployments mature, network
   management will shift from reactive scripting to proactive,
   intent-driven closed-loop automation.

   Specifically, future application scenarios will likely encompass:

   a) Autonomous Edge Healing: AI agents dynamically generating,
      auditing, and pushing localized remediation bytecode to edge
      nodes within milliseconds of network anomaly detection,
      bypassing human intervention loops.
   b) Cross-Vendor Network Function Virtualization: Universal execution
      of proprietary traffic engineering and monitoring tasks across
      heterogeneous equipment vendors without requiring specialized
      board support packages (BSPs) or proprietary firmware.
   c) Distributed Telemetry and Deterministic Control: Distributing
      lightweight computation pipelines across resource-constrained
      forwarding nodes and access gateways to optimize real-time
      telemetry processing and closed-loop control at the data-plane
      periphery.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

   * WASM Sandbox: A lightweight, isolated virtual machine environment
     running inside a host process, executing WebAssembly bytecode with
     linear memory isolation.
   * UBI (Universal Bytecode Interface): An abstraction layer that exposes
standardized host functions to WASM modules. UBI interfaces describe
available device capabilities and hide hardware-specific
implementations.
   * Node Profile: A structured metadata model used by the framework to
advertise a network element's supported UBI interfaces, execution
capabilities, and runtime information.
   * UBI-DP (Universal Bytecode Distribution Protocol): The encapsulation
     and messaging mechanism used to dispatch compiled WASM bytecode
     binaries along with metadata across network management channels.
   * WIT (Wasm Interface Type): A WebAssembly standard IDL format used to
     define types and functions crossing the sandbox boundary.

#  Architectural Framework Overview

   The proposed framework consists of three operational planes
   aligned with standard network management architectures:

~~~~
   +--------------------------------------------------------+
   |        Intent & Orchestration Plane (Management)       |
   | [Intent Parsing] -> [Workflow Compile] -> [Cap. Match] |
   +--------------------------------------------------------+
                                |
                                v (UBI-DP Dispatching)
   +--------------------------------------------------------+
   |        Coordination & Policy Plane (Control Layer)     |
   | [Static Audit Matrix] -> [CaaS Pool] -> [SHA256 Sign]  |
   +--------------------------------------------------------+
                                |
                                v
   +--------------------------------------------------------+
   |        Embodied Execution Plane (Device Layer)         |
   | [Warm Process Pool] -> [WASM Sandbox] -> [UBI Drivers] |
   +--------------------------------------------------------+
~~~~
 {:#arch title="Intent based Framework" artwork-align="center"}

   1. Intent & Orchestration Plane: Translates network operator intents
      into structured workflow models and maps required management tools.
   2. Coordination & Policy Plane: Acts as the central management
controller. It evaluates requested tasks against security policies,
performs deterministic compilation via Compilation-as-a-Service (CaaS),
and generates signed bytecode distribution packages.
   3. Embodied Execution Plane: The physical edge node running embedded
      WASM runtimes. It manages a warm process pool to guarantee low
      sandbox setup latency (<10ms) and restricts hardware access via
      UBI symbols.

# Core Operational Mechanisms

## Dynamic Resource and Capability Discovery Mechanism

   To establish reliable task routing, the management framework relies
   on an automated Discovery Mechanism to map requirements to
   heterogeneous resources:

   1. Capability Advertisement: Upon network attachment or status update,
      the Edge Node registers its capability information using a Node Profile. The profile describes platform information, supported UBI interfaces, and available execution capabilities. For highly constrained nodes, this registration SHOULD use CoAP {{?RFC7252}}.  For standard elements, it MAY use gRPC or HTTP/3 {{?RFC9114}} over TLS.

   2. Semantic Feature Graph: The Central Management Controller parses
      incoming profiles into an active Resource Capability Graph cataloged
      by sensing domains and computational budgets.

   3. Intent-Driven Matching Filter: When an operator emits a management
      workflow, the controller queries the feature graph to match valid
      target nodes supporting the required UBI primitives.

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

The framework uses three types of manifests for different lifecycle
stages:

1. Platform Manifest: describes device capabilities and execution
environment.

2. Data Collection Manifest: describes data subscriptions and runtime
collection context.

3. SUIT Manifest: describes secure software delivery and execution
parameters for WASM components.

These manifests address different lifecycle stages and avoid
duplicating information.

## Platform Capability Manifest

The Platform Manifest describes the capabilities and execution environment of an edge node.

It can reuse the platform information model defined in
draft-ietf-opsawg-collected-data-manifest.

The WASM/UBI extension may include:

- WASM runtime information;
- supported UBI interfaces;
- available host functions;
- security capabilities.

~~~~
{
  "ubi-wasm-extension:server-metadata": {
    "runtime-version": "wasm-runtime-v1",
    "supported-ubi-interfaces": [
      "telemetry.read-metric",
      "actuation.gpio-write"
    ]
  },
  "ubi-wasm-extension:governance-and-security": {
    "authentication-supported": [
      "mTLS_v1.3"
    ],
    "capability-tokens-defined": [
      "telemetry_read",
      "net_outbound"
    ]
  }
}
~~~~

## Data Collection Manifest

The Data Collection Manifest describes data collection subscriptions and runtime collection context.

It reuses the data collection model defined in
draft-ietf-opsawg-collected-data-manifest.

The manifest may reference external resource policies using policy identifiers.

~~~~
{
 "data-collection": {
    "platform-id": "edge-node-hetero-01",
    "subscription-id": 1001,
    "current-period": 300,
    "policy-id": "tier_heavy_compute",
    "runtime-status": {
       "cpu-percent": 87.5,
       "action-triggered": "forced-downsampling"
    }
 }
}
~~~~

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

   The following example illustrates possible WASM-specific execution parameters carried by a SUIT Manifest. Existing SUIT fields such as manifest version, component identification, authentication, and payload retrieval follow the SUIT specification.

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

## UBI Resource Constraint Policy

Resource constraints are modeled separately from manifests.

Platform Manifest describes available capabilities.
Data Collection Manifest references runtime policies.
SUIT Manifest describes execution constraints.

The resource policy defines:
- concurrency limits;
- rate limits;
- sandbox quotas;
- resource isolation rules.

Manifests reference the policy using policy identifiers.

~~~~

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
      declared in the Node Profile or execution policy.
   3. Link-Time Rejection: The binary is scanned to ensure no raw
      hardware address references bypass the UBI layer. Any mismatch or
      inclusion of unauthorized tokens SHALL result in an immediate
      Link-Time rejection, aborting the transmission and the transmission is rejected.

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
