# `grcov` Action

![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)
[![Gitter](https://badges.gitter.im/actions-rs/community.svg)](https://gitter.im/actions-rs/community)
![Experimental status](https://img.shields.io/badge/status-experimental-yellow.svg)

This GitHub Action collects and aggregates code coverage data with the
[grcov](https://github.com/mozilla/grcov) tool.

## Example workflow

```yaml
on: [push]

name: Code Coverage

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: nightly
          override: true
      - uses: actions-rs/cargo@v1
        with:
          command: test
          args: --all-features --no-fail-fast
        env:
          CARGO_INCREMENTAL: '0'
          RUSTFLAGS: '-Zprofile -Ccodegen-units=1 -Cinline-threshold=0 -Clink-dead-code -Coverflow-checks=off -Zno-landing-pads'
      - uses: actions-rs/grcov@v0.1
```

## Usage

1. As `grcov` works with coverage data generated by the unstable [`-Z profile` feature](https://github.com/rust-lang/rust/issues/42524),
    you need to install `nightly` toolchain, for example,
    with the [`actions-rs/toolchain`](https://github.com/actions-rs/toolchain) Action:
    
    ```yaml
    - uses: actions-rs/toolchain@v1
      with:
        toolchain: nightly
        override: true
    ```

2. It is strongly recommended to call `cargo clean` command
    if any of `cargo check`, `cargo build`, `cargo test`
    or other similar commands were executed already in this workspace.

    ```yaml
    - uses: actions-rs/cargo@v1
      with:
        command: clean
    ```

3. Create a configuration file for `grcov`, see [config section](#config) for details.

4. Execute the `cargo test` command.
    It is required to add `CARGO_INCREMENTAL` and `RUSTFLAGS` environment variables
    for this command, see [grcov](https://github.com/mozilla/grcov) page for details.

    ```yaml
    - uses: actions-rs/cargo@v1
      with:
        command: test
        args: --all-features --no-fail-fast  # Customize args for your own needs
      env:
        CARGO_INCREMENTAL: '0'
        RUSTFLAGS: '-Zprofile -Ccodegen-units=1 -Cinline-threshold=0 -Clink-dead-code -Coverflow-checks=off -Zno-landing-pads'    
    ```

    Note that `-Clink-dead-code` flag might be broken for macOS environments,
    see [#63047](https://github.com/rust-lang/rust/issues/63047)

5. Add the `actions-rs@grcov` Action to the workflow.

    ```yaml
    - id: coverage  
      uses: actions-rs/grcov@v0.1
    ```

6. After the successful execution, `actions-rs@grcov`
    will set an Action output called `report`
    with an absolute path to the coverage report file.

    This file can be uploaded to any code coverage service,
    ex. with [codecov](https://github.com/marketplace/actions/codecov) or [coveralls](https://github.com/marketplace/actions/coveralls-github-action) Actions help:

    ```yaml
    - name: Coveralls upload
      uses: coverallsapp/github-action@master
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        path-to-lcov: ${{ steps.coverage.outputs.report }}
    ```

## Inputs

* `config`: Configuration file path (relative to repository root, default to `.github/actions-rs/grcov.yml`)

## Outputs

* `report`: Absolute path to the report file

## Config

`grcov` execution can be tuned with the configuration file.\
By default this Action tries to load it from the `.github/actions-rs/grcov.yml` path
(relative to the repository root directory); you can change file location with the [`config`](#inputs) input, ex.

```yaml
- uses: actions-rs/grcov@v0.1
  with:
    config: configs/grcov.yml
```

If configuration file is missing, default values will be used by the `grcov`.

All possible keys in the config are optional and directly mapped
to the [grcov](https://github.com/mozilla/grcov#usage) flags and options.\
Note that not all parameters can be changed via this file, see [example](#example) below
for config file format and available options.

### Example

```yaml
branch: true
ignore-not-existing: true
llvm: true
filter: covered
output-type: lcov
output-file: ./lcov.info
prefix-dir: /home/user/build/
ignore:
  - "/*"
  - "C:/*"
  - "../*"
path-mapping:
  - "/path1"
  - "/path2"
excl-br-line: "#\\[derive\\("
excl-br-start: "mod tests \\{"
excl-br-stop: "EXAMPLE"
excl-line: "#\\[derive\\("
excl-start: "mod tests \\{"
excl-stop: "EXAMPLE"
```

## Notes

1. [Coveralls](https://github.com/marketplace/actions/coveralls-github-action) Action is expecting `LCOV` format,
    do not use the `coveralls` or `coveralls+` output type for the `grcov`.\
    Instead, set the `output-type` config value to `lcov`.

2. Generated report file is stored in the temporary directory by default,
    which might not be accessible by the Docker-based Actions,
    such as [codecov](https://github.com/marketplace/actions/codecov).\
    Consider either mount it as [a Docker volume](https://help.github.com/en/articles/workflow-syntax-for-github-actions#jobsjob_idcontainervolumes)
    or use the `output-file` option in the [config](#config)
    to store report in the path accessible by container.
