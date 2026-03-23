# SARIF Support for Swift Diagnostics

* Proposal: [SE-0nnn](0nnn-sarif-support-for-swift-diagnostics.md)
* Authors: [Aviral Goel](https://github.com/aviralg)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/pull/NNNNN)
* Experimental Feature Flag: `--sarif-log`

## Introduction

We propose introducing support in the Swift compiler for serializing Swift diagnostics to the [Static Analysis Results Interchange Format (SARIF) v2.1.0](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.pdf), a standardized JSON-based format with a rich information model for representing static analysis results. This will facilitate integration of Swift diagnostic output with a variety of language tools and enable comprehensive diagnostic tracking via result management systems for automated code quality workflows.

This proposal is organized as follows. We begin with the **Motivation** for adding SARIF support, followed by a discussion of **Benefits** for the Open-Source Swift Community. We then describe how **Swift Diagnostics are Represented in SARIF**, providing concrete examples of the format and detailed mappings of diagnostic fields. The **Implementation Plan** outlines the architecture across three components: the Swift compiler, a new `sarif` library, and a new `swift-sarif` library. We conclude with **Future Implementation Work** that extends beyond the initial proposal and **Alternatives Considered.**

## Motivation

The Swift compiler currently emits diagnostics in [LLVM’s bitstream format](https://llvm.org/docs/BitCodeFormat.html#bitstream-format), which is not widely supported by diagnostic consumer tools such as CI systems, and the binary representation creates friction in designing these tools. SARIF is a JSON-based format for representing results from static analyzers designed specifically for consumption by tracking and reporting tools. It supports a wide variety of diagnostic metadata beyond what is currently implemented in Swift’s diagnostic bitstream representation. SARIF enjoys widespread industry adoption; major compilers, code analysis tools, development platforms, and IDEs such as MSVC, Clang, GCC, GitHub, and Visual Studio Code support SARIF. Implementing this proposal will bring the Swift compiler diagnostic reporting at par with these compilers and enable the Swift community to leverage the diagnostic analysis and visualization support of existing tools.

## Benefits

We believe that SARIF support will benefit the Swift community in the following ways.

#### 1. Cross-Tool Integration

SARIF's widespread industry adoption will enable Swift diagnostics to be consumed by a rich ecosystem of existing tools without requiring custom integration work. For example, GitHub can surface SARIF diagnostics directly in pull requests and Visual Studio Code has a plugin to display SARIF results alongside source code.

#### 2. Mixed Swift/C++ codebases

The MSVC, Clang and GCC compilers already support serializing diagnostics from C/C++ projects to SARIF format. Adding SARIF support to Swift will enable unified diagnostic tracking in mixed Swift/C++ projects. A project using both Swift and C++ can merge SARIF logs from both compilers to get a complete view of all the diagnostics.

#### 3. Improved Diagnostic Tracking

SARIF's JSON representation eliminates the need for custom parsers and converters at integration points, enabling rapid development of tools for tracking diagnostics in large-scale Swift projects for a variety of important use cases.

* **Result Management**: Tracking the introduction and resolution of diagnostics across commits and pull requests to ensure that projects stay "clean" over time.
* **Auditing**: Recording all instances of specific diagnostic categories to audit the security and correctness of codebases.
* **Suppression Tracking**: Tracking suppressed diagnostics for code with security, correctness and performance issues.

#### 4. Rich Information Model

SARIF supports a rich information model that exceeds the capability of other diagnostic formats enabling expression of a variety of program contexts beyond the standard diagnostic metadata. 

* **Logical Locations**: Fully-qualified names of functions, types, and other programmatic constructs containing the diagnostic.
* **Code Flow Paths**: Code execution paths for dataflow analysis diagnostics.
* **Stack Traces**: Call stacks with frame location information.
* **Thread Flows**: Multi-threaded execution paths for analyzing concurrency issues and understanding thread interleaving.
* **Graph Representations**: Arbitrary directed graphs, enabling visualization of object relationships, memory references, data dependencies, and execution flows.

While the scope of the current proposal is limited to the serialization of compiler diagnostics, this rich information model will allow us to easily encode static analysis diagnostics in the future. We could consider extending the existing LLVM bitstream format to include the required program context, but it will require significant effort to achieve feature parity with SARIF, and we will end up with a custom format incapable of being understood by the larger tooling ecosystem. Furthermore, SARIF objects also support inclusion of arbitrary data, not standardized as part of SARIF schema, as property bags. This flexibility will enable SARIF to be used as a unified representation for sharing and correlating findings across the Swift tooling ecosystem.

## Representing Swift Diagnostics in SARIF

This section describes the representation of Swift diagnostics in SARIF. First we will show a Swift diagnostic and its SARIF representation to present the high-level structure of the format. Then we will describe how individual diagnostic object fields map to SARIF.

Consider a `main.swift` Swift file with the following code:

```
class MyClass {
  func myMethod() {}
}

func test() {
  let obj = MyClass()
  obj.myMethod
}
```

The Swift compiler issues the following diagnostic for this file:

```
$ main.swift:7:7: error: function is unused
  obj.myMethod
  ~~~~^~~~~~~~
```

Here is a representation of this diagnostic output in SARIF.

```
{
  "$schema": "https://json.schemastore.org/sarif-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "artifacts": [
        {
          "location": {
            "uri": "main.swift"
          },
          "mimeType": "text/x-swift",
          "sourceLanguage": "swift"
        }
      ],
      "results": [
        {
          "kind": "fail",
          "level": "error",
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "index": 0,
                },
                "region": {
                  "startColumn": 7,
                  "startLine": 7
                }
              }
            }
          ],
          "message": {
            "text": "function is unused"
          },
          "ruleId": "expression_unused_function"
        }
      ],
      "tool": {
        "driver": {
          "informationUri": "https://swift.org",
          "name": "Swift Compiler",
          "organization": "Swift Project",
          "version": "6.3"
        }
      }
    }
  ]
}
```


The top-level `sarifLog` object includes the `version` and `$schema` properties that specify the SARIF standard version (2.1.0) being used, along with a `runs` array containing the diagnostic output from each execution of an analysis tool. In this example, `runs` contains output from a single invocation of the Swift compiler (version 6.3). A `run` object contains three key sections:

1. `tool` - Includes the `driver` object that contains metadata about the analysis tool that produced the results such as its name ("Swift Compiler"), version ("6.3"), organization ("Swift Project"), and a URI.
2. `artifacts` - A catalog of all source files analyzed during the run. In this case, a single file `main.swift` is listed with its MIME type (text/x-swift) and source language (swift). Artifacts are assigned indices starting at 0, and all diagnostic locations reference these indices rather than repeating full URIs, reducing file size in large-scale analyses.
3. `results` - An array of diagnostic findings. Each result object represents a single diagnostic message emitted by the tool.

A result object contains several properties that describe the diagnostic.

1. `ruleId` - A unique identifier for the diagnostic rule that was violated.
2. `message` - The human-readable text shown to developers.
3. `kind` - Indicates the nature of the result using a fixed vocabulary. The value `fail` signifies that the code violates a rule, as opposed to `pass` (code satisfies a rule) and `informational` (educational note).
4. `level` - The severity of the diagnostic: `error`, `warning`, `note`, or `none`. This diagnostic is marked as an `error`.
5. `locations` - An array specifying where the diagnostic occurred. Each `location` includes `physicalLocation`; the position in source code where the diagnostic was issued. Multiple `location` objects can be supplied for diagnostics that span multiple source locations.

This provides a high-level presentation of Swift diagnostics in SARIF. Below we describe the mapping of diagnostic fields to SARIF in more detail.

### 1. Diagnostic Identifier

Diagnostic identifier is a unique identifier for a diagnostic, represented as an enum in the Swift compiler. This maps to the `ruleId` property of the SARIF `result` object. Instead of the enum value, we will use the diagnostic name for its `ruleId`. For example, `ID` 1 will have `kind_declname_declared_here` as its `ruleId`. SARIF expects `ruleId` to be a stable identifier for a result. While the Swift compiler does not guarantee stability of these diagnostic names, they are changed infrequently and remain stable across most runs. For diagnostics whose names change over time, SARIF provides support for tracking the old names. We can add support for generating this information in the `deprecatedIds` property as needed.

### 2. Diagnostic Location

Diagnostic location represents the physical location in the source where the diagnostic was issued. This includes the start and end line and column numbers that are expressed in the `physicalLocation.region` of a `location` object. The source file is expressed in `artifactLocation.index` as an index into the `artifact` field. Swift reports column numbers in UTF-8 byte offsets whereas SARIF supports either UTF-16 offsets or Unicode code point offsets. We will convert Swift's column numbers to Unicode code point offsets.

### 3. Diagnostic Kind

Swift Diagnostics can be one of `Error`, `Warning`, `Remark`, and `Note`. The first three kind of diagnostics are represented as independent `result` objects using a combination of `kind` and `level` properties as described below:

| Diagnostic Kind | SARIF `kind`  | SARIF `level` |
|-----------------|---------------|---------------|
| Error           | fail          | error         |
| Warning         | fail          | warning       |
| Remark          | informational | note          |

Diagnostics of kind `Note` are not represented as independent `result` objects. Instead, they are represented in the `relatedLocations` property of the parent diagnostic `result` object. A `relatedLocations` entry is effectively a `physicalLocation` object with an associated text message, as shown below.

```
"relatedLocations": [
  {
    "physicalLocation": { /* ... */ },
    "message": {
      "text": "..."
    }
  }
]
```

### 4. FixIts

Swift fix-it objects map directly to SARIF `fix` objects stored in the `fixes` property of the parent diagnostic `result` object. Consider the following snippet of swift code.

```
struct Point {
  var x: Int
  var y: Int
}

let p = Point()  // error: missing arguments for parameters 'x', 'y' in call
                 // fix-it: insert ', x: <#Int#>, y: <#Int#>'
```

The corresponding `fixes` property of this error diagnostic will look like this.

```
"fixes": [
   {
     "artifactChanges": [
       {
         "artifactLocation": {
           "index": 0
         },
         "replacements": [
           {
             "deletedRegion": {
               "endColumn": 15,
               "endLine": 16,
               "startColumn": 15,
               "startLine": 16
             },
             "insertedContent": {
               "text": "x: <#Int#>, y: <#Int#>"
             }
           }
         ]
       }
     ],
     "description": {
       "text": ""
     }
   }
]
```

## Implementation Plan

We propose to implement support for the SARIF diagnostic format in the Swift compiler and associated tools by splitting the implementation across three primary components: `swift` (the Swift compiler), `sarif` library (to be created), and `swift-sarif` library (to be created), with the latter two new components serving as building blocks for the compiler implementation as well as all future clients and additional tools with a library-first design.

### 1. The Swift Compiler

We propose to make the following changes to the Swift compiler.

1. We will add a new command-line flag, `--sarif-log=`*`path`*, for supplying the path of the SARIF file for serializing the diagnostics. Swift builds use incremental compilation and parallel execution to improve build performance. To avoid race conditions from concurrent compilation tasks, we will emit one SARIF file fragment per primary input source file. Only the relevant subset of these files will be updated during incremental compilation.
2. We will introduce a new class, `SARIFDiagnosticConsumer`(`lib/Frontend/SARIFDiagnosticConsumer.cpp`), that will extend the `DiagnosticConsumer` interface to queue the diagnostics in the Swift-level `DiagnosticsBridge` as they are emitted by the compiler.
3. We will modify `DiagnosticsBridge.swift` to add support for converting and serializing these diagnostics to SARIF by invoking the appropriate APIs from `swift-sarif` library. This will require the addition of `swift-sarif` as a dependency in the compiler.

### 2. `sarif` Library

We will create a `sarif` library with the intention of providing general-purpose SARIF support in Swift to facilitate the development of warning management systems. It will provide the required subset of SARIF v2.1.0 schema, JSON serialization and deserialization support, and validation utilities for SARIF.

### 3. `swift-sarif` Library

While the changes to the Swift compiler will introduce support for queuing diagnostics as Swift objects with the intention of serializing them to SARIF, the actual conversion will be performed by the `swift-sarif` library. This ensures that the Swift compiler remains decoupled from SARIF format details. The conversion process will involve building an artifact catalog from all the diagnostics and expressing the relevant fields from diagnostic objects as SARIF. This library will import `swift-syntax` library for accessing Swift-level representation of diagnostics and `sarif` library for SARIF schema definitions.

## Future Implementation Work

This proposal establishes the foundational infrastructure for SARIF serialization of Swift diagnostics. Several important capabilities are intentionally deferred to future work to keep the initial implementation focused and achievable.

### 1. Macros

Swift macros can generate code that produces diagnostics. The current proposal does not address how diagnostics originating from macro expansions should be represented in SARIF. Future work will investigate the representation of the relationship between a diagnostic in expanded macro code and the original macro invocation site, potentially using SARIF's `physicalLocation.contextRegion` or custom property bags to capture the expansion context.

### 2. Invocation Metadata

While the current proposal only includes basic tool information, SARIF supports comprehensive metadata about the tool invocation itself through the `invocations` property. To the extent possible while maintaining deterministic output, future work could expand this to include command-line arguments to ensure reproducible builds, working directory to resolve relative paths, environment variables that affect compilation behavior, exit code to record execution success or failure, and other compiler configuration settings that affect diagnostic behavior. This metadata can be particularly valuable for auditing and compliance workflows where full provenance of diagnostic results must be maintained.

### 3. Detailed Diagnostic Location Metadata

The current proposal captures start line and column numbers for diagnostics. SARIF supports additional location metadata that could improve diagnostic presentation such as end line and column numbers, and snippet text. These features will be added as needed.

### 4. Tracking Suppressed Diagnostics

One of SARIF's key features is its support for tracking suppressed diagnostics, which is essential for warning management systems. To enable audit workflows, future work will add support for detecting and recording diagnostics that are suppressed, along with details about how those diagnostics were suppressed..

### 5. Log Size

A limitation imposed by SARIF’s JSON representation is that the SARIF logs will be larger than their equivalent LLVM-bitstream versions. We can address this in the future by providing comprehensive filtering mechanisms for SARIF generation to ensure that the logs only contain the information needed for downstream processing. We can also explore the possibility of storing SARIF logs in a compressed format.

## Alternatives Considered

The Swift compiler already implements LLVM bitstream serialization for its diagnostics. While it is attractive to consider extending this format as needed, it will require substantial effort to match SARIF's capabilities. Furthermore, it's not an industry standard and has limited adoption outside of the `clang` ecosystem.
