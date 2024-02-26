# `namespace-actions/upload-artifact`

Namespace's version of Upload [Actions Artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts) compatible with the [upload-artifact](https://github.com/actions/upload-artifact).

The artifacts are stored in Namespace internal storage. They will not be visible and billed in GitHub.

Download artifacts with [download-artifact](https://github.com/namespace-actions/download-artifact).

## Usage

### Inputs

```yaml
- uses: namespace-actions/upload-artifact@v0
  with:
    # Name of the artifact to upload.
    # Optional. Default is 'artifact'
    name:

    # A file, directory or wildcard pattern that describes what to upload
    # Required.
    path:

    # The desired behavior if no files are found using the provided path.
    # Available Options:
    #   warn: Output a warning but do not fail the action
    #   error: Fail the action with an error message
    #   ignore: Do not output any warnings or errors, the action does not fail
    # Optional. Default is 'warn'
    if-no-files-found:

    # Duration after which artifact will expire in days. 0 means using default retention.
    # Minimum 1 day.
    # Maximum 90 days unless changed from the repository settings page.
    # Optional. Defaults to repository settings.
    retention-days:

    # The level of compression for Zlib to be applied to the artifact archive.
    # The value can range from 0 to 9.
    # For large files that are not easily compressed, a value of 0 is recommended for significantly faster uploads.
    # Optional. Default is '6'
    compression-level:

    # If true, an artifact with a matching name will be deleted before a new one is uploaded.
    # If false, the action will fail if an artifact for the given name already exists.
    # Does not fail if the artifact does not exist.
    # Optional. Default is 'false'
    overwrite:
```

### Outputs

| Name | Description | Example |
| - | - | - |
| `artifact-id` | GitHub ID of an Artifact, can be used by the REST API | `1234` |
| `artifact-url` | URL to download an Artifact. Can be used in many scenarios such as linking to artifacts in issues or pull requests. Users must be logged-in in order for this URL to work. This URL is valid as long as the artifact has not expired or the artifact, run or repository have not been deleted | `https://github.com/example-org/example-repo/actions/runs/1/artifacts/1234` |

## Examples

### Upload an Individual File

```yaml
steps:
- run: mkdir -p path/to/artifact
- run: echo hello > path/to/artifact/world.txt
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: path/to/artifact/world.txt
```

### Upload an Entire Directory

```yaml
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: path/to/artifact/ # or path/to/artifact
```

### Upload using a Wildcard Pattern

```yaml
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: path/**/[abc]rtifac?/*
```

### Upload using Multiple Paths and Exclusions

```yaml
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: |
      path/output/bin/
      path/output/test-results
      !path/**/*.tmp
```

For supported wildcards along with behavior and documentation, see [@actions/glob](https://github.com/actions/toolkit/tree/main/packages/glob) which is used internally to search for files.

If a wildcard pattern is used, the path hierarchy will be preserved after the first wildcard pattern:

```
path/to/*/directory/foo?.txt =>
    ∟ path/to/some/directory/foo1.txt
    ∟ path/to/some/directory/foo2.txt
    ∟ path/to/other/directory/foo1.txt

would be flattened and uploaded as =>
    ∟ some/directory/foo1.txt
    ∟ some/directory/foo2.txt
    ∟ other/directory/foo1.txt
```

If multiple paths are provided as input, the least common ancestor of all the search paths will be used as the root directory of the artifact. Exclude paths do not affect the directory structure.

Relative and absolute file paths are both allowed. Relative paths are rooted against the current working directory. Paths that begin with a wildcard character should be quoted to avoid being interpreted as YAML aliases.

### Altering compressions level (speed v. size)

If you are uploading large or easily compressable data to your artifact, you may benefit from tweaking the compression level. By default, the compression level is `6`, the same as GNU Gzip.

The value can range from 0 to 9:
  - 0: No compression
  - 1: Best speed
  - 6: Default compression (same as GNU Gzip)
  - 9: Best compression

Higher levels will result in better compression, but will take longer to complete.
For large files that are not easily compressed, a value of `0` is recommended for significantly faster uploads.

For instance, if you are uploading random binary data, you can save a lot of time by opting out of compression completely, since it won't benefit:

```yaml
- name: Make a 1GB random binary file
  run: |
    dd if=/dev/urandom of=my-1gb-file bs=1M count=1000
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: my-1gb-file
    compression-level: 0 # no compression
```

But, if you are uploading data that is easily compressed (like plaintext, code, etc) you can save space and cost by having a higher compression level. But this will be heavier on the CPU therefore slower to upload:

```yaml
- name: Make a file with a lot of repeated text
  run: |
    for i in {1..100000}; do echo -n 'foobar' >> foobar.txt; done
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: foobar.txt
    compression-level: 9 # maximum compression
```

### Customization if no files are found

If a path (or paths), result in no files being found for the artifact, the action will succeed but print out a warning. In certain scenarios it may be desirable to fail the action or suppress the warning. The `if-no-files-found` option allows you to customize the behavior of the action if no files are found:

```yaml
- uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: path/to/artifact/
    if-no-files-found: error # 'warn' or 'ignore' are also available, defaults to `warn`
```

### (Not) Uploading to the same artifact

Uploading to the same artifact via multiple jobs is _not_ supported.

```yaml
- run: echo hi > world.txt
- uses: namespace-actions/upload-artifact@v0
  with:
    # implicitly named as 'artifact'
    path: world.txt

- run: echo howdy > extra-file.txt
- uses: namespace-actions/upload-artifact@v0
  with:
    # also implicitly named as 'artifact', will fail here!
    path: extra-file.txt
```

Artifact names must be unique since each created artifact is idempotent so multiple jobs cannot modify the same artifact.

In matrix scenarios, be careful to not accidentally upload to the same artifact, or else you will encounter conflict errors. It would be best to name the artifact _with_ a prefix or suffix from the matrix:

```yaml
jobs:
  upload:
    name: Generate Build Artifacts

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        version: [a, b, c]

    runs-on: ${{ matrix.os }}

    steps:
    - name: Build
      run: ./some-script --version=${{ matrix.version }} > my-binary
    - name: Upload
      uses: namespace-actions/upload-artifact@v0
      with:
        name: binary-${{ matrix.os }}-${{ matrix.version }}
        path: my-binary
```

This will result in artifacts like: `binary-ubuntu-latest-a`, `binary-windows-latest-b`, and so on.

Previously the behavior _allowed_ for the artifact names to be the same which resulted in unexpected mutations and accidental corruption. Artifacts created by upload-artifact@v4 are immutable.

### Environment Variables and Tilde Expansion

You can use `~` in the path input as a substitute for `$HOME`. Basic tilde expansion is supported:

```yaml
  - run: |
      mkdir -p ~/new/artifact
      echo hello > ~/new/artifact/world.txt
  - uses: namespace-actions/upload-artifact@v0
    with:
      name: my-artifacts
      path: ~/new/**/*
```

Environment variables along with context expressions can also be used for input. For documentation see [context and expression syntax](https://help.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions):

```yaml
    env:
      name: my-artifact
    steps:
    - run: |
        mkdir -p ${{ github.workspace }}/artifact
        echo hello > ${{ github.workspace }}/artifact/world.txt
    - uses: namespace-actions/upload-artifact@v0
      with:
        name: ${{ env.name }}-name
        path: ${{ github.workspace }}/artifact/**/*
```

For environment variables created in other steps, make sure to use the `env` expression syntax

```yaml
    steps:
    - run: |
        mkdir testing
        echo "This is a file to upload" > testing/file.txt
        echo "artifactPath=testing/file.txt" >> $GITHUB_ENV
    - uses: namespace-actions/upload-artifact@v0
      with:
        name: artifact
        path: ${{ env.artifactPath }} # this will resolve to testing/file.txt at runtime
```

### Retention Period

Artifacts are retained for 90 days by default. You can specify a shorter retention period using the `retention-days` input:

```yaml
  - name: Create a file
    run: echo "I won't live long" > my_file.txt

  - name: Upload Artifact
    uses: namespace-actions/upload-artifact@v0
    with:
      name: my-artifact
      path: my_file.txt
      retention-days: 5
```

The retention period must be between 1 and 90 inclusive. For more information see [artifact and log retention policies](https://docs.github.com/en/free-pro-team@latest/actions/reference/usage-limits-billing-and-administration#artifact-and-log-retention-policy).

### Using Outputs

If an artifact upload is successful then an `artifact-id` output is available. This ID is a unique identifier that can be used with [Artifact REST APIs](https://docs.github.com/en/rest/actions/artifacts).

#### Example output between steps

```yml
    - uses: namespace-actions/upload-artifact@v0
      id: artifact-upload-step
      with:
        name: my-artifact
        path: path/to/artifact/content/

    - name: Output artifact ID
      run:  echo 'Artifact ID is ${{ steps.artifact-upload-step.outputs.artifact-id }}'
```

#### Example output between jobs

```yml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      output1: ${{ steps.artifact-upload-step.outputs.artifact-id }}
    steps:
      - uses: namespace-actions/upload-artifact@v0
        id: artifact-upload-step
        with:
          name: my-artifact
          path: path/to/artifact/content/
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - env:
          OUTPUT1: ${{needs.job1.outputs.output1}}
        run: echo "Artifact ID from previous job is $OUTPUT1"
```

### Overwriting an Artifact

Although it's not possible to mutate an Artifact, can completely overwrite one. But do note that this will give the Artifact a new ID, the previous one will no longer exist:

```yaml
jobs:
  upload:
    runs-on: ubuntu-latest
    steps:
      - name: Create a file
        run: echo "hello world" > my-file.txt
      - name: Upload Artifact
        uses: namespace-actions/upload-artifact@v0
        with:
          name: my-artifact # NOTE: same artifact name
          path: my-file.txt
  upload-again:
    needs: upload
    runs-on: ubuntu-latest
    steps:
      - name: Create a different file
        run: echo "goodbye world" > my-file.txt
      - name: Upload Artifact
        uses: namespace-actions/upload-artifact@v0
        with:
          name: my-artifact # NOTE: same artifact name
          path: my-file.txt
          overwrite: true
```

## Limitations

### Zip archives

When an Artifact is uploaded, all the files are assembled into an immutable Zip archive. There is currently no way to download artifacts in a format other than a Zip or to download individual artifact contents.

### Permission Loss

File permissions are not maintained during artifact upload. All directories will have `755` and all files will have `644`. For example, if you make a file executable using `chmod` and then upload that file, post-download the file is no longer guaranteed to be set as an executable.

If you must preserve permissions, you can `tar` all of your files together before artifact upload. Post download, the `tar` file will maintain file permissions and case sensitivity.

```yaml
- name: 'Tar files'
  run: tar -cvf my_files.tar /path/to/my/directory

- name: 'Upload Artifact'
  uses: namespace-actions/upload-artifact@v0
  with:
    name: my-artifact
    path: my_files.tar
```
