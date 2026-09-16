# Command Execution

> [!NOTE]
>
> The **hook runner** and **interactive shell** functionalities have been **removed entirely** in
> this fork (they no longer exist in the code, rather than merely being disabled). Upstream carried
> them disabled by default from v2.33.8 onwards due to continuous, unresolved security
> vulnerabilities (see [#5199](https://github.com/filebrowser/filebrowser/issues/5199)), and this
> project does not need the feature, so it was removed instead of kept as a re-enableable risk.
> There is no configuration flag, CLI subcommand, or API endpoint left for it; re-adding it would
> require restoring the removed code.
>
> This does not affect **Hook Authentication** (`auth.method=hook`), which is a separate feature
> for delegating login decisions to an external command — see [`authentication.md`](authentication.md).
