> [!WARNING]
> 
> **File Browser is archived on 2026-09-01**. The last planned release has already shipped. There will be no further releases, bug fixes, or security fixes.   

<p align="center">
  <img src="./branding/banner.png" width="550"/>
</p>

File Browser provides a file managing interface within a specified directory and it can be used to upload, delete, preview and edit your files. It is a **create-your-own-cloud**-kind of software where you can just install it on your server, direct it to a path and access your files through a nice web interface.

**Background:** [Goodbye File Browser, for Real This Time](https://hacdias.com/2026/07/28/filebrowser/), July 2026.

## Security

Published advisories are listed under [security advisories](https://github.com/filebrowser/filebrowser/security/advisories),
and reporting instructions are in [SECURITY.md](SECURITY.md). Upstream flagged two known issue
classes as unaddressed. In this fork:

- **Command execution, runner, and hooks.** Eliminated on 2026-09-16: the hook runner and interactive
  shell were removed from the codebase entirely rather than left disabled-by-default, since this
  feature was plagued with vulnerabilities across many published advisories and upstream said it
  would need a full rewrite to be made safe. There is no flag to re-enable it. Background:
  [#5199](https://github.com/filebrowser/filebrowser/issues/5199), removal details in
  [`docs/command-execution.md`](docs/command-execution.md).
- **Session and JWT handling.** Still present, tracked as an accepted residual risk with compensating
  controls. Sessions are self-contained JWTs rather than server-side identifiers, so they cannot be
  revoked, which means that logout, password changes, and renewal leave previously issued tokens
  valid until they expire, and the same refresh token can be redeemed repeatedly. Assume a leaked
  token is valid until expiry. Background: [#5216](https://github.com/filebrowser/filebrowser/issues/5216).

If you run File Browser:

- **Do not expose it directly to the internet.** Put it behind a reverse proxy that terminates TLS and performs its own authentication.
- **Run it unprivileged, inside a container**, with only the directory you intend to serve mounted into it.

## Documentation

Documentation on how to install, configure, and build this project lives in [`docs`](docs) in this repository.

[CONTRIBUTING.md](CONTRIBUTING.md) documents how to build and develop the project, which remains useful to anyone forking it.

## License

[Apache License 2.0](LICENSE) © File Browser Contributors
