# Westeros Graphics Layer Southbound Interface

**RDK GRAPHICS SOUTHBOUND INTERFACE 

*DRM-based platform adaptation for the Westeros compositor and GL renderer*

| Document field | Value |
|---|---|
| Component | `westeros-gl-drm / westeros-gl` |
| Pattern | Aligned with the Westeros SoC HAL and `rdk-halif-libdrm` documentation structure |
| Interface category | Legacy graphics-layer southbound interface |
| Date | 8 September 2026 |

> **Scope note:** This specification documents the graphics backend boundary represented by the `WstGL*` contract and proposes requirements for treating it as a stable SoC-facing interface. Proposed requirements are explicitly separated from source-observed behavior.

[Source path](https://github.com/rdkcentral/westeros-gl-drm/tree/develop/westeros-gl) | [Repository](https://github.com/rdkcentral/westeros-gl-drm)

## Table of Contents

1. Acronyms, Terms and Abbreviations
2. Description
3. Introduction
4. References
5. Architecture and Boundary
6. Runtime Execution Requirements
7. Non-functional Requirements
8. Licensing and Build Requirements
9. Variability Management
10. Interface API Documentation
11. Theory of Operation
12. General Graphics Code Flow
13. SoC Implementation Requirements
14. Validation Checklist

## 1. Acronyms, Terms and Abbreviations

| Term | Meaning |
|---|---|
| ABI | Application Binary Interface |
| API | Application Programming Interface |
| DMA-BUF | Linux file-descriptor-based buffer sharing |
| DRM | Direct Rendering Manager |
| EGL | Native-platform interface between rendering APIs and platform window systems |
| GBM | Generic Buffer Management |
| GL | OpenGL or OpenGL ES rendering layer |
| GPU | Graphics Processing Unit |
| HAL | Hardware Abstraction Layer |
| KMS | Kernel Mode Setting |
| SoC | System on Chip |
| WstGL | Westeros graphics southbound API prefix |

## 2. Description

The Westeros graphics layer southbound interface is the platform-facing graphics backend used by Westeros to initialize graphics services, create native rendering targets, expose native pixmaps, report display information, and terminate graphics resources. The DRM provider supplies the implementation used for DRM-based platforms, while the renderer and compositor remain above this boundary.

> **Interface objective:** Give the common Westeros compositor and renderer a small platform-neutral contract while allowing the SoC implementation to use DRM/KMS, GBM, EGL native objects, or equivalent vendor services.

| Layer | Responsibility | Typical artifacts |
|---|---|---|
| Applications / clients | Submit Wayland surfaces and application content | Wayland clients |
| Westeros compositor | Manages displays, surfaces, composition policy, and lifecycle | Westeros common |
| Westeros GL renderer | Performs composition using EGL/GL and native targets | Westeros renderer |
| Graphics southbound interface | Provides `WstGL*` platform services and native objects | `westeros-gl.h` / provider implementation |
| SoC / kernel graphics stack | Implements display, allocation, scanout, synchronization, and GPU integration | DRM/KMS, GBM, EGL, vendor driver |

## 3. Introduction

Westeros is a lightweight Wayland compositor library designed for embedded systems. The graphics southbound boundary isolates the common compositor and renderer from the device-specific implementation needed to create native windows and pixmaps and to interact with the display stack. Internal program material identifies `westeros-gl-drm` as the DRM graphics-layer integration and `westeros-gl-brcm` as a Broadcom graphics-layer integration.

The document is intended to:

- Document the current `WstGL*` role and lifecycle.
- Define implementation expectations for SoC vendors.
- Provide a baseline for versioning, testing, and publishing the interface under a Westeros HAL documentation repository.
- Keep this legacy interface distinct from newer graphics-player or plane-control HAL definitions.

## 4. References

- [Westeros GL DRM source path](https://github.com/rdkcentral/westeros-gl-drm/tree/develop/westeros-gl)
- [Westeros GL DRM repository](https://github.com/rdkcentral/westeros-gl-drm)
- [Westeros common repository](https://github.com/rdkcentral/westeros)
- [LibDRM HAL documentation pattern](https://github.com/rdkcentral/rdk-halif-libdrm)

> **Source-access note:** The named public path was used as the governing source reference. The document avoids asserting private structure layouts or unverified provider-specific behavior where the source page did not expose sufficient detail.

## 5. Architecture and Boundary

### 5.1 Northbound callers

- Westeros compositor startup and display management.
- Westeros GL renderer when creating an EGL-compatible native rendering target.
- Embedded renderer paths that require native window or pixmap access.
- Test components that validate backend initialization, dimensions, or native-object lifecycle.

### 5.2 Southbound dependencies

- Linux DRM/KMS for display-resource and scanout control on DRM platforms.
- Native buffer/window allocation services such as GBM where selected by the provider.
- EGL platform integration and the SoC GPU driver.
- Platform display, synchronization, and memory-management services required by the selected implementation.

### 5.3 Boundary rules

| Rule | Requirement |
|---|---|
| Separation | Common compositor and renderer code shall not include private SoC driver headers. |
| Native objects | Native window/pixmap handles may be opaque, but ownership and valid lifetime shall be documented. |
| Policy | Composition policy stays above the provider; hardware execution stays below it. |
| Compatibility | Provider substitution shall not require application or Wayland-client changes. |
| Extensions | Vendor extensions shall not alter mandatory API semantics. |

## 6. Runtime Execution Requirements

### 6.1 Initialization and Startup

- Initialize the graphics backend before creating native windows or requesting native pixmaps.
- Validate required display and graphics resources and return failure synchronously.
- Support deterministic cleanup after partial initialization failure.
- Document whether one or multiple backend contexts and displays are supported.

### 6.2 Threading Model

The interface specification shall identify whether each WstGL operation is thread-safe, caller-serialized, or restricted to the compositor/render thread. A provider shall protect shared display and allocation state and shall not destroy resources still in use by rendering or presentation operations.

### 6.3 Process Model

The graphics provider executes in the process hosting the Westeros compositor. Native handles are consequently process-local unless the contract explicitly defines DMA-BUF or another shareable representation.

### 6.4 Memory and Object Ownership

| Resource | Ownership expectation |
|---|---|
| Graphics context | Created by initialization and released by termination. |
| Native window | Created by the provider and destroyed by the matching destroy function. |
| Native pixmap wrapper | Released through the matching WstGL release operation. |
| DMA-BUF file descriptor | Transfer, borrow, or duplicate semantics shall be explicit. |
| EGL display/context/surface | Ownership between renderer and provider shall be documented and not assumed. |
| DRM/GBM resources | Released by the layer that allocates or imports them. |

### 6.5 Power Management

The interface does not define a standalone power-management policy. Termination and display teardown shall release or quiesce resources so platform power management is not prevented by stale graphics objects.

### 6.6 Asynchronous Notifications and Blocking

The minimal WstGL contract is principally request/response oriented. If a provider adds page-flip, hot-plug, fence, or display-change callbacks, callback thread, ordering, cancellation, and lifetime rules shall be defined. Public calls shall not block indefinitely.

### 6.7 Error Handling and Persistence

- Report initialization and object-creation failures through stable return values.
- Preserve detailed provider diagnostics without exposing vendor-only status values in the common ABI.
- Treat double-destroy, stale handles, invalid dimensions, and unavailable displays as controlled failures.
- No persistent configuration is required by the interface unless separately specified.

## 7. Non-functional Requirements

| Area | Requirement |
|---|---|
| Logging | Provide errors for initialization, allocation, import, display, and release failures; verbose frame logging disabled by default. |
| Performance | Avoid avoidable per-frame allocation and copying; support efficient native-buffer reuse. |
| Resource limits | Bound native windows, pixmaps, imported buffers, and pending display operations. |
| Security | Validate dimensions, formats, modifiers, handles, and descriptor ownership before use. |
| Quality | Warnings-as-errors, static analysis, leak detection, and negative tests for owned code. |
| Reliability | Repeated initialize/create/render/destroy/terminate cycles shall not leak or deadlock. |

## 8. Licensing and Build Requirements

| Build item | Requirement |
|---|---|
| Library | Produce the graphics-provider library expected by the Westeros build and runtime integration. |
| Header | Install a stable public `westeros-gl.h` or equivalent contract header. |
| Dependencies | Declare DRM, GBM, EGL, and vendor dependencies without exposing private headers northbound. |
| Independent validation | Allow provider compilation and interface testing outside a complete product image where dependencies exist. |
| Symbols | Export only supported ABI symbols; hide provider-internal implementation. |
| License | Retain applicable repository and third-party license/notice obligations. |

## 9. Variability Management

| Variation | Contract treatment |
|---|---|
| DRM vs vendor provider | Same mandatory WstGL semantics; provider-selected implementation. |
| Native window type | Opaque handle with documented EGL compatibility. |
| Native pixmap type | Opaque wrapper or platform-native handle with paired release. |
| Display dimensions | Provider-reported width and height using defined units and timing. |
| Pixel format/modifier support | Capability discovery or explicit supported-set documentation. |
| DMA-BUF | Optional capability with plane, fd, offset, stride, modifier, and ownership definitions. |
| Multi-display | Optional capability; display selection and handle scoping must be versioned. |
| Explicit synchronization | Optional extension using documented acquire/release fence semantics. |

## 10. Interface API Documentation

> **Observed contract:** Public descriptions identify a compact `WstGL*` API centered on initialization, termination, native-window creation/destruction, native-pixmap acquisition/release, dimension queries, and EGL-native-pixmap access. Exact prototypes shall be copied from the approved source header before publication.

| API family | Purpose | Normative expectation |
|---|---|---|
| `WstGLInit` | Initialize the graphics backend | Return success/failure and establish context lifetime. |
| `WstGLTerm` | Terminate the backend | Release provider-owned resources after child objects are destroyed. |
| `WstGLCreateNativeWindow` | Create an EGL-compatible native window | Validate dimensions; return opaque handle or failure. |
| `WstGLDestroyNativeWindow` | Destroy matching native window | Accept only handles created by the same provider/context. |
| `WstGLGetNativePixmap` | Acquire or wrap a native pixmap | Define source buffer, format, dimension, and ownership semantics. |
| `WstGLGetNativePixmapDimensions` | Query pixmap width and height | Return dimensions without changing object ownership. |
| `WstGLGetNativePixmapEGLNativePixmap` | Obtain EGL-compatible pixmap representation | Handle remains valid only for the documented pixmap lifetime. |
| `WstGLReleaseNativePixmap` | Release pixmap wrapper/import | Release provider references and imported resources deterministically. |

### 10.1 Required API documentation fields

- Exact C prototype and symbol name.
- Parameter direction, units, valid ranges, nullability, and structure version.
- Return values and error mapping.
- Threading and reentrancy requirements.
- Ownership and lifetime of every input and output.
- Mandatory versus optional capability.
- Provider cleanup behavior when the caller terminates with outstanding objects.

## 11. Theory of Operation

### 11.1 Initialization

The compositor or renderer initializes the graphics provider. The provider establishes access to the selected platform display and graphics services and prepares the state needed for native object creation.

### 11.2 Native window and EGL surface

The renderer requests a native window and uses that native object when creating its EGL rendering surface. The renderer performs its graphics work above the southbound interface, while the provider supplies the platform-compatible target and corresponding lifecycle.

### 11.3 Native pixmap path

When a render or embedded path requires a native pixmap, the provider imports, wraps, or allocates the underlying resource, reports its dimensions, exposes the EGL-compatible representation, and releases it through the matching API.

### 11.4 Presentation boundary

The graphics interface supplies platform objects and execution services. Wayland surface policy, scene composition, damage, and application lifecycle remain responsibilities of the common compositor and renderer unless a separately approved extension assigns them to the provider.

## 12. General Graphics Code Flow

1. **Compositor startup:** Westeros selects and initializes the installed graphics provider.
2. **Backend discovery:** Provider validates display and graphics services and creates its context.
3. **Native-window creation:** Renderer requests a platform-native window for the target size.
4. **EGL setup:** Renderer uses the native target to create its EGL surface and GL rendering state.
5. **Composition loop:** Wayland surfaces are composed by the renderer and submitted through the native target.
6. **Pixmap use when required:** Provider imports or wraps a native pixmap and exposes dimensions and EGL-native representation.
7. **Resource release:** Pixmap and native-window objects are released through matching API calls.
8. **Termination:** Renderer/compositor tears down EGL state, then terminates the graphics provider.

## 13. SoC Implementation Requirements

| ID | Requirement | Level |
|---|---|---|
| GFX-WST-001 | Implement every mandatory WstGL operation | Required |
| GFX-WST-002 | Support deterministic init/term and partial-failure cleanup | Required |
| GFX-WST-003 | Create native targets compatible with the documented EGL platform | Required |
| GFX-WST-004 | Pair every created window and pixmap with matching release behavior | Required |
| GFX-WST-005 | Define all native-handle and fd ownership | Required |
| GFX-WST-006 | Validate dimensions, formats, strides, offsets, modifiers, and handles | Required |
| GFX-WST-007 | Keep private vendor types out of the common public header | Required |
| GFX-WST-008 | Avoid unbounded blocking in public API calls | Required |
| GFX-WST-009 | Document thread affinity and synchronization requirements | Required |
| GFX-WST-010 | Provide stable error reporting and diagnostic logging | Required |
| GFX-WST-011 | Expose DMA-BUF only with explicit plane and fence semantics | Conditional |
| GFX-WST-012 | Expose multi-display only with versioned display selection | Conditional |
| GFX-WST-013 | Provide ABI version and capability discovery | Target architecture |
| GFX-WST-014 | Pass lifecycle, negative, leak, and stress validation | Required |
| GFX-WST-015 | Validate integration with the common Westeros renderer | Required |

## 14. Validation Checklist

| Area | Minimum validation |
|---|---|
| Build | Clean supported configurations, symbol audit, dependency and license check. |
| Initialization | Valid device, missing device, permission failure, partial initialization failure. |
| Native windows | Multiple sizes, invalid dimensions, create/destroy loops, outstanding-object termination. |
| EGL integration | Surface creation, render, swap, resize, and teardown on supported platform. |
| Pixmaps | Acquire, dimensions, EGL native conversion, release, invalid and repeated-release cases. |
| Buffers | Supported formats/modifiers, DMA-BUF ownership, multi-plane handling where applicable. |
| Display | Resolution reporting, mode changes or hot-plug only where supported. |
| Concurrency | Documented thread model, racing create/destroy, termination during operations. |
| Robustness | Memory, fd, GPU object, mapping, and thread leak checks. |
| End-to-end | Representative Wayland client rendering through compositor, renderer, provider, and driver. |
