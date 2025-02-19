# Meta
[meta]: #meta
- Name: `.profile` Utility Buildpack
- Start Date: 2022-01-25
- Author(s): mboldt
- Status: Approved
- RFC Pull Request: [rfcs#200](https://github.com/buildpacks/rfcs/pull/200)
- CNB Pull Request: (leave blank)
- CNB Issue: N/A
- Supersedes: N/A

# Summary
[summary]: #summary

As part of RFC 93, [`.profile` scripts will cease to be supported by the platform API](https://github.com/buildpacks/rfcs/blob/main/text/0093-remove-shell-processes.md#appprofile).
This RFC proposes developing a [utility buildpack](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md) to support `.profile` scripts to prevent regressions after RFC 93 is implemented.

# Definitions
[definitions]: #definitions

*utility buildpack*: A buildpack officially supported by the Buildpack Authors' Tooling Team per [RFC 97](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md) that provides a generic capability.

# Motivation
[motivation]: #motivation

[RFC 93](https://github.com/buildpacks/rfcs/blob/main/text/0093-remove-shell-processes.md) resolves to remove shell-specific logic from the CNB Specification.
Part of this includes removing support for `.profile` script in a future version of the Platform API.
RFC 93 recommends supporting the `.profile` script functionality in a utility buildpack to avoid regressions.
This proposal is to develop and support the `.profile` utility buildpack, allowing RFC 93 to be implemented without regression.

# What it is
[what-it-is]: #what-it-is

The target persona is a platform operator or implementor who wants to update to the latest platform API, while maintaining the `.profile` functionality for application developers.

We propose developing and supporting a buildpack to provide an identical interface to the existing `.profile` functionality.
It will:

- Detect a `.profile` file in the app dir
- Wrap the `.profile` file so that it implements the `exec.d` interface
- Add the `exec.d` executable to the `<layer>/exec.d` directory so the launcher will apply it

We propose the ID `buildpacksio/profile` for this buildpack (cf. [RFC 97](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md#what-it-is)).

## Example 1 (environment variables)

Here is an example of a `.profile` script, inspired by paketo-buildpacks/node-engine:

```
memory_in_bytes="$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes)"
MEMORY_AVAILABLE="$(( $memory_in_bytes / ( 1024 * 1024 ) ))"
export MEMORY_AVAILABLE
```

The wrapper should ensure that the `MEMORY_AVAILABLE` environment variable is set in the environment with the proper value.

## Example 2 (file system side effects)

With this `.profile` script:

```
echo 'hello' >> "$HOME/hello"
```

The wrapper would not need to set any environment variables, but should maintain the side effect of creating the `$HOME/hello` file.


# How it Works
[how-it-works]: #how-it-works

On Linux, the wrapper will:

1. Ensure `bash` exists on the run image, and error if it does not.
1. Launch a `bash` shell to:
  1. Source the `.profile` script, which may set environment variables and/or have other side effects.
  1. Write the environment variables to the [`exec.d` output TOML](https://github.com/buildpacks/spec/blob/main/buildpack.md#execd-output-toml).

Likewise on Windows, the wrapper will:

1. Ensure `cmd.exe` exists on the run image, and error if it does not.
1. Launch a `cmd.exe` shell to:
  1. Source the `.profile.bat` script, which may set environment variables and/or have other side effects.
  1. Write the environment variables to the [`exec.d` output TOML](https://github.com/buildpacks/spec/blob/main/buildpack.md#execd-output-toml).

Sourcing the `.profile`/`.profile.bat` script will execute any side effects.
Writing the `exec.d` output TOML will ensure that all environment variables will be set.
So, this will solve for both of the simple examples above.

Per the [Operating System Conventions in the CNB spec](https://github.com/buildpacks/spec#operating-system-conventions), this buildpack will support scripts compatible with bash version 3 or greater on Linux, and cmd.exe on Windows.

# Migration
[migration]: #migration

This buildpack is new, so has no inherent migration considerations.

For the related migration concerns for removing shell functionality in general, see the [Migration Path section of RFC 93](https://github.com/buildpacks/rfcs/blob/main/text/0093-remove-shell-processes.md#migration-path).

# Drawbacks
[drawbacks]: #drawbacks

This is a new component to maintain and support.
That being said, this will enable spec and lifecycle simplifications noted in [RFC 93](https://github.com/buildpacks/rfcs/blob/main/text/0093-remove-shell-processes.md) (where this approach was first suggested).
Also, [RFC 97](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md) resolves to support and maintain utility buildpacks.

# Alternatives
[alternatives]: #alternatives

If we do nothing, we introduce a regression in functionality, and force application developers to rework their `.profile` scripts.


# Prior Art
[prior-art]: #prior-art

- Current CNB spec:
  - `.profile` scripts are part of the buildpack app interface ([spec](https://github.com/buildpacks/spec/blob/main/buildpack.md#app-interface))
  - `.profile.d/` scripts are part of the buildpack build phase output ([spec](https://github.com/buildpacks/spec/blob/main/buildpack.md#build)).

- Profile scripts:
  - Heroku supports [`.profile` scripts](https://devcenter.heroku.com/articles/dynos#the-profile-file) and [`.profile.d/` scripts](https://devcenter.heroku.com/articles/buildpack-api#profile-d-scripts).
  - Cloud Foundry supports [`.profile and .profile.d/ scripts](https://docs.cloudfoundry.org/devguide/deploy-apps/deploy-app.html#profile).

- Prior related RFCs:
  - [Officially Supported Utility Buildpacks](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md)
  - [Remove Shell Processes](https://github.com/buildpacks/rfcs/blob/main/text/0093-remove-shell-processes.md)
  - [Exec.d - Shell-Free Profile.d](https://github.com/buildpacks/rfcs/blob/main/text/0057-exec.d-shell-free-profile-d.md)

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- **Are there things that `.profile` scripts do that will not be covered by the exec.d interface?**
  For example, defining environment variables and side effects like writing files should be supported.
  But, something like defining a bash function will not be supported.

  This is definitely an edge case.
  The app profile runs after the buildpack provided profiles and just before the process type is executed so the only thing that could depend on one of the functions in the process.
  This will almost certainly never happen with a buildpack-provided process (the buildpack depending on the app to define a bash function would be very weird and brittle).

  If the user provides the process definition (e.g. with a procfile) this is slightly more plausible.
  But in this case there is a very easy workaround - source a script containing the functions as part of the command provided in the procfile.

- **What if the run image does not contain bash (e.g., `FROM scratch`)?**
  Should it error, no-op, should this buildpack add a layer with bash?

  Note that the [utility buildpacks RFC](https://github.com/buildpacks/rfcs/blob/main/text/0097-official-utility-buildpacks.md#how-it-works) states:
  > The buildpack MUST NOT install any dependencies in the output image
  which precludes this buildpack (or any CNB utility buildpack) from adding bash to the run image.

  While one can imagine a buildpack to add bash, we would like to see the community move to more secure run images that don't include bash.

  We will document that this buildpack requires bash to be in the run image, and this buildpack will not add bash.
  If bash is not in the run image, the exec.d wrapper will error out.

- **How should we implement the buildpack?**
  Should it be a bash script?
  A Go binary (or two)?
  What should we use for CI, frameworks, etc.?

  We will leave these implementation details to the Buildpack Authors' Tooling sub-team.
  We can finalize these details after the RFC is approved.
  We have captured some initial ideas for the [project plumbing](https://github.com/mboldt/utility-buildpack-plumbing) and had some discussion about [implementation details](https://github.com/buildpacks/rfcs/pull/200#discussion_r828063918) to seed those conversations.

# Spec. Changes
[spec-changes]: #spec-changes

None.
