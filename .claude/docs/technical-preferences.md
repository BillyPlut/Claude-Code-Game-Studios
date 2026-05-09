# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Unreal Engine 5.7
- **Language**: C++ (primary), Blueprint (gameplay prototyping)
- **Rendering**: Lumen (Global Illumination), Nanite (virtualized geometry)
- **Physics**: Chaos Physics

## Input & Platform

<!-- Written by /setup-engine. Read by /ux-design, /ux-review, /test-setup, /team-ui, and /dev-story -->
<!-- to scope interaction specs, test helpers, and implementation to the correct input methods. -->

- **Target Platforms**: PC (Steam / Epic)
- **Input Methods**: Keyboard/Mouse, Gamepad
- **Primary Input**: Gamepad
- **Gamepad Support**: Full
- **Touch Support**: None
- **Platform Notes**: PC-first. All UI must support both keyboard/mouse and gamepad navigation. No hover-only interactions.

## Naming Conventions

- **Classes**: Prefixed PascalCase — `A` (Actor), `U` (UObject), `F` (struct), `I` (Interface), `E` (enum)
- **Variables**: PascalCase (e.g., `MoveSpeed`, `MaxHealth`)
- **Functions**: PascalCase (e.g., `TakeDamage()`, `GetCurrentHealth()`)
- **Booleans**: `b` prefix (e.g., `bIsAlive`, `bIsGrounded`)
- **Files**: Match class name without prefix (e.g., `PlayerController.h` / `PlayerController.cpp`)
- **Scenes/Levels**: PascalCase .umap (e.g., `Level_GoldenPeople.umap`)
- **Constants**: PascalCase or `ALL_CAPS` for macros

## Performance Budgets

- **Target Framerate**: [TO BE CONFIGURED]
- **Frame Budget**: [TO BE CONFIGURED]
- **Draw Calls**: [TO BE CONFIGURED]
- **Memory Ceiling**: [TO BE CONFIGURED]

## Testing

- **Framework**: Unreal Automation Framework (FAutomationTestBase) + Functional Tests
- **Minimum Coverage**: [TO BE CONFIGURED]
- **Required Tests**: Balance formulas, gameplay systems, networking (if applicable)

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]

## Engine Specialists

<!-- Written by /setup-engine when engine is configured. -->
<!-- Read by /code-review, /architecture-decision, /architecture-review, and team skills -->
<!-- to know which specialist to spawn for engine-specific validation. -->

- **Primary**: unreal-specialist
- **Language/Code Specialist**: ue-blueprint-specialist (Blueprint graphs) or unreal-specialist (C++)
- **Shader Specialist**: unreal-specialist (Materials, Lumen configuration, Niagara VFX)
- **UI Specialist**: ue-umg-specialist (UMG widgets, CommonUI, input routing, widget styling)
- **Additional Specialists**: ue-gas-specialist (Gameplay Ability System, attributes, gameplay effects), ue-replication-specialist (property replication, RPCs — for future multiplayer if needed)
- **Routing Notes**: Invoke primary for C++ architecture and broad engine decisions. Invoke Blueprint specialist for Blueprint graph architecture and BP/C++ boundary design. Invoke GAS specialist for all ability and attribute code. Invoke UMG specialist for all UI implementation.

### File Extension Routing

<!-- Skills use this table to select the right specialist per file type. -->

| File Extension / Type                              | Specialist to Spawn        |
|----------------------------------------------------|----------------------------|
| Game code (.cpp, .h files)                         | unreal-specialist          |
| Shader / material files (.usf, .ush, Material assets) | unreal-specialist       |
| UI / screen files (.umg, UMG Widget Blueprints)    | ue-umg-specialist          |
| Scene / level files (.umap, .uasset)               | unreal-specialist          |
| Plugin files (.uplugin, modules)                   | unreal-specialist          |
| Blueprint graphs (.uasset BP classes)              | ue-blueprint-specialist    |
| General architecture review                        | unreal-specialist          |
