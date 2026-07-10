# NEATO `sys` API Specification

Written by UsUsStudios July 10th, 2026
Draft 1
Last updated in NEATO version alpha

---

A NEATO-compliant operating system must expose a global API in `_G.sys` to retrieve system information about the running operating system, computer and system that is not retrievable from the `chip` API.
Below is a list of functions that the `sys` API must expose:

|      name      |                                                                 description                                                                 |         arguments         |             returns             |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------|---------------------------------|
| getOSName      | returns the official name of the active operating system that exposed the `sys` API. This should stay exactly the same between versions.    |                           | the OS name (str)               |
| getOsVer       | returns the version of the active operating system in whatever format the OS uses.                                                          |                           | the OS version (str)            |
| getNeatoCompat | returns the latest released NEATO version that the OS is fully compatible with, or 0 if the OS is incompatible with any released version.   |                           | the NEATO version number (int)  |
