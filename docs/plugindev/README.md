# Tanzu CLI Plugin Implementation Guide

## Introduction

The Tanzu CLI was built to be extensible across teams and cohesive across
SKUs. To this end, the Tanzu CLI provides tools to make creating and compiling
new plugins straightforward.

Before embarking on developing a plugin, the developer should be familiar with
the following materials:

1. The key concepts related to the Tanzu CLI
2. The [CLI plugin architecture](../full/cli-architecture.md#plugins)
3. The [Tanzu CLI Styleguide](style_guide.md), which describes the user
interaction best practices to be followed. It is highly recommended that anyone
interested in developing a Tanzu CLI plugin be familiarized with the
recommendations.
4. The [build stage](build_stage_styleguide_checklist.md) and [design stage](design_stage_styleguide_checklist.md) style guide checklists are useful resources to refer to maximize UX consistency in the plugin being developed.

This document will primarily focus on setting up a development environment to
build and publish plugins.

### Note to developers with existing plugin projects

For plugin projects implemented based on the legacy Tanzu CLI codebase, some minor
adjustments will have to be made to account for the new code dependencies and
plugin publishing process. See the [transition guide](migrate.md) for more
details.

## Environment

The [Tanzu Plugin Runtime](https://github.com/vmware-tanzu/tanzu-plugin-runtime)
(also referred to as "runtime library" or simply "runtime")
is a library the plugin implementation should integrate with to
implement the [plugin contract](contract.md).

The Tanzu Core CLI and runtime library are written in the Go programming
language. While the plugin architecture technically does not prevent the
development of plugins using other programming languages, at the moment
the only supported means of plugin development is that which integrates with
the released version of the runtime library.

The minimum supported Go version is 1.18.

You will need Docker to be installed on your development system.

------------------------------

### Starting a plugin project

Some CLI functionality essential for plugin development are available as Tanzu
CLI plugins. These are termed "admin plugins" for the rest of the document.

The easiest way to bootstrap a new plugin project is to make use of the
`builder` admin plugin. This plugin provides commands to construct scaffolding
for plugins and plugin commands along with removing the need to write boilerplate
code. Use one of the following method to install the builder plugin.

#### Installing the official release of the builder plugin

```console
tanzu plugin install builder
```

For more details on the builder plugin, run `tanzu builder --help`.

### Bootstrapping a plugin project

#### 1) create a new plugin repository

```shell
tanzu builder init <repo-name>
```

either specify the `--repo-type` or be prompted to choose between GitLab or
GitHub type repository. The choice will determine the type of skeleton CI
configuration file generated.

#### 2) add the main package

```shell
cd <repo-name> && tanzu builder cli add-plugin <plugin-name>
```

will add a `main` package for the new plugin. You should now adjust the
newly created `main` package to implement the functionality of your new plugin.

#### 3) update plugin metadata

You will notice in the generated `main.go` file, that CLI plugins have to instantiate a
[Plugin Descriptor](https://github.com/vmware-tanzu/tanzu-plugin-runtime/blob/main/plugin/types.go#L60)

``` go
import (
  "github.com/vmware-tanzu/tanzu-plugin-runtime/plugin/buildinfo"
  "github.com/vmware-tanzu/tanzu-plugin-runtime/plugin"
)

var descriptor = plugin.PluginDescriptor{
    Name:         "helloworld",
    Description:  "Hello world plugin",
    Target:       types.TargetUnknown, // <<<FIXME! set the Target of the plugin to one of {TargetGlobal,TargetK8s,TargetTMC}
    Version:      buildinfo.Version,
    BuildSHA:     buildinfo.SHA,
    Group:        plugin.ManageCmdGroup, // set group
}

func main() {
    p, err := plugin.NewPlugin(&descriptor)
    //...
}
```

#### 4) commit the changes to the repository

Create an initial commit.

```shell
git add -A
git commit -m "Initialize plugin repository"

git tag v0.0.1 # TAG the repository if it has no tag.
```

#### 5) Setting up the go modules

```shell
# Download and configure modules and update the go.mod and go.sum files
make gomod

git add -A
git commit -m "Configure go.mod and go.sum"
```

### Building a Plugin

At this point, the source repository does not yet have any specific commands
implemented, but yet a fully functional plugin (with some common
commands included) should already be buildable.

The `builder` plugin also provides functionality to build, install or publish
the plugins. These capabilities are most easily accessed through Makefile
targets already set up for the same purposes. All plugin-related tooling has been
added using the `plugin-tooling.mk` file.

#### Build the plugin binary for local install

```sh
# Building all plugins within the repository
make plugin-build-local

# Building only a single plugin
make plugin-build-local PLUGIN_NAME="<plugin-name>"

# Building multiple plugins at a time
make plugin-build-local PLUGIN_NAME="{plugin-name-1,plugin-name-2}"
```

This will build plugin artifacts under `./artifacts` with plugins organized under: `artifacts/plugins/<OS>/<ARCH>/<TARGET>`

```sh
# Installing all plugins from a local source using the makefile target
make plugin-install-local
# Installing plugins from local source using the tanzu-cli command
tanzu plugin install --local ./artifacts/plugins/${HOSTOS}/${HOSTARCH} [pluginname|all]
```

Users can also use the below make target to build and install plugins at once.

```sh
# Combined Build and Install target is also available
make plugin-build-install-local
```

Your plugin is now available for you through the Tanzu CLI. You can confirm
this by running `tanzu plugin list` which will now show your plugin.

Plugins are installed into `$XDG_DATA_HOME`, (read more about the XDG Base Directory Specification [here.](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)

#### Build the plugin binary for publishing

```sh
# Building all plugins within the repository for all required os-arch combination
make plugin-build

# Building only a single plugin for all required os-arch combination
make plugin-build PLUGIN_NAME="<plugin-name>"

# Building multiple plugins at a time for all required os-arch combination
make plugin-build PLUGIN_NAME="{plugin-name-1,plugin-name-2}"
```

The `builder` plugin v1.0.0 or greater, supports generating the correct `plugin_manifest.yaml` and
`plugin_bundle.tar.gz` when the user runs `make plugin-build PLUGIN_NAME="<plugin-name>"` individually
multiple times to build different plugins possibly with different plugin versions. For example:

```sh
# Build `foo`. This will generate `plugin_bundle.tar.gz` containing just `foo:v1.0.0` plugin
make plugin-build PLUGIN_NAME="foo" PLUGIN_BUILD_VERSION=v1.0.0

# Build `bar`. This will generate `plugin_bundle.tar.gz` containing `foo:v1.0.0` and `bar:v2.0.0`
make plugin-build PLUGIN_NAME="bar" PLUGIN_BUILD_VERSION=v2.0.0

# Build `baz`. This will generate `plugin_bundle.tar.gz` containing `foo:v1.0.0`, `bar:v2.0.0`, `baz:v3.0.0`
make plugin-build PLUGIN_NAME="baz" PLUGIN_BUILD_VERSION=v3.0.0
```

**Note:** Because the builder plugin will automatically create merged plugin bundle if the `plugin_manifest.yaml` already exists,
For each new plugin build to replace any previous available bundle and start fresh, please configure the environment variable `PLUGIN_BUNDLE_OVERWRITE=true`

The next steps are to write the plugin code to implement what the plugin is meant to do.

#### Adding plugin commands

The scaffolded code creates a Plugin object to which additional sub [cobra.Command](https://pkg.go.dev/github.com/spf13/cobra#Command) can be added.

#### Tests

Every CLI plugin should have a nested test executable. The executable should
utilize the test framework found in `pkg/v1/test/cli`.

Tests are written to ensure the stability of the commands and are compiled
alongside the plugins. Tests can be run by the admin `test` plugin of the Tanzu
CLI.

#### Docs

Since every plugin is required a `README.md` document that explains its basic
usage, a basic one is generated in the top-level directory of the repository as
well.

Edit the file as appropriate.

#### Optimizations, debugging and build flags

By default the `builder` plugin `v1.2.0` or later optimizes build flags to
reduce the binary size of the plugin being built.  Two optimizations are done:
building without debug symbols (which reduces the binary size by up to 30%),
and deactivating function inlining (which reduces the binary size by about 6%).

Building without debug symbols has the side-effect of preventing the use of a
debugger towards the built binary.  However, when using an IDE, a different
binary is used, one built by the IDE, and therefore the debugger can be used
directly from the IDE.

If you require using a debugger directly on a plugin binary, you have to build
a binary that includes the debug symbols.  You can build such a binary by using
`PLUGIN_ENABLE_DEBUG=1` along with your `make` command.

The `builder` plugin deactivates function inlining by implicitly injecting
the `-gcflags=all=-l` go flags.  Be aware that if you use the `PLUGIN_GO_FLAGS`
variable to inject other `-gcflags`, you should pay attention to also include
the `all=-l` to keep the optimization. If for some reason you need to activate
function inlining (at the cost of a larger binary), you can do so by
using `PLUGIN_GO_FLAGS=-gcflags=all=` (without the `-l` specified), which will
have the effect of replacing the go flags specified by the `builder` plugin.

### Publishing a plugin

To publish one or more built plugins to a target repository, one would need to

1. Have built each plugin for all required os/architecture combinations (`make plugin-build`)
1. Specify the repository with the env var PLUGIN_PUBLISH_REPOSITORY (the default location is localhost:5001/test/v1/tanzu-cli/plugins), which is where the local test repository is deployed
1. Have push access to said repository location

```sh
make plugin-build
make plugin-publish-packages
```

one can also combine the building and publishing of the plugins with

```sh
make plugin-build-and-publish-packages
```

### Updating the plugin inventory of the plugin repository

```sh
# Initialize empty SQLite database
make inventory-init
# Add plugin entry to the SQLite database
make inventory-plugin-add
```

### Creating a plugin group

A Plugin groups define a list of plugin/version combinations that are applicable together. They facilitate an efficient installation of these plugins. Below are instructions on how to create and publish a plugin group.

#### Creating plugin group when all plugins are build together from same repository

Create a `plugin-scope-association.yaml` file under `cmd/plugin/plugin-scope-association.yaml` of your plugin repository.

Sample file content

```yaml
plugins:
- name: builder
  target: global
  isContextScoped: false
- name: test
  target: global
  isContextScoped: false
```

Update the `PLUGIN_SCOPE_ASSOCIATION_FILE` variable in the `plugin-tooling.mk` file to point to the created association file:

``` makefile
PLUGIN_SCOPE_ASSOCIATION_FILE ?= $(ROOT_DIR)/cmd/plugin/plugin-scope-association.yaml
```

Whenever you change the `plugin-scope-association.yaml`, the plugins must be rebuilt to generate the proper plugin group information.

``` sh
make plugin-build
```

#### Creating plugin group manually when plugins are published separately and group is created with combination of different plugins

Sample `plugin-group-manifest.yaml` file content

```yaml
plugins:
- name: builder
  target: global
  isContextScoped: false
  version: v1.0.0
- name: test
  target: global
  isContextScoped: false
  version: v1.2.0
```

Note: Starting with `v1.1.0` version of Tanzu CLI, it supports providing shortened
version (`vMAJOR` or `vMAJOR.MINOR`) as part of the plugin version.
Using shortened version as above, will install the latest available minor.patch of
`vMAJOR` and latest patch version of `vMAJOR.MINOR` respectively.

Example: If the plugin group is defined like below installing the plugin group will
install latest minor.patch of `v1` for `builder` plugin and latest patch
of `v1.2` for the `test` plugin.

```yaml
- name: builder
  target: global
  isContextScoped: false
  version: v1
- name: test
  target: global
  isContextScoped: false
  version: v1.2
```

#### Adding plugin group to the inventory database

Run the below command to add a plugin group with the respective plugins:

```sh
make inventory-plugin-group-add PLUGIN_GROUP_NAME_VERSION=name:version PLUGIN_GROUP_DESCRIPTION="description"
```

name:version format ```{vendor-publisher/plugin_group_name}:vX.X.X```

Examples

- vmware-tanzucli/essentials:v1.0.0
- vmware-tkg/default:v2.3.0
- vmware-tap/default:v1.6.1

will update the inventory database image at the plugin repository with the new plugin entries

### Using the published plugins

Assuming the default location was used in the publishing step.

The following CLI configuration can be set to instruct the CLI to query it

```sh
tanzu config set env.TANZU_CLI_ADDITIONAL_PLUGIN_DISCOVERY_IMAGES_TEST_ONLY localhost:5001/test/v1/tanzu-cli/plugins/plugin-inventory:latest
tanzu config set env.TANZU_CLI_PLUGIN_DISCOVERY_IMAGE_SIGNATURE_VERIFICATION_SKIP_LIST localhost:5001/test/v1/tanzu-cli/plugins/plugin-inventory:latest
```

Once set, plugin lifecycle commands like `tanzu plugin search` will interact with the test repository as well.

Note: as the configuration variable implies, the
`TANZU_CLI_ADDITIONAL_PLUGIN_DISCOVERY_IMAGES_TEST_ONLY` setting is meant for
plugin development and testing only. The use of this setting in the production setting
is not supported.

Note: the `TANZU_CLI_PLUGIN_DISCOVERY_IMAGE_SIGNATURE_VERIFICATION_SKIP_LIST` configuration
variable instructs the CLI to skip the inventory database signature verification. This is
needed because the inventory database image in the local registry is not signed since it is
only used for testing. The signature verification should not be skipped for remote registries
as it would introduce a security risk.

### Testing the plugins

#### Manual testing

As a plugin developer, you will want to test your plugin manually at some point.
During the development cycle, you can easily build and install the latest version of your plugin
as described in the section on [building a plugin](#building-a-plugin).

While testing your plugin, interacting with a context that recommends an older version of said
plugin might result in your plugin version being overwritten.
In such a case, if you want to test your new version and prevent the CLI from overwriting it
with the version recommended by the context you can set the `TANZU_CLI_SKIP_CONTEXT_RECOMMENDED_PLUGIN_INSTALLATION`
variable by doing:

```sh
tanzu config set env.TANZU_CLI_SKIP_CONTEXT_RECOMMENDED_PLUGIN_INSTALLATION true
```

#### Automated tests using the test plugin

Plugin tests can be run by installing the admin `test` plugin.
Currently, we only support testing plugins built locally.

**Note:** The `test` admin functionality has been deprecated and no future enhancements are planned for this plugin.

Steps to test a plugin:

1. Bootstrap a new plugin
2. Build a plugin binary
3. Run below command

``` go
tanzu test fetch -l ~/${PLUGIN_NAME}/artifacts/plugins/${HOSTOS}/${HOSTARCH}
tanzu test plugin PLUGIN_NAME
```

Example: `helloworld` plugin

``` go
tanzu test fetch -l ~/helloworld/artifacts/plugins/darwin/amd64

[i] Installing plugin 'helloworld:v0.0.1' with target 'global'
[i] Installing test plugin for 'helloworld:v0.0.1'

❯ tanzu test plugin helloworld
---
[i] testing helloworld


[i] cleaning up

[ok] ok: successfully tested helloworld
```

------------------------------

## CLI command best practices

### Components

CLI commands should, to the extent possible, utilize the
[Plugin UX component library](https://github.com/vmware-tanzu/tanzu-plugin-runtime/tree/main/component)
for interactive features like prompts or table printing to achieve consistency
across different plugin usage.

### Asynchronous Requests

Commands should be written in such a way as to return as quickly as possible.
When a request is not expected to return immediately, as is often the case with
declarative commands, the command should return immediately with an exit code
indicating the server's response.

The completion notice should include an example of the `get` command the user
would need to poll the resource to check the state/status of the
operation.

### Shell Completion

Shell completion (or "command-completion" or "tab completion") is the ability
for the program to automatically fill in partially typed commands, arguments,
flags, and flag values. The Tanzu CLI provides an integrated solution for shell
completion which will automatically take care of completing commands and flags
for your plugin. To make the completions richer, a plugin can add logic to
also provide shell completion for its arguments and flag values; these are
referred to as "custom completions".

The Tanzu CLI provides such custom completions for all its core command arguments
and flag values. To allow for the best user-experience, it is valuable for each
plugin to also provide similar completions for its own command arguments and flag values.

To find examples of custom completions, please refer to implementations of `ValidArgsFunction`
and calls to `RegisterFlagCompletionFunc()` in the
[Tanzu CLI repository](https://github.com/vmware-tanzu/tanzu-cli).

For detailed documentation including troubleshooting and debugging, please refer to the
Cobra project's documentation on
[customizing completions](https://github.com/spf13/cobra/blob/main/site/content/completions/_index.md#customizing-completions)
to learn how to make your plugin more user-friendly using shell completion.

#### ActiveHelp as part of Shell Completion

ActiveHelp are messages printed through shell completion as the program is being used.
The Tanzu CLI provides ActiveHelp in different situations to improve the user-experience.
Plugins are encouraged to make use of ActiveHelp to enhance their own user-experience.

To better understand how ActiveHelp can be used, please refer to calls to
`cobra.AppendActiveHelp()` in the
[Tanzu CLI repository](https://github.com/vmware-tanzu/tanzu-cli).

For detailed documentation including troubleshooting and debugging, as well as configuration,
please refer to the Cobra project's documentation on
[ActiveHelp](https://github.com/spf13/cobra/blob/main/site/content/active_help.md)
to learn how to make your plugin more user-friendly using ActiveHelp.

### Configuration file

The tanzu configuration files reside in
XDG_DATA_HOME/.config/tanzu, which will typically be `~/.config/tanzu/` on most systems.

(See [XDG](https://github.com/adrg/xdg/blob/master/README.md) for more details)

For more details on the APIs available to retrieve or set various CLI configuration settings, refer to
[Plugin UX component library](https://github.com/vmware-tanzu/tanzu-plugin-runtime/tree/main/docs/config.md)

### Other states kept on the CLI machine

Besides `XDG_DATA_HOME/.config/tanzu`, the following directories are also used
to exclusively store data and artifacts specific to Tanzu CLI:

- [XDG](https://github.com/adrg/xdg/blob/master/README.md)_DATA_HOME/tanzu_cli: where plugins are installed
- _your home directory_/.cache/tanzu: contains the catalog recording information about installed plugins, as well as plugin inventory information.

Cleaning out the above-mentioned directories should restore the CLI to a pristine state.

### Plugin environment variables

When the Tanzu CLI executes a plugin, it passes the outer environment to the
plugin, and also injects some additional environment variables.

Currently, the only additional environment variable passed to the plugin is
`TANZU_BIN` which provides the path to the `tanzu` command (as executed by the user).
If a plugin needs to trigger a Tanzu CLI operation, it can do so by externally calling
the `tanzu` binary specified by the `TANZU_BIN` variable.

For example, the plugin can use the following to perform a `tanzu plugin sync`:

``` go
cliPath := os.Getenv("TANZU_BIN")
command := exec.Command(cliPath, "plugin", "sync")
command.Stdout = os.Stdout
command.Stderr = os.Stderr
command.Run()
```

## Deprecation of existing plugin functionality

It is highly recommended that plugin authors follow the same process used by
the Core CLI to announce and implement deprecation of specific plugin
functionality.

For more details on the deprecation policy and process please refer to the
[Deprecation document](../dev/deprecation.md).

## Plugin Descriptor and support for command remapping

The output of the plugin binary's `info` command reflects what is specified by
in the plugin's [Plugin Descriptor](https://github.com/vmware-tanzu/tanzu-plugin-runtime/blob/main/plugin/types.go#L60)

### CommandMap

EXPERIMENTAL: subject to change prior to the next official minor release

The descriptor now supports an additional field `commandMap` that customizes
how plugin's commands are shown in the CLI's command tree. Most plugin authors
should not need to provide the mapping fields.

The command map specifies one or more CommandMapEntry's and describes how one
or more parts of the plugin's command tree will be remapped in the Tanzu CLI

Each CommandMapEntry has the following fields

- SourceCommandPath: is a space-delimited path to the command relative to the root Command of this plugin, with the root Command's path being ""
- DestinationCommandPath: is a space-delimited path to the command relative to the root Command of the Tanzu CLI that the command associated with SourceCommandPath should be mapped to and invocable by the user
- Overrides: By default, the command previously situated at the DestinationCommandPath of the Tanzu CLI, if exist, will be the one overridden by this entry. If this mapping attempt in intended to override another part of the Tanzu CLI command tree, the override path should be used.
- Description: Required when remapping a subcommand of this plugin outside of the plugin's command tree (e.g. whe elevating a plugin's subcommand to a top level command of the Tanzu CLI). This enables the CLI to provide better help information about the remapped command.

Assume there is a plugin named foo with a non empty CommandMap

### Rename the plugin command group for plugin

```go
        plugin.CommandMapEntry{
            SourceCommandPath:      "",
            DestinationCommandPath: "bar",
        },
```

instead of every plugin command X be invocable via `tanzu foo X`, it is now
invocable via `tanzu bar X`

### Elevate subcommand to toplevel of Tanzu CLI command tree

```go
        plugin.CommandMapEntry{
            SourceCommandPath:      "echo",
            DestinationCommandPath: "echo",
            Description:            "the echo command elevated to the top level",
        },
```

instead of invoking `tanzu foo echo`, the command is now invocable via `tanzu echo`

Note: if the intention is to "move" not "copy" the `tanzu foo echo` command,
authors are advised to mark the echo Command as 'Hidden: true'

### Hiding original plugin command group

```go
        plugin.CommandMapEntry{
            SourceCommandPath:      "",
            DestinationCommandPath: "",
        },
```

All valid mappings requires that destination path should have a valid parent
command in the Tanzu CLI command tree.
As a special case, specifying both the source and destination command path as
"" indicates that the CLI command mapping to the root command of the plugin
will NOT be created. This could be useful when all the plugin's command have been
elevated (see previous section) and nothing interesting should be invoked via
`tanzu foo ...` anymore.

### Reorganizing the plugin commands under a different category group for plugin

```go
        plugin.CommandMapEntry{
            SourceCommandPath:      "",
            DestinationCommandPath: "operations foo",
        },
```

instead of every plugin command X be invocable via `tanzu foo X`, it is now
invocable via `tanzu operations foo X`. Note that there is a requirement that
the command group a plugin is mapping under ("operations" in the example
provided) has to be already present in the CLI.

### SupportedContextType

EXPERIMENTAL: subject to change prior to the next official minor release

In some rare situations plugins might elect to conditionally activate based on
the type of the CLI context that is currently active.

SupportedContextType specifies one or more ContextType that this plugin will specifically apply to.
When no context of matching type is active, the command tree specified by this plugin should be omitted.
When unset, the plugin does not define any specific opinions on this aspect.

Note that plugins omitted from the CLI command tree due to the
SupportedContextType condition appears with a false value for the ACTIVE column
in the `tanzu plugin list --wide` output.

### Communicating mapped command invocation to the plugin

When command mapping is involved, details about how a TANZU CLI command is invoked
will be communicated to the plugin via environment variables.

- TANZU_CLI_INVOKED_COMMAND: the name of the mapped CLI command
- TANZU_CLI_INVOKED_GROUP: the path to the command group under which the TANZU_CLI_INVOKED_COMMAND is found
- TANZU_CLI_COMMAND_MAPPED_FROM: the command path (relative to the plugin) that a mapped CLI command is mapped to.

Plugin can retrieve the above mentioned information by accessing corresponding
fields in the result of the GetInvocationContext() functiion.
