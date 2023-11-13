# The BuildKit Cache Dance

Workaround for Buildx/BuildKit's [lack of support](https://github.com/moby/buildkit/issues/1512) for
[the dedicated `RUN` caches](https://docs.docker.com/build/cache/#use-the-dedicated-run-cache) when building Docker
images in GitHub Actions.

```yaml
---
name: Build

on:
  push:
    branches: [main]
    tags: [v*]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        platform: [linux/amd64, linux/arm64]

    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v2
        with:
          cleanup: false
          platforms: ${{ matrix.platform }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: YOUR_IMAGE
          tags: |
            type=edge
            type=ref,event=tag
            type=sha,format=long

      - name: Inject Docker Build(x|Kit) cache mounts
        uses: sid-maddy/buildkit-cache-dance/inject@main
        with:
          cache-mounts: |
            cargo-registry
            rust-target-release
          github-token: ${{ secrets.GITHUB_TOKEN }}
          key: rust-buildkit-cache-${{ matrix.platform }}-${{ hashFiles('Cargo.toml', 'Cargo.lock') }}
          restore-keys: |
            rust-buildkit-cache-${{ matrix.platform }}-

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          builder: ${{ steps.buildx.outputs.name }}
          cache-from: type=gha,scope=${{ matrix.platform }}
          cache-to: type=gha,mode=max,scope=${{ matrix.platform }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: ${{ matrix.platform }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}

      - name: Extract Docker Build(x|Kit) cache mounts
        uses: sid-maddy/buildkit-cache-dance/extract@main
        with:
          cache-mounts: |
            cargo-registry
            rust-target-release
          github-token: ${{ secrets.GITHUB_TOKEN }}
          key: rust-buildkit-cache-${{ matrix.platform }}-${{ hashFiles('Cargo.toml', 'Cargo.lock') }}
```

## Credits

- [Alexander Pravdin](https://github.com/speller), for the basic idea in
  [this comment](https://github.com/moby/buildkit/issues/1512#issuecomment-1319736671).
- <https://github.com/overmindtech/buildkit-cache-dance> and <https://github.com/dcginfra/buildkit-cache-dance>, for
  laying the foundations.
