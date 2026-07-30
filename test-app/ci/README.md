# Concourse image build and promotion

The three pipelines deliberately separate artifact creation from production
promotion:

```text
dev commit ─────→ dev pipeline ─────┐
                                    ├─→ :quarantine-<commit>
hotfix commit ──→ hotfix pipeline ──┘
                                              |
                                              v
                                    Harbor vulnerability scan
                                      ├─ pass → :git-<commit>
                                      └─ reject → remove quarantine tag
                                              |
normal merge into main                        |
        |                                     |
        v                                     |
Git tag vMAJOR.MINOR.PATCH                     |
        |                                     |
        v                                     |
production resolves the merged branch SHA ────┘
        |
        v
Harbor API adds immutable :MAJOR.MINOR.PATCH
```

The dev pipeline tracks `dev`; the separate hotfix pipeline tracks `hotfix`.
Both build into the same Harbor repository. Each build is initially pushed only
as `quarantine-<full-commit-sha>`, and the pipeline asks Harbor to scan that
artifact. A passing digest receives the immutable
`git-<full-commit-sha>` tag and loses its temporary quarantine tag. If Harbor
confirms a vulnerability at a blocked severity, the pipeline logs the report,
removes the quarantine tag, and fails without creating a production-eligible
Git-SHA tag.

The production pipeline never rebuilds the image and no longer stores release
numbers in a SemVer resource. It triggers only for Git tags matching
`vMAJOR.MINOR.PATCH`. It verifies that the tagged commit belongs to `main`,
removes the leading `v` for the Harbor tag, and then resolves the candidate
image. For a normal two-parent merge, it selects the second parent SHA—the head
commit built by either the dev or hotfix pipeline. For a single-parent result
such as fast-forward, squash, or rebase, it selects the tagged main commit SHA.

Production resolves `git-<selected SHA>` to a digest through Harbor's API
without downloading image layers. It adds the immutable semantic release tag,
then verifies that the Git-SHA and release tags point to the selected digest.

The appropriate build pipeline must publish the selected SHA-tagged candidate
before the release tag is created. Normal merge commits can reuse a pre-merge
dev or hotfix image. Squash and rebase strategies generate new commit SHAs, so
they require a build of the resulting `main` commit. Production waits briefly
for the candidate and fails safely if it does not exist.

## Create a production release

After the dev or hotfix image has passed its pipeline and the change has been
merged into `main`, create an annotated release tag on the main commit:

```sh
git checkout main
git pull --ff-only origin main
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3
```

Release tags and their corresponding Harbor semantic-version tags must be
immutable. A tag that does not match `vMAJOR.MINOR.PATCH`, or points to a commit
that is not on `main`, is not eligible for promotion.

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
  repository: harbor.example.com/project
  username: ...
  password: ...
```

For a public source repository, remove `username` and `password` from the git
resource instead of supplying unused credentials.

If Harbor uses a private CA, make the CA available to both the
`registry-image` resources and the Python Harbor API task image. Do not use
plain HTTP except for an isolated local test registry.

## Install the pipelines

From the repository root:

```sh
fly -t TARGET set-pipeline \
  -p test-app-dev \
  -c iac/test-app/ci/dev/pipeline.yaml \
  -l iac/test-app/ci/dev/vars.yaml

fly -t TARGET set-pipeline \
  -p test-app-hotfix \
  -c iac/test-app/ci/hotfix/pipeline.yaml \
  -l iac/test-app/ci/hotfix/vars.yaml

fly -t TARGET set-pipeline \
  -p test-app-prod \
  -c iac/test-app/ci/prod/pipeline.yaml \
  -l iac/test-app/ci/prod/vars.yaml

fly -t TARGET unpause-pipeline -p test-app-dev
fly -t TARGET unpause-pipeline -p test-app-hotfix
fly -t TARGET unpause-pipeline -p test-app-prod
```

The dev and hotfix jobs start on matching changes to their respective branches.
Production starts only when a matching release tag is pushed. Protect `main`
from direct pushes, and protect `v*` tags from deletion or reassignment.

## Security behavior

`harbor_blocked_severities` defaults to `HIGH,CRITICAL`. The pipeline explicitly
starts a Harbor vulnerability scan when one is not already queued or complete,
polls until it completes, and obtains the full report through Harbor's API.
Harbor must have a project or system scanner configured and its vulnerability
database must be available.

A confirmed blocked result removes `quarantine-<SHA>` and fails the job. Scanner
errors, unsupported results, API failures, missing severity counts, and timeouts
also fail the job, but retain the quarantine tag for diagnosis or retry. These
infrastructure failures never create `git-<SHA>`. Deleting the temporary tag
does not immediately reclaim blob storage; Harbor retention and garbage
collection should remove untagged rejected artifacts according to policy.

Only passing builds receive a published candidate tag. Deploy development and
hotfix builds by their `git-<full-commit-sha>` tag or digest, and deploy
production releases by their semantic-version tag or digest. Configure Harbor
tag immutability for `git-*` and semantic-version tags, but do not make
`quarantine-*` immutable because the pipeline must remove those temporary tags.
