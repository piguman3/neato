# NEATO `requestNeato` API Specification

Extension: `core`

Version: 4

---

All NEATO compliant environments, even those that are only compliant via compatibility layer, should include this
function under the name `_G.requestNeato`. This function is used for non-NEATO compliant environments to be able
to request the environment be changed into a compatibility layer, which makes the environment officially considered
NEATO compliant. An environment that is NEATO compliant out-of-the-box still must define `_G.requestNeato`, but
it can simply be a NOP function. After calling this function, a program should be certain that its global environment
matches that which is defined in the [API spec](../README.md), the [\_G spec](../globals.md), and the NEATO API module as a
whole.
