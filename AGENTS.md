# Agent Instructions

## Toolchain

- Reuse the prepared common toolchain; do not install/reinstall host tools or SDK/NDK components without approval.
- Do not run environment setup scripts, create new setup scripts, or persist shell configuration changes without approval.
- `setup-environment.sh` imports the latest upstream `termux/termux-packages` infrastructure independently of fork updates and can change the required NDK version; read the current requirement from `scripts/properties.sh`, and do not refresh infrastructure or override NDK version variables without approval.
- Use TUR's `build-package.sh`; do not assemble packages manually or build outside its work directories.
- Resolve the package name and architecture from the current task; do not assume either.

## Commands

| Task | Command |
| --- | --- |
| Recipe syntax | `bash -n <component>/<package>/build.sh` |
| Patch whitespace | `git diff --check` |
| Force rebuild | `./build-package.sh -f -a <arch> <package>` |

- Replace placeholders with the selected recipe and architecture.
- Use `-s` only when skipping dependency checks is justified; it is not a general build default.
- Keep `TERMUX_INSTALL_DEPS` unset or false unless downloading/extracting target dependencies is explicitly approved. Recipe-local build dependencies are handled by the selected recipe.

### Before every build: mount and verify

1. Inspect `scripts/properties.sh` and `scripts/build/termux_step_setup_toolchain.sh` to resolve `NDK` and `TERMUX_STANDALONE_TOOLCHAIN` using the selected architecture, API level, and cache revision; do not hardcode these paths or invoke the full-copy setup logic to discover them.
2. Verify the preserved `-upper` directory contains `.termux-standalone-toolchain`; if absent, stop and request proper initialization. Never fabricate the marker.
3. With those variables resolved, run in Bash:

```bash
: "${NDK:?Resolve NDK first}" "${TERMUX_STANDALONE_TOOLCHAIN:?Resolve toolchain path first}"
tc="$TERMUX_STANDALONE_TOOLCHAIN"
test -f "$tc-upper/.termux-standalone-toolchain" || exit 1
if ! mountpoint -q "$tc"; then
    fuse-overlayfs -o "lowerdir=$NDK/toolchains/llvm/prebuilt/linux-x86_64,upperdir=$tc-upper,workdir=$tc-work" "$tc" || exit 1
fi
mountpoint -q "$tc" && test -f "$tc/.termux-standalone-toolchain" || exit 1
```

### After every build: unmount

- Once no build or process is using the toolchain, leave its directory and run `fusermount3 -u "$TERMUX_STANDALONE_TOOLCHAIN"` in the same environment. Preserve `-upper` and `-work`.

## Key Conventions

- Do not edit or commit ignored, imported build infrastructure; see `.gitignore`.
- Keep package recipes and source patches in the selected component/package directory; leave separate upstream checkouts unchanged.
- Never symlink or hardlink the writable toolchain cache to the original NDK, or recursively delete a mounted overlay.
- Inspect build logs for unexpected downloads, duplicate installations, and full NDK copies; stop rather than bypassing missing prerequisites.
- Keep artifacts in ignored `output/`; distinguish host smoke tests from Android/device tests.
- Do not commit or push without an explicit request.
- Use matching PR titles and commit subjects: `addpkg(<component>/<package>): <version>` for additions, `bump(<component>/<package>): <version>` for updates, and `fix(<component>/<package>): <short description>` for recipe fixes.
- When amending a new-package PR to fix CI, retain its `addpkg(...)` title and commit subject.

## External References

| Need | File |
| --- | --- |
| Contribution and build guide | `README.md` |
| Package recipe template | `sample/build.sh` |
| Overlay/full-copy behavior | `common-files/building-system-patches/0008-disable-apparmor.patch` |
