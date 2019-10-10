# Plugins Proposal
#plugins

# Spinnaker Plugins
| | |
|-|-|
| **Status**     | **Proposed**, Accepted, Implemented, Obsolete |
| **RFC  Num**      | _Update after PR has been made_ |
| **Author(s)**  | Rob Zienert (`@robzienert`) |
| **SIG / WG**   | _Applicable SIG(s) or Working Group(s)_ |
| **Obsoletes**  | _<RFC-#s>, if any, or remove header_ |

## Overview
This proposal builds atop prior proposals and proof of concepts for a plugin system within Spinnaker. This proposal is for an experimental body of work spanning kork and multiple services including Deck, and eventually, standardization of some development lifecycles of Spinnaker itself.

### Goals and Non-Goals
* Goals
	* Support plugins on both the frontend & backend services
	* Support plugins spanning multiple services
	* Support in-process and RPC-backed plugins for backend services
	* Provide tooling for developers and operators
* Non-goals
	* Support for alternative runtimes (e.g. Lambda, Docker; to be addressed later)
	* Dictation of what can or cannot be a plugin within Spinnaker
	* Event-based plugins
	* Finalization of extension point acceptance processes
	* Finalization of plugin development and publishing workflow
	* Finalization of plugin operator workflow

It should be clear from these lists that this proposal is focusing on the service-side of plugins and to be a launching point for discovery and subsequent narrowly-focused proposals of ecosystem work.

Initial plugin targets will be:

* Pipeline Stages
* Pipeline SpEL functions
* Micrometer backends
* Gate proxy endpoint configuration

## Motivation and Rationale
Extending Spinnaker today falls into two categories, 1) writing Java code that directly extends internal APIs, or 2) usage of web hook stages or preconfigured jobs.

In the first case, a native Spring extension model, extensions need to be written on the JVM and released in lock-step with service releases. Further, it usually requires running the Spinnaker ecosystem to validate correct behavior - a technical hurdle that many possible contributors do not have the time or patience for. In the second case, extensions can be written and tested outside of the Spinnaker release cycle, but can only affect functionality of Spinnaker in the narrow scope of a pipeline stage.

In an effort to lower the bar for Spinnaker development, a native plugin model is being proposed that can be used for federated development of Spinnaker. This native plugin system will need to support clearly defined extension points within Spinnaker at both service- and product-level abstractions, from either an in-process or remote runtime model.

By creating this plugin model, Spinnaker will be able to iterate functionality faster as changes will be more easily developed, distributed and installed without adding bloat to the core product offering. This work aims to be a quality of life improvement for Spinnaker developers, operators, and users alike.

Early stakeholders for this effort are:

* Netflix Delivery Engineering
* Netflix Cloud Database Engineering
* Armory

We have explicitly not called out the Spinnaker OSS community, as the early deliverables of this effort are experimental and won’t be subject to backwards compatibility constraints. Community members are welcome to be involved early nonetheless, provided acceptance of instability.

## Timeline
Proof of concepts were developed both by Armory and Netflix independently in 2019Q3. In Q4, we plan to deliver an alpha-quality plugin implementation that exhibits capability of extending multiple services as well as Deck, and include a basic distribution story for both in-process and out-of-process extensions.

Development will continue into and through 2020. If the plugin proposal is successful, plugin development and thereby the core plugin framework will continue iteration for the foreseeable future.

## Design
### Taxonomies
| Term | Definition |
| - | - |
| **Extension Point** | A strongly typed service- or library-defined point of extension |
| **Plugin** | An implementation of an Extension Point |

### Multiple Plugin Runtimes
There’s a place and a time for different plugin runtimes. In-process extensibility offers a lot of power and network efficiency at the cost of higher coupling to core service rollouts, whereas a remote plugin runtime trades runtime efficiency and reliability for development and release efficiency.

The goal of this proposal is to offer the best of both worlds by allowing extension point developers (Core Spinnaker developers) to decide what runtime(s) are the best for the extension.

Extension points will be defined and exposed via PF4J.

### PF4J as a Foundation
[PF4J](https://github.com/pf4j/pf4j) is to be used as the foundation for all plugin functionality. While designed for in-process extensibility, we’ll be using PF4J to define both the contract for in-process plugins, as well as remote plugins. Remote plugins would be supported in two parts: First, by including a plugin that activates a new plugin runtime (e.g. gRPC, or a Docker co-process). Second, a remote plugin stub that uses the runtime plugin as a transport.

PF4J has a lot to offer out of the box that we would need to build ourselves, regardless of final implementation. It doesn’t go as far as OSGI, but provides basic (yet extensible) ClassLoader isolation and a simple extension contract that we can use broadly. PF4J was originally built for performing in-process plugins only, but its own extension points make it feasible to enable remote plugins using the same contract. What runtime a particular plugin uses would be determined by a combination of: 1) The extension point developer allowing remote invocation, and 2) an operator selecting the particular runtime they want to use from the available options.

An extension looks like the following:

```kotlin
// Definition of the PluginStage extension point.
// The annotation defined here also informs the Spinnaker
// plugin framwework which runtimes are acceptable. This 
// annotation would additionally be used for other metadata
// Spinnaker needs to know atop whatever PF4J offers out of
// the box.
@SpinnakerExtension(drivers = [
  ExtensionDriver.LOCAL, 
	ExtensionDriver.REMOTE
])
class PipelineStageExtension : ExtensionPoint {
	// ...
}
```

It is highly recommended to read the documentation of PF4J to fully understand the scope of this proposal.

### Front50 as the Source of Truth
Operators will need the ability to understand, at a glance, what plugins are installed and enabled or disabled, their versions, as well as any other metadata about a plugin. Whether plugins are implemented as in-process or not, they Front50 should be used to track and expose a single source of truth of what plugins are installed in any particular service, as well as which ones are enabled or disabled. 

The Front50 plugin registry will also be exposed through Gate, and can be consumed by Deck to load the correct plugins on application startup.

Plugins can also be enabled or disabled at runtime without redeploying a service. This flag is sourced from Front50, which all services will be listening for change events from. Whether this change event is published or polled for initially is unknown.

Halyard configs can be used to inform services of their _initial state_. When a service starts up, it will notify Front50 of what in-process plugins are installed, however Front50 will be responsible for determining the enabled state.

Using Front50 as a source of truth will require extension of PF4J.

```proto
syntax = "proto3";

// PluginManifest is used during registration of a plugin
message PluginManifest {
	// The name of the plugin, which is used as a fully 
	// qualified identifier, including a namespace and 
	// plugin name, ex: "netflix/fast-properties"
	string name = 1;

	// A short description of the plugin
	// ex: "Provides rollout safety of runtime application 
	// config via pipelines"
	string description = 3;

	// The name of the plugin author, ex: "Netflix"
	string provider_name = 4;

	// The semver plugin version, ex: "1.0.0"
	string version = 5;

	// A list of plugin dependencies. Initially this will be
	// informational only. ex: "netflix/somedep>=1.x"
	repeated string dependencies = 6;

	// required_versions is a repeated string of service version	// requirements. e.g. "clouddriver >= 5.33.0"
	repeated string required_versions = 7;

	// A list of extension points that the plugin implements.
	// e.g. "pipelineStage"
	repeated string provided_capabilities = 8;

	// Whether or not the plugin is enabled. An unset value 
	// should be considered as true.
	boolean enabled = 9;
}
```

### Protobuf as the Remote Contract
As a departure from PF4J, plugin contracts will be defined as Protobuf messages (not services), so long as they are _remote capable_. If a plugin is in-process only, and there’s no expectation that it becomes available for remote plugins, it does not need to define its contract as protobuf.

However, for any remotely-capable extension, its contract must be defined as protobufs, rather than exposed as simple POJOs or interfaces. A key distinction is that we’re not saying gRPC: A remote transport may be Amazon SQS or Redis PubSub, not necessarily gRPC.

### Technology Compatibility Kits
_alias: TCKs_

A convention of writing TCKs for each Extension Point will be established. The concept of a TCK is borrowed from the JSR world, where compatibility tests are written to ensure that various JREs must implement to assert correct behavior of various features. For Spinnaker plugins, TCKs can be written for two different consumers: Extension Point providers and Plugin developers.

TCKs satisfy asserting correct behavior of an extension point and its associated plugin(s). For any extension point, a TCK will be written that asserts even pathological plugins behave in an expected way while integrating with the extension point. For a plugin, a TCK will be provided that a plugin developer can use to assert correctness without necessarily needing to run other Spinnaker services.

This goes further than the convention of using Protobuf to define the interfaces of a particular plugin, where Protobufs only define the interface, whereas a TCK will assert behavior that lies beneath the interface. Combining a TCK and adherence to the interface, contracts will be fully asserted between a plugin and its extension point(s).

### Up/Downgrading Plugins
This section applies to federated development scenarios, where the operator is not necessarily the same as the plugin developers; where the release cadence of a plugin does not adhere to the release cadence of the core Spinnaker services.

The upgrading (or downgrading, from now on broadly upgrading) plugin story is different based on the particular plugin runtime.

#### In-process
When a service starts up, PF4J will scan a designated locally-accessible directory for plugins to install (these plugins may be downloaded at runtime or baked into the image at built-time). Once an application has started, it will notify Front50 of the plugins that are installed, and get a response dictating which plugins are to be enabled.

In the case of a runtime-downloaded plugin, version to download will be dictated by Front50. Should Front50 be unavailable, the service will download the version defined by the service, or if none is defined, the latest plugin version that matches the Spinnaker service version.

When a new plugin version is released, it is the responsibility of the operator to release this new version. _**TBD**: How will an operator know a new version exists?_ This new version will need to be released in lock-step with the services that it is associated with. In the future, this functionality _may_ be dynamically sourced and then re-activated. See [pf4j/pf4j-update](https://github.com/pf4j/pf4j-update)

Tightly-coupled service and plugin releases is a poor developer and operator story, but an acceptable shortcoming for an initial release. By using Front50 as the source of truth, we’ll be fairly well equipped to make an acceptable runtime plugin loading story.

Assuming a world where in-process plugins can be upgraded at runtime, services will need a background process to download these new plugins whenever a delta is detected in Front50, and the plugin refreshed. PF4J offers an abstraction for this reloading mechanism, but we would need to figure out how Spring integrations are handled. For example, it’s possible we may need to make the constraint that exported beans from a plugin are not refreshed on plugin up/down grade at least initially.

#### Remote
Remote plugin upgrade process is not much different than iterating a remote service, since that’s what it is.

When a remote plugin is released, its internal functionality will change immediately, just as it would be expected for a remote service. When the plugin starts, it must send a registration event to Spinnaker. This registration event would contain, at least:

* Plugin ID
* Other plugin metadata (author, description, version, etc).
* What extension points it integrates with

Upon receiving this registration event, Spinnaker would then update the Front50 registry with the new information and downstream services would re-bootstrap the plugin, adding or removing it from extension points as necessary.

### Dependencies
A series of new kork modules will be provided for libraries and services alike to consume to expose new extension points. Under the covers, this introduces PF4J as a new global service dependency.

In-process plugins, by default, will have their own ClassLoader that is isolated from other Plugin ClassLoaders. These Plugin ClassLoaders will be children of the root ClassLoader, meaning they cannot have dependencies that conflict with the root ClassLoader. Future work may be performed to allow for different levels of isolation if necessary.

**TBD**: We would like to offer different plugin runtimes besides in-process as plugins themselves. These plugin runtimes would then come with their own dependencies that would be exposed to the parent service, such as gRPC or Lambda function invocation.

## Drawbacks
There are a handful major issues with the proposed direction:

1. In-process plugins have a poor developer and operator experience which can negatively affect end-users, especially in rollback scenarios. In-process plugins also require developers to have more knowledge of the Spinnaker toolchain and potentially the service architecture. Exposing logging, telemetry and crash reports to plugin developers for in-process plugins will be difficult or impossible dependent on a Spinnaker installation’s preferences.
2. Supporting alternative runtimes adds complexity to the final implementation and will likely require good judgment on an extension point developer to write the correct abstractions.

## Prior Art and Alternatives
https://github.com/spinnaker/spinnaker/issues/4181 … and linked PRs

The existing plugins code that has come out from [#4181](https://github.com/spinnaker/spinnaker/issues/4181)  is a successful implementation of plugins, but only supports an in-process runtime model, and is implementing a bespoke plugin system. It would be preferential to use an existing library for plugins, where possible, that we don’t need to solely maintain.

Furthermore, the existing system does not afford alternate plugin runtimes. In-process plugins have the benefit of being capable of a wide variety of changes, but come at the cost of a poor operator story in that service and plugin release cadences must be in lock-step. This model is fine for Spinnaker installations where the operator is also the plugin developer, but offers a poor story for environments where the plugin developer and Spinnaker service operators are not the same group and do not want to coordinate and be coupled to each other’s release cadences.

The proposed plugin system will aim for feature parity of the existing plugin system, but will not guarantee backwards compatibility of developer or operator experience. The priority is in satisfying broad-strokes business requirements, as opposed to satisfying existing technical capabilities.

## Known Unknowns
There are a lot of known unknowns. It’s the intention that separate proposals will be made for many of these unknowns, such as development and publishing lifecycle, as well as operator experiences.

* In-process plugins
	* How will plugin authors get metrics, logs, or crash reports from their plugins?
	* How will operators be notified that new plugin versions exist and can be released?
	* Do we want to support updating backend plugins at runtime?
* Deck plugins
	* We want to experiment with some Webpack magic to achieve plugin loading, although we’re unsure if this will be possible yet.
* 

_What parts of the design do you expect to resolve through the RFC process?_
_What parts of the design do you expect to resolve through implementation before stabilization?_
_What related issues do you consider out of scope for this RFC that could be addressed in the future?_

## Security, Privacy, and Compliance
In-process plugins present an interesting security issue. Plugins can introduce new dependencies which expose CVEs, they can access internal APIs that may break core—or other plugins—behavior. Further, a story around metrics, logs and error reporting for plugin developers becomes harder as there’s greater possibility for leaking potentially sensitive information to parties that do not necessarily have organizational approval for such data.

Initially speaking, things like logs, metrics and crash reports for in-process plugins will go unaddressed, putting the sole burden of security judgment on the Operator. 

In-process plugins are not fully isolated, so close scrutiny of a plugin’s behavior is mandatory for any Operator prior to installation or enabling.

## Operations
This proposal alters how Spinnaker development will be performed, as well as how Spinnaker functionality is released and delivered to various environments. We do not yet know the exact ways that this will change, however the goal is to make development of Spinnaker—for both core and extensions—easier.

## Risks
* There is an existing plugin system that has made assumptions on plugin loading. There is risk that switching its internal functionality to PF4J may be in some ways backwards incompatible; such as how plugins are detected, or how config is loaded for them.
	* This will likely cause breakages in existing plugins that will need to be migrated.
* Service rollbacks when in-process plugins are being used will also rollback plugin versions, which may be undesired behavior.
* Having Front50 as the source of truth for what plugins are installed across the system, and which ones are enabled or disabled introduces it as a single point of failure for a yet unknown, wide-sweeping segment of functionality for Spinnaker.
	* Should Front50 be unavailable, services should use the shipped / baked config as defaults.
	* Gate should store an in-memory cache of the desired plugin state such that Deck may have less dependence on a healthy Front50 deployment.

## Future Possibilities
_What are the natural extensions and evolution of this RFC that would affect the project as a whole?_

_Feel free to dump ideas here, they're meant to get wheels turning, and won't affect the outcome of the approval process._
