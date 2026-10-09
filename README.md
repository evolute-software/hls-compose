# hls-compose

LSP bridge that runs haskell-language-server (HLS) inside the
docker compose dev service that owns the project you are editing,
and pipes the LSP stream over stdio to your editor on the host.

The host never needs a Haskell toolchain: the editor talks to this
script, the script talks to HLS inside the container. File URIs
match one-to-one because the dev service mounts the sources at the
identical host path inside the container.

Repository: https://github.com/evolute-software/hls-compose

## Behavior

The project root is taken from the `rootUri` (or the first
`workspaceFolders` entry) of the client's first LSP message, the
`initialize` request, which the script reads from stdin before
doing anything else. LSP clients cannot be relied on to start the
server with the project root as the working directory: coc.nvim,
for example, spawns language servers with the editor's cwd. If the
request carries neither, the script falls back to its own cwd.
From the project root the script:

1. Walks up to the nearest compose file (`docker-compose.yml`,
   `docker-compose.yaml`, `compose.yml` or `compose.yaml`).
2. Picks the compose service to use, in this order:
   - `HLS_COMPOSE_SERVICE` environment variable (explicit
     override, always wins);
   - a service named `<project-dir>-dev`;
   - a bind-mount scan of the resolved compose config: the service
     whose bind mount source is the project directory, or covers
     it (a mount of an ancestor directory also works).
3. Runs `docker compose up -d <service>` so the service is up
   (diagnostics go to stderr).
4. Execs `haskell-language-server-wrapper --lsp` in the service
   with the working directory set to the project root, replays the
   buffered `initialize` request, and streams the rest of stdin.

The buffered message is replayed byte-exact (headers and body), so
the bridge is transparent to the client.

stdout carries only the LSP stream. All script and docker
diagnostics go to stderr, which most editors show in their LSP
log.

If discovery finds nothing, the script exits with an error that
names the compose file and points to the contract below.

## The contract your compose project must satisfy

- A dev service (any name) that bind-mounts the project source
  tree at the identical absolute host path:

  ```yaml
  volumes:
    - ${HOME}/path/to/project:${HOME}/path/to/project
  ```

- `haskell-language-server-wrapper` reachable on PATH inside the
  service's image. With the official `haskell:<version>` images
  there is no ghcup; HLS comes from the upstream release tarball,
  which ships its own GNUmakefile install target:

  ```dockerfile
  ADD https://github.com/haskell/haskell-language-server/releases/download/<ver>/haskell-language-server-<ver>-x86_64-linux-<distro>.tar.xz /tmp/hls.tar.xz
  RUN mkdir -p /opt/hls \
   && tar -xJf /tmp/hls.tar.xz -C /opt/hls --strip-components=1 \
   && rm /tmp/hls.tar.xz \
   && make -C /opt/hls install
  ```

  Pick the prebuilt that matches your image distro (e.g. `deb12`
  for Debian bookworm) and supports your GHC version.

- Recommended: put the dev service behind a dedicated compose
  profile (e.g. `dev`) so it does not autostart with the stack.
  The script always passes `--profile dev` to compose, and starts
  the chosen service by name, which works regardless of profile.

- Recommended: run the service as the same uid:gid as the host
  user, so build artifacts the toolchain writes into the mounted
  tree (`dist-newstyle` for cabal) stay host-owned. Build the
  dependency store in the image as that same user to avoid a
  chown layer.

If your service does not follow the `<project-dir>-dev` naming,
set `HLS_COMPOSE_SERVICE` in the environment the editor starts
the bridge in.

## Install

Check the repository out somewhere stable and make sure the script
is executable (git preserves the mode, so a clone usually needs
nothing):

```
git clone https://github.com/evolute-software/hls-compose.git \
  ~/.vim/hls-compose
```

### coc.nvim

In `~/.vim/coc-settings.json`:

```json
"languageserver": {
  "haskell": {
    "command": "/home/YOU/.vim/hls-compose/hls-compose",
    "args": [],
    "filetypes": ["haskell", "lhaskell"],
    "rootPatterns": ["*.cabal", "cabal.project"]
  }
}
```

Adjust `command` to wherever you cloned the repository. The
`rootPatterns` matter less now that the project root comes from
`rootUri`, but keep them so coc resolves the right workspace root
for the request.

### Other LSP clients

Any client that can spawn a server command works: point it at the
script with no arguments. The project root is taken from the
client's own `initialize` request, so the client's cwd does not
matter.

## Notes

- HLS cold start can take a minute or more on a busy host; if your
  editor's init timeout is short, a retry usually connects fast
  because caches are warm.
- Requirements: docker and docker compose v2 on the host. Nothing
  else; no host Haskell toolchain is used.
