# NEATO
(Norms Expressed As a Tool for Operating systems)

NEATO is a standard that defines an for Neetcomputers operating systems and their method of booting. This is mostly because the encourages making your own operating systems, and while I think we can all agree that is very sick, it does create a problem of application compatibility.

The standards are not meant to be super strict rules, but rather a small shim over the already existing functions that Neetcomputers offers, with basic things added most applications would need, such as terminal emulation or standards for how arguments should work. 

Of course, an OS can comply with these rules however it likes to, as long as the environment is the same for the application. If an OS wants to have a visual interface rather than a terminal emulator, it can make apps that use the graphics API pop up as a window, for example, instead of overwriting the global environment.

API definitions, aka stuff to do with the application's environment, like functions or events, go in the `api` folder.

Definitions about the boot process, aka stuff about how OSes are to be booted (which is useful for bootloaders or bioses), go in the `boot` folder.
