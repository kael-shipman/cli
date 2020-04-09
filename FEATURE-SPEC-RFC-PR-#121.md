Feature Spec for RFC PR #121
============================================================================

*Implementation of RFC PR #121 - Add `link#[version](comment)` syntax to package.json*

There are four behaviors that need to change:

1. Running `npm install` and encountering a link-spec in the `package.json` file;
2. Running `npm update` and encountring a link-spec in the `package.json` file;
3. Running `npm install (--save(-dev)) link#[version-spec](comment)`;
4. Running `npm link (--save(-dev)) [package]@[version-spec](comment)`.

## Core Functionality

In all four cases, the core functionality will be executed:

```
// If the package is already installed in the global location
if (LinkDir.hasPackage(arg.name)) {
  // And it's at a compatible version
  if (SemVer.isCompatible(arg.version, LinkDir.package(arg.name).version)) {
    // Installed and compatible. Just skip to the linking part.

  // If it's not at a compatible version
  } else {
    // But we've indicated we want to force link it
    if (arg.force) {
      // Then proceed with a standard npm global install of the package at that version
      NpmInstallGlobal(arg.name, arg.version);

    // Otherwise, if we're not forcing it
    } else {
      // Throw an error
      throw new Error(
        `You've requested to link package ${arg.name} at version ${arg.version}, but that ` +
        `package already exists at ${LinkDir.path}/${arg.name} at incompatible version ` +
        `${LinkDir.package(arg.name).version}. Can't proceed.`
      );
    }
  }

// Otherise, if not yet installed globally, install it globally
} else {
  NpmInstallGlobal(arg.name, arg.version);
}

// Now the package is installed globally. We just need to link it in.

// If it's already installed here
if (LocalDir.hasPackage(arg.name)) {
  // And if it's already a link to the global dir
  if (LocalDir.package(arg.name).isLink()) {
    // Then there's nothing more to do

  // If it's not already a link
  } else {
    // Remove it and install a link
    LocalDir.removePackage(arg.name);
    LocalDir.linkPackageToGlobal(arg.name);
  }

// Otherwise, just link it to the global location
} else {
  LocalDir.linkPackageToGlobal(arg.name);
}
```
This logic results in either a package that has been successfully linked to a compatible package
in the global directory, or an error explaining what's going on.


## Editing of `package.json` File

### Case 1 - `npm install`

No changes need to be made when running `npm install` with no arguments.

### Case 2 - `npm update`

On successful completion of core logic execution on `npm update`, npm should revise the recorded
version spec in `package.json` to match the current version of the linked package.

For example, if the current version in `package.json` is `link#^3.1.5(github.com/me/my-project/tree/v3.x)`,
and the actual version of the linked package is `3.3.0`, npm should set the version in `package.json`
to be `link#^3.3.0(github.com/me/my-project/tree/v3.x)`

### Case 3 - `npm install --save(-dev) link#[version-spec](comment)`

On successful completion of core logic execution, npm should add the specified version spec to the
requested section of `package.json`. This is no different than it does today.

### Case 4 - `npm link (--save(-dev)) [package]@[version-spec](comment)`

On successful completion of core logic execution, npm should add the specified version spec to the
request section of `package.json`. This is new functionality, since previously linking information
was not saved.
