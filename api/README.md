# NEATO API

Program API environment specifications.

The NEATO API consists of the default _G, events and handlers of the provided NEET Computers API, with changes and additions defined by the specifications.

This does not require the API to explicitly behave the same as stock NEET Computers API, or that it cannot have additions from an operating system, but rather that the expected parameters and return types are the same across all OSes.

For example, in a GUI based operating system with applications that each get their own window, it may modify the behavior of the mouse event to be offset by the window's position, and the graphics API to use a specific layer for the window, rather than the main global layer, and remain NEATO compliant.

### Changes to the default API:

*Note: get started writing specs for changes to the default API.*

### Additions to the default API:

- [Terminal emulation (term.md)](term.md) - Extra functions for a standardized terminal emulator.
- [System info (sys.md)](sys.md) - Provides system information to check for OS name, version, and NEATO compatibility.
