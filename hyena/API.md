# API Design

## Overview

This document describes the API design philosophy and usage patterns for Hyena. Specific API details and examples are distributed throughout the implementation-specific documentation files (ARCHITECTURE.md, RENDERER.md, MATERIALS.md, etc.).

## Design Philosophy

Hyena's API design focuses on:

- **Minimal abstraction**: Direct control over WebGPU/WebGL2 without excessive wrapper layers
- **Type safety**: Leveraging TypeScript for compile-time safety
- **Performance**: Zero-cost abstractions where possible
- **Clarity**: Explicit over implicit behavior

## Usage Examples

Detailed usage examples and API patterns can be found in:
- **ARCHITECTURE.md**: Core engine initialization and configuration
- **SCENE-GRAPH.md**: Scene hierarchy and object management
- **MATERIALS.md**: Material system API
- **GEOMETRY.md**: Geometry creation API
- **ANIMATION.md**: Animation system API
- **REACT.md**: React bindings and declarative API

## Configuration

Engine configuration examples are provided throughout the relevant documentation files, particularly in sections labeled "Configuration API".
