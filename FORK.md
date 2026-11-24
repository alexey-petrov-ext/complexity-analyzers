# ComplexityAnalysis.Analyzers: patch branch for QUINDOS

This repository carries changes to
[complexity-analyzers](https://github.com/rodri-oliveira-dev/complexity-analyzers) that QUINDOS
needs, until they are merged upstream. It is not a separate product. Every change is meant for
upstream, except the fork-only files listed below.

## Branches

| Branch | Content |
|---|---|
| `main` | Mirror of `rodri-oliveira-dev/complexity-analyzers` `main`. Never commit here. |
| `upstream/<topic>` | One upstream pull request per branch, based on `main`. Each one has its regression tests, keeps the gates of `AGENTS.md`, and has nothing fork-specific. |
| `quindos` | `main` + the commits of all `upstream/*` branches + fork-only commits. Releases are tagged here. |

### Changes waiting for upstream

| Branch | Change | Upstream issue / PR |
|---|---|---|
| `upstream/anonymous-type-identity` | A lambda that reads a member of an anonymous type no longer stops `ComplexityAnalyzer` with `AD0001` ("Value must not be empty. (Parameter 'metadataName')"). `KnownOperationResolver` gives no mapping for a member of an anonymous type or an anonymous delegate type, so the estimate stays `Unknown`. Based on `main` at `8058882`. In `v1.6.1-quindos.1`. | PR [#149](https://github.com/rodri-oliveira-dev/complexity-analyzers/pull/149), for issue [#148](https://github.com/rodri-oliveira-dev/complexity-analyzers/issues/148) |

When a pull request merges, remove its branch from this table and from the update steps below.

### Fork-only files

- `FORK.md` (this file).

## Updating from upstream

```bash
git remote add upstream https://github.com/rodri-oliveira-dev/complexity-analyzers.git   # once
git remote set-url --push upstream DISABLED-no-push                                         # once
git fetch upstream
old_main=$(git rev-parse main)
git checkout main && git merge --ff-only upstream/main
for b in upstream/anonymous-type-identity; do
  git rebase main "$b"
done
git rebase --onto main "$old_main" quindos   # replays topic commits + fork-only commits
git push --force-with-lease origin main 'upstream/*' quindos
```

- If git reports a commit as already upstream during the rebase, drop it. When a topic is merged
  upstream, delete its branch and remove its row above.
- Consumers pin release packages (version + SHA-256), not branch commits, so rewriting `quindos`
  does not affect them.
- Exit: when no patches are left, pin the NuGet.org release again and archive this repository.

## Releases

Tag `quindos` with `v<VersionPrefix>-quindos.<n>`, for example `v1.6.1-quindos.1`, where
`VersionPrefix` is the value of `Directory.Build.props`. Then build and pack the analyzer with the
same version:

```bash
dotnet build ComplexityAnalysis.Analyzers.slnx --configuration Release
dotnet pack src/ComplexityAnalysis.Analyzers/ComplexityAnalysis.Analyzers.csproj --configuration Release --no-build -p:PackageVersion=1.6.1-quindos.1 --output artifacts/local-packages
sha256sum artifacts/local-packages/ComplexityAnalysis.Analyzers.1.6.1-quindos.1.nupkg
```

Attach the `.nupkg` and its SHA-256 to the GitHub release of the tag.

The release workflows of upstream start only by `workflow_dispatch`, so a pushed tag starts no
run. The fork has no secrets and no `release` environment, so a run would publish nothing to
NuGet.org. Never start them on this fork.

## Checking a change

The gates are the ones of `AGENTS.md`:

```bash
dotnet build ComplexityAnalysis.Analyzers.slnx --configuration Release
dotnet test ComplexityAnalysis.Analyzers.slnx --configuration Release --no-build
```

`global.json` names the .NET SDK. On a computer with an older SDK of the same band, change
`global.json` for the local run only, and never commit that change. The test
`Repository_build_time_contract_matches_the_documented_baseline` then fails, and that one failure
is expected. `analyzer-ci.yml` runs on each push, also on a topic branch, with the SDK of
`global.json`.
