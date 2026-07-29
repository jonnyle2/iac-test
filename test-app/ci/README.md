# Concourse image build and promotion

The two pipelines deliberately separate artifact creation from production
promotion:

```text
Git commit
   |
   v
build OCI image -> Trivy vulnerability gate -> push :dev + :git-<commit>
                                                   |
PR/hotfix merges into main ------------------------+
                                                   |
                                                   v
                                      resolve matching git-<SHA>
                                                   |
                                                   v
                                      copy OCI layout -> push :prod
```

The production pipeline never rebuilds the image. A Git resource processes
every qualifying commit that lands on `main`. For a normal two-parent merge,
the pipeline selects the second parent SHA—the head commit of the merged PR or
hotfix. For a single-parent result such as fast-forward, squash, or rebase, it
selects the resulting `main` commit SHA. It resolves `git-<selected SHA>` to a
digest and downloads that exact digest as an OCI layout. The production `put`
step publishes that layout as `prod`, as the semantic release version, and as
the same `git-<SHA>` tag.

The build pipeline must publish the selected SHA-tagged candidate. Two-parent
merges can reuse a pre-merge branch image. Squash and rebase strategies generate
new commit SHAs, so configure the build pipeline to build the resulting `main`
commit after the merge. The production job waits briefly for that candidate and
fails safely without promoting another image if it never appears.

## Variables and secrets

Edit the non-secret values in:

- `dev/vars.yaml`
- `prod/vars.yaml`

The pipeline expects these secrets from a Concourse credential manager:

```yaml
git:
  username: ...
  password: ...
harbor:
  username: ...
  password: ...
```

For a public source repository, remove `username` and `password` from the git
resource instead of supplying unused credentials.

If the registry uses a private CA, add `ca_certs` to both `registry-image`
resource sources and resolve its value from the credential manager. Do not set
`insecure: true` except for an isolated local test registry.

## Install the pipelines

From the repository root:

```sh
fly -t TARGET set-pipeline \
  -p test-app-dev \
  -c iac/test-app/ci/dev/pipeline.yaml \
  -l iac/test-app/ci/dev/vars.yaml

fly -t TARGET set-pipeline \
  -p test-app-prod \
  -c iac/test-app/ci/prod/pipeline.yaml \
  -l iac/test-app/ci/prod/vars.yaml

fly -t TARGET unpause-pipeline -p test-app-dev
fly -t TARGET unpause-pipeline -p test-app-prod
```

The dev job starts on a matching Git change. The production job triggers
automatically for qualifying changes to `main`. Set repository branch
protection so that direct pushes are rejected and only approved pull requests
or hotfixes can update the branch. A Git resource observes branch changes; it
cannot distinguish a permitted merge from a direct push by itself.

## Security behavior

`trivy_severity` defaults to `HIGH,CRITICAL`, and `--exit-code 1` prevents the
registry push when matching vulnerabilities are found. Trivy must be able to
download its vulnerability database; in an air-gapped environment, mirror the
Trivy databases and configure the scan task to use those mirrors.

The moving `dev` and `prod` tags are convenient deployment aliases. For
rollbacks and audit records, use the immutable `git-<full-commit-sha>` tag
published by the dev pipeline or deploy by digest.
