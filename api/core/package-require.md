# NEATO `package` and `require` API Specification

Extension: `core`

Version: 4

---

A NEATO compliant environment must provide a function under the name `_G.require`, along with other facilities exported
in the global `package` table.

## `require(modname) --> result, loaderdata`

Loads the given module. The function starts by looking into the `package.loaded` table to determine whether modname is
already loaded. If it is, then require returns the value stored at `package.loaded[modname]`. (The absence of a second
result in this case signals that this call did not have to load the module.) Otherwise, it tries to find a loader for the
module.

To find a loader, require is guided by the table package.searchers. Each item in this table is a search function, that
searches for the module in a particular way. By changing this table, we can change how require looks for a module. The
following explanation is based on the default configuration for `package.searchers`.

First `require` queries `package.preload[modname]`. If it has a value, this value (which must be a function) is the loader.
Otherwise it searches for a Lua loader using the path stored in `package.path`.

Once a loader is found, `require` calls the loader with two arguments: `modname` and an extra value, a loader data, also
returned by the searcher. The loader data can be any value useful to the module; for the default searchers, it indicates where
the loader was found. (For instance, if the loader came from a file, this extra value is the file path.) If the loader returns
any non-nil value, require assigns the returned value to `package.loaded[modname]`. If the loader does not return a non-nil
value and has not assigned any value to `package.loaded[modname]`, then require assigns true to this entry. In any case,
require returns the final value of `package.loaded[modname]`. Besides that value, require also returns as a second result the
loader data returned by the searcher, which indicates how require found the module.

If there is any error loading or running the module, or if it cannot find any loader for the module, then `require` raises an
error.

## Entries of global `package` table

### `package.config`

A string describing some compile-time configurations for packages. This string is a sequence of lines:

    The first line is the directory separator string. Default is `/` for NEATO paths.
    The second line is the character that separates templates in a path. Default is ';'.
    The third line is the string that marks the substitution points in a template. Default is '?'.

The rest exist in Lua, but are omitted in NEATO because they are irrelevant for NEET Computers.

### `package.loaded`

A table used by `require` to control which modules are already loaded. When you require a module `modname` and `package.loaded
[modname]` is not false, `require` simply returns the value stored there.

This variable is only a shallow copy of the real table; assignments to this variable do not change the table used by require.

### `package.path`

A string representing the `;`-separated (or whatever separator is defined in `package.config` list of path templates to
look for modules in. Each path template is a NEATO path with a `?` character (or whatever substituation character is defined
in `package.config`) being replaced with whatever the `modname` string was passed to `require`. For example, if the path
template is `0:system:/lib/?/init.lua;0:system:/lib/?.lua` and the `modname` is `module`, `require` will look for the module
in `0:system:/lib/module/init.lua` and `0:system:/lib/module.lua`.

### `package.searchers`

A table used by require to control how to find modules.

Each entry in this table is a searcher function. When looking for a module, `require` calls each of these searchers in ascending
order, with the module name (the argument given to `require`) as its sole argument. If the searcher finds the module, it returns
another function, the module loader, plus an extra value, a loader data, that will be passed to that loader and returned as a
second result by `require`. If it cannot find the module, it returns a string explaining why (or nil if it has nothing to say).

NEATO defines this table with two searcher functions, but an OS can add more.

The first searcher simply looks for a loader in the package.preload table.

The second searcher looks for a loader as a Lua library, using the path stored at package.path. The search is done as described
in function package.searchpath.

The second searcher returns as the extra value the file path where the module was found, as returned by package.searchpath. The
first searcher always returns the string ":preload:".

Searchers should raise no errors and have no side effects.

### `package.searchpath (name, path [, sep [, rep]])`

Searches for the given `name` in the given `path`.

A path is a string containing a sequence of templates separated by semicolons. For each template, the function replaces each
interrogation mark (if any) in the template with a copy of name wherein all occurrences of `sep` (a dot, by default) were replaced
by `rep` (the system's directory separator, a front slash by default), and then tries to open the resulting file name.

For instance, if the path is the string

     "./?.lua;./?.lc;/usr/local/?/init.lua"

the search for the name foo.a will try to open the files ./foo/a.lua, ./foo/a.lc, and /usr/local/foo/a/init.lua, in that order.

Returns the resulting name of the first file that it can open in read mode (after closing the file), or fail plus an error
message if none succeeds. (This error message lists all file names it tried to open.)
