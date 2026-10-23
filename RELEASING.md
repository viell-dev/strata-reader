# Releasing strata-reader

This procedure applies the
[Rust library verification recipe](https://github.com/viell-dev/strata-template/blob/main/ci/rust-library/README.md).
For contributor checks, see [CONTRIBUTING.md](CONTRIBUTING.md).

## 1. Prepare the release commit

In a pull request, as for any other change:

- Set `version` in `Cargo.toml`. If the release needs a newer compiler, raise `rust-version` too
  and note it in the changelog and the README.
- If `Cargo.lock` is tracked, run `cargo update --workspace` and include its changes in the
  release commit.
- Give the changelog entry its version, date, and link.
- Add anything new that should ship to the `include` list in `Cargo.toml`.

## 2. Run the gate

Check out the pull request's head commit and confirm that `git status --porcelain` prints
nothing; never use `--allow-dirty`. Plain `cargo` uses the development compiler that
`rust-toolchain.toml` selects. All three stages must pass. If CI has already run the first two
on this exact commit and passed, cite that run instead of repeating them.

### Fast checks

```bash
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-targets --all-features
cargo test --doc --all-features
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps --all-features
```

### Minimum supported Rust version

With a toolchain of exactly the version `rust-version` declares:

```bash
msrv=<rust-version>
rustup toolchain install "$msrv" --profile minimal
cargo +"$msrv" test --all-targets --all-features
cargo +"$msrv" test --doc --all-features
```

### Package

```bash
cargo package --list
cargo package
(
  cd target/package/strata-reader-<version> &&
  cargo test --all-targets --all-features &&
  cargo test --doc --all-features
)
```

The listing must contain every repository-local file the packaged README, changelog, and licensing
documents link to, plus every test, example, or benchmark meant to ship. If it does not, fix the
`include` list in a new commit and run the gate again. The tests in the extracted copy check the
shipped source, including the benchmark smoke cases; keep separate doctests because
`--all-targets` excludes them.

Report the results in the pull request description: the commit, the host, the compiler versions,
the test counts per stage, and every ignored test with its reason.

## 3. Merge and verify the final commit

Complete the repository's review requirements and merge using a permitted method. A true
fast-forward preserves the gated commit; GitHub's Rebase and merge creates new commit SHAs, as do
squash and merge-commit operations. Do not bypass branch protection to preserve a SHA.

Check out the resulting commit on `main` and compare its full SHA with the gated SHA. If it
changed, rerun all three stages of the gate on that resulting commit and record the new SHA and
results on the release pull request. If a stage fails, fix the problem through a new pull request
and repeat this process. A gate on an earlier PR head does not verify the merged commit.

## 4. Tag and publish

From a clean repository root, on the exact final commit verified above:

```bash
git tag <version>
git push origin <version>
cargo publish
```

Tags are the bare version, without a `v` prefix. Then:

- Check that the registry shows the intended version, README, license, and `rust-version`, and
  that docs.rs builds it.
- Create the forge release from the tag with the changelog entry as its body.
- Update anything that reports release status, such as an ecosystem hub, in its own repository.
