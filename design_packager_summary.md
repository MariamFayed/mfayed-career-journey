# Design Packager Architecture Summary

## Architectural Idea

Design Packager is a layered system that transforms an observed execution into a reproducible package.

Its central abstraction is a dependency graph: commands, files, environments, working directories, and generated artifacts are modeled as related inputs and outputs. The package is generated only after that graph has been collected and normalized.

It is therefore a dependency-discovery and reconstruction framework, rather than a direct file-copy utility.

## System Flow

```text
Input script or command
        |
        v
Input extraction
        |
        v
Normalized Command model
        |
        v
Command interpretation
        |
        v
File and environment discovery
        |
        v
Recursive file parsing
        |
        v
Dependency metadata graph
        |
        v
Deferred path rewriting
        |
        v
Package materialization and archive
```

The pipeline is intentionally stable while individual input front ends and semantic definitions vary.

## Layered Architecture

The architecture separates orchestration, interpretation, discovery, and materialization.

### Orchestration layer

`SessionManager` coordinates phase ordering and session state. It invokes extraction, preprocessing, processing, generation, reporting, and archival without embedding every parsing rule.

### Input and extraction layer

`Input`, `CommandExtractor`, and `CommandExtractionStrategy` convert different input mechanisms into one internal `Command` model.

Extraction may come from a direct command, a script executed through wrappers, or an external converter. The downstream pipeline is insulated from the original source of the command.

### Interpretation layer

`ToolsOptionsMgr`, `OptTemplate`, `CmdParser`, and `CmdLineMgr` interpret normalized commands.

The parser is generic; variable command semantics are supplied as metadata. This keeps the architecture from becoming one parser implementation per tool.

### Discovery layer

`FileProcessingManager` resolves paths, deduplicates files, recursively processes dependencies, and coordinates intermediate representations. `FileMetaData` stores the dependency graph and the information needed to reconstruct each file.

### File parsing layer

`LineMgr` defines the common file-processing contract. Specialized managers and format parsers translate different file representations into common line items, which the discovery layer can consume without knowing parser details.

### Materialization layer

`OutputStructure` converts the normalized graph and intermediate records into the final package layout. This includes final path assignment, hierarchy preservation or reduction, file reconstruction, generated scripts, symbolic links, and archive creation.

## Metadata-Driven Semantics

The architecture moves variable behavior into declarative metadata wherever possible:

- Options and option values
- File arguments and file formats
- Environment-variable rules
- Predefined files
- Search precedence
- Included or cross-tool options
- Output directories and configuration syntax

`ToolsOptionsMgr` loads these definitions into `OptTemplate` structures. The common parser then interprets commands using those templates.

```text
Tool metadata
      |
      v
OptTemplate
      |
      v
Generic CmdParser
      |
      v
Generic CmdLineMgr
      |
      v
Common FileProcessingManager
```

This is the primary scalability mechanism: many new command forms can be added without changing the parsing or packaging algorithm.

## Extension Seams

### Command extraction strategy

New input mechanisms can be integrated by implementing the extraction strategy interface and producing the existing `Command` representation.

### Declarative command semantics

New options, file arguments, environment rules, and search rules can be represented in XML and consumed by the existing parser.

### Embedded command languages

A specialized front end can recognize commands embedded in a script, then delegate interpretation to the common parser. `QFTLineManager` is an example of this boundary: domain-specific recognition stays at the edge while file discovery and output generation remain shared.

### File parser polymorphism

New file representations can be supported by adding a parser or `LineMgr` implementation behind the existing interface.

### Output strategy

Alternative package layouts can be supported by implementing another output-structure strategy and selecting it through the output abstraction.

## Design Patterns and Mechanisms

### Strategy

Command extraction varies behind `CommandExtractionStrategy` while the rest of the system uses one command model.

### Factory

Factories select input, line-parser, and output-structure implementations at runtime.

### Template method and polymorphism

`LineMgr` defines the common file-processing contract while specialized managers implement format-specific behavior.

### Composite

Nested command options are represented through composite command-line option objects.

### Registry

Executable names, file types, tool definitions, and converter mappings are resolved through registries.

### Graph model

`FileMetaData` and discovered-file maps form a dependency graph that supports deduplication and multiple parent relationships.

### Intermediate representation

Intermediate records defer final path decisions until the dependency graph is complete. Parsing and output layout therefore remain separate phases.

### Singleton services

Session, file-processing, tool-definition, and logging services use shared singleton access. This simplifies coordination but increases global coupling.

## Extension Cost Model

The cost of adding functionality depends on which abstraction boundary is affected.

### Low-cost extension

Usually metadata-only:

- Add an option or option value
- Associate a file with an option
- Add an environment-variable rule
- Add a search path or predefined file

### Medium-cost extension

Metadata plus a small front-end change:

- Add an embedded command family
- Add a command-prefix recognizer
- Add a converter mapping

### High-cost extension

Requires new runtime or parsing behavior:

- Add a standalone executable
- Add a new source-file language
- Add new tool-specific runtime artifacts
- Add a fundamentally different output layout

A new executable may require changes to `TargetExec`, executable registries, wrapper generation, preprocessing, and runtime handling. A new source format may require a new `lineType`, parser, `LineMgr`, and dispatch path.

## Architectural Tradeoffs

### Strengths

- Shared pipeline across different input mechanisms
- Declarative semantics for many variations
- Separation between command interpretation and file discovery
- Recursive processing centralized in one manager
- Dependency graph retained until output generation
- Multiple output strategies are conceptually supported

### Constraints

- Executable and file-type registries introduce compile-time coupling.
- Embedded command recognizers may require procedural registration.
- Global singleton state makes independent sessions and isolated testing harder.
- Current-directory changes and shared mutable maps constrain concurrency.
- Special cases can accumulate when a new behavior does not fit the metadata model.

## Main Architectural Principle

The central separation is between:

```text
What an input command means
```

and:

```text
What packaging does with discovered dependencies
```

Metadata and front-end adapters answer the first question. The shared pipeline answers the second.

## Final Characterization

Design Packager is a layered, metadata-driven dependency packaging framework with:

- Pluggable input extraction
- A normalized command model
- Declarative command semantics
- Polymorphic file parsing
- Recursive dependency discovery
- A retained dependency graph
- Deferred path rewriting
- Configurable package materialization

Its flexibility comes from keeping the packaging algorithm generic while allowing the input front end, command vocabulary, file semantics, and output strategy to vary behind defined abstraction boundaries.
 