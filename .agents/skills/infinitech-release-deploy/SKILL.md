---
name: infinitech-release-deploy
description: Publish the current Infinitech project source as a GitHub version tag, then SSH to its LAN deployment server, update that checkout to the exact tag, and rebuild, restart, and verify the deployed services. Use only when the user asks to release or deploy this project.
metadata:
  short-description: Release Infinitech to GitHub and deploy over LAN SSH
---

# Infinitech release and deploy

Use this workflow for requests that ask to publish the current project changes and deploy them to the LAN server. The repository is `HCRXchenghong/infinitech2.0`; its root `package.json` version defines the platform release tag (`v<version>`). The project’s guarded publisher is `npm run release:github`, and its production Compose update is `npm run deploy -- up --config <private-config>`.

## Before publishing

1. Inspect the current repository root, branch, Git status, remotes, root `package.json`, release notes and `md/infra/deploy/README.md`. Read applicable `AGENTS.md` files. Do not silently publish unrelated or generated files.
2. Establish the exact release contents. The publisher copies source from the current workspace onto GitHub’s current default branch, so review its dry-run output and the workspace changes. Preserve the user’s intended edits. If the release version is already tagged, stop; never move or overwrite an existing tag. For a routine fix, bump the patch version; use a minor or major bump only when the requested change warrants it. Use `npm version patch --no-git-tag-version` (or `minor`/`major` as appropriate) to update the root manifest and lockfile together, then verify their versions match. Follow the repository’s release-note convention when adding notes.
3. Confirm the workspace has the required `gh` authentication and that the LAN server can be reached through an already configured SSH host/alias and trusted host key. Never guess credentials, SSH identities, hostnames, directories, or config paths. The deployment docs currently record `192.168.31.120` as the LAN host, but its address may have changed; prefer a matching SSH config alias and verify it resolves to the intended server. Ask for the missing connection settings if they cannot be discovered locally.
4. On the server, require an existing clone of this repository, the private deployment config (normally `infra/deploy/.env.production`), Docker Compose, Node/npm, and enough disk space to build. Keep private config and generated runtime material out of Git. Confirm the target checkout is clean before changing its revision; stop if it contains local edits or unrelated work.
5. Run the repository’s release dry-run and inspect the file summary. Run only the verification the user requested or the release documentation requires; do not claim checks that were not run.

## Publish exact version tag

Use the project publisher rather than creating refs manually. It reads the root version, refuses a pre-existing tag, requires `CONFIRM_GITHUB_RELEASE` and creates a GitHub commit, annotated `v<version>` tag, and GitHub Release. Pass `--tag` explicitly to both commands: the script’s fallback tag is a fixed historical version.

```bash
release_tag="v$(node -p "require('./package.json').version")"
confirmation="PUBLISH_$(printf '%s' "$release_tag" | sed 's/[^[:alnum:]]/_/g' | tr '[:lower:]' '[:upper:]')_TO_GITHUB"
npm run release:dry-run -- --tag "$release_tag"
CONFIRM_GITHUB_RELEASE="$confirmation" npm run release:github -- --tag "$release_tag"
```

The publisher may retry transient GitHub API failures a bounded number of times. If it reports an error after a partial remote mutation, inspect the remote branch, tag, and Release before retrying; do not rerun blindly.

After publishing, fetch `refs/tags/v<version>` from `origin` into the local repository and resolve `git rev-parse 'v<version>^{commit}'` to record its peeled commit SHA. Confirm it is the commit created by this release. Continue only with this exact immutable tag, not a moving branch head or a separately inferred “latest” tag.

## Update LAN server

Run commands over the verified SSH alias, using the server’s configured repository root and private config path. Pass paths and tag as safely quoted positional values; do not interpolate secrets into shell text or print private config contents. On the server:

1. Check `git status --porcelain` is empty. Fetch the exact tag from `origin`, verify the fetched tag’s commit SHA equals the GitHub SHA recorded above, and switch the checkout to detached `v<version>`. Verify the checked-out `package.json` version matches the tag.
2. Install dependencies from the lockfile (`npm ci`) when needed.
3. Run `npm run deploy -- preflight --config <private-config>`. Stop on failure. Preserve the current database and deployment config; never run reset, `down`, volume deletion, or force-push as part of an update.
4. Run `npm run deploy -- up --config <private-config>`. This project command builds the Compose stack from the checked-out source, runs pending API migrations before bringing up the new application stack, recreates the edge proxy and runs deployment health verification. Do not replace this with a plain `git pull` or ad-hoc `docker compose up`.
5. After `up` succeeds, run `npm run deploy -- refresh-deployed-revision --revision <full-commit-sha> --version <version-without-v> --config <private-config>`, then `npm run deploy -- mark-deployed-revision --revision <full-commit-sha> --version <version-without-v> --config <private-config>`. These commands record and verify the deployed revision using the project’s update-center contract.
6. Run `npm run deploy -- verify --config <private-config>` and inspect `npm run deploy -- status --config <private-config>`. Report the exact tag, commit SHA, health result, and any migration/deployment errors. A successful GitHub release alone does not mean deployment succeeded.

If SSH disconnects or a command fails during deployment, reconnect and inspect Git revision, Compose status, health checks, and the recorded deployed SHA before taking another action. Do not automatically roll back or rerun a partially completed deployment. Report the observed state and ask for direction if recovery requires an uncertain or destructive change.

## Completion report

Summarize the published tag and GitHub Release, the server and checkout path used, the deployed commit, and the service verification result. Clearly distinguish a release that succeeded from a deployment that failed or could not be reached. Never include credentials, private environment values, or secret output.
