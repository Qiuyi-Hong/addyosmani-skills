# One package for skills.sh and Codex

Research decision for [Verify one package for skills.sh and the Codex marketplace](https://github.com/Qiuyi-Hong/addyosmani-skills/issues/3), 1 October 2026. This is packaging research, not an implementation or a published release.

## Recommendation

Publish one canonical, flat `skills/<name>/` tree at the repository root. Put the adapted commands in that tree as ordinary skills. Add the portable schema to root `plugin.json` and use `addyosmani-skills` for both the plugin and marketplace identity. No npm package, MCP server, second skill tree, or public-directory submission is required for these GitHub installation routes. The skills CLI clones Git sources and discovers conventional skill directories; OpenAI documents skills-only portable packages and separate repository marketplaces. [Skills Git installer](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/add.ts#L1352-L1421), [OpenAI packaging documentation](https://developers.openai.com/plugins/build/plugins).

```text
plugin.json
.agents/plugins/marketplace.json
LICENSE
skills/
  <existing-or-converted-skill>/
    SKILL.md
    LICENSE
    references/  # only supporting files this skill needs
    scripts/     # only scripts this skill needs
```

Root `plugin.json`:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "addyosmani-skills",
  "version": "0.1.0",
  "description": "Addy Osmani engineering workflows adapted for Codex.",
  "repository": "https://github.com/Qiuyi-Hong/addyosmani-skills",
  "license": "MIT"
}
```

`.agents/plugins/marketplace.json`:

```json
{
  "name": "addyosmani-skills",
  "plugins": [
    {
      "name": "addyosmani-skills",
      "source": { "source": "local", "path": "./" },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
```

These are exact native manifest shapes. `./` means the repository root, not `.agents/plugins/`. Plugin installation rejects a manifest name that differs from the marketplace entry; marketplace top-level `name` supplies the selector suffix. Portable packages discover direct children of root `skills/`, so keep the tree flat. [Codex marketplace resolver](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/marketplace.rs#L651-L693), [identity validation](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/store.rs#L303-L327), [portable skill loader](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/loader.rs#L1046-L1068).

## What already exists upstream

Inspected upstream `main` at immutable commit `2686b620fc1fed2e8f60c704839c766b8594c6b6`. It already has 25 ordinary skills, root `plugin.json`, `.agents/plugins/marketplace.json` with source `./`, and a `.codex-plugin/plugin.json` compatibility manifest, all named `agent-skills`, version `0.6.11`. Reuse that packaging rather than build another hierarchy. [Upstream tree](https://github.com/addyosmani/agent-skills/tree/2686b620fc1fed2e8f60c704839c766b8594c6b6), [native marketplace](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/.agents/plugins/marketplace.json), [compatibility manifest](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/.codex-plugin/plugin.json).

Upstream root `plugin.json` lacks `$schema`; Codex 0.159.3 consequently ignores it as a portable manifest and selects the legacy compatibility manifest. Add the schema for the proposed package. `.codex-plugin/plugin.json` is optional for this verified baseline; retain and align its identity/version only if older clients remain supported. A schema-declared portable package also avoids the legacy install-time command importer. [Upstream root manifest](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/plugin.json), [manifest selection and fallback](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/utils/plugins/src/plugin_namespace.rs#L25-L86), [command-import gate](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/store.rs#L635-L644).

## Cross-installer constraints

- **Resources must travel with each skill.** The skills CLI copies the selected skill directory, not the whole repository. Eleven upstream skills reference `../../references/`; those links break after per-skill installation. Generate the necessary resource closure inside each affected skill and rewrite paths; preserve existing skill-local resources. Normalize `idea-refine`'s repository-relative script invocation too. Codex copies the full plugin root, so a plugin-only check would miss this failure. [Skills copy behavior](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/installer.ts#L352-L390), [recursive resource copy](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/installer.ts#L459-L524), [upstream documented portability gap](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/docs/skill-anatomy.md#L115-L125), [idea-refine invocation](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/skills/idea-refine/SKILL.md#L15-L23).
- **Keep source snapshots out of discovery.** Normal discovery prioritizes known skill containers; fallback and `--full-depth` recursively search up to five levels. Hidden directories are not excluded. Only `node_modules`, `.git`, `dist`, `build`, and `__pycache__` are skipped; repeated names are normally silently deduplicated. A hidden upstream mirror can expose unadapted new skills. Fetch pristine sources temporarily and record their immutable SHA, rather than keep a second live `SKILL.md` tree in the distributable checkout. [Discovery implementation](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skills.ts#L10-L39), [recursive fallback](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skills.ts#L140-L156), [priority search and deduplication](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skills.ts#L254-L331).
- **Use regular files for bundled resources.** Codex's package copier skips symlinks. Generate small supporting-file copies where needed; do not symlink them outside a skill. [Codex package copier](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/store.rs#L731-L762).
- **Carry attribution through both routes.** Preserve the upstream MIT copyright and permission notice at the root and in each independently installable skill, for example as skill-local `LICENSE`; a root-only notice does not travel through the skills CLI. [Upstream license](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/LICENSE).

The literal `npx skills add Qiuyi-Hong/addyosmani-skills` uses the CLI's selection flow and defaults to project installation. It makes the collection available to choose; it does not guarantee every skill is selected. Document the full-collection Codex form `--agent codex --skill '*' --yes`. Codex project skills land in `.agents/skills/`; plugin skills belong to the plugin namespace. Do not assume a selectively installed wrapper has every sibling workflow installed. [CLI options](https://skills.sh/docs/cli), [first-party CLI README](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/README.md#L85-L120), [Codex target path](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/agents.ts#L224-L232).

## Versions and refresh

Use a fork-owned version in root `plugin.json` and bump it for every shipped package change; record the upstream SHA separately. Codex uses the declared manifest version for its cache path, `plugins/cache/<marketplace>/<plugin>/<version>`, and normal refresh may skip unchanged versions. Explicit `codex plugin marketplace upgrade addyosmani-skills` fetches a newer configured Git snapshot and forces reinstall of configured plugins when that snapshot changes; `codex plugin add` also supports reinstall. This CLI has no `codex plugin update` command. [Cache/version implementation](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/store.rs#L112-L119), [normal version gate](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/loader.rs#L665-L690), [explicit marketplace refresh](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/core-plugins/src/manager.rs#L2926-L2992), [reinstall regression test](https://github.com/openai/codex/blob/01fc69f4026735edfdf6789820549727a4867b11/codex-rs/cli/tests/plugin_cli.rs#L1039-L1065).

The skills CLI tracks source/ref and skill-folder content hashes, not the plugin version; use `npx skills update`. Bundled resource changes are therefore part of the same skill's update. [Global lock fields](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skill-lock.ts#L15-L34), [project folder hashing](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/local-lock.ts#L141-L159).

## Verification and implementation acceptance

Baseline: installed Codex CLI **0.159.3**, official release commit `01fc69f4026735edfdf6789820549727a4867b11`; published skills CLI **1.7.0**, tested with Node **26.8.2**. npm reports publication `gitHead` `7407f3893ad4dceab546ac002c3ef806e4000c73`. The cited current source `3694740352eeef5cdd689af694c485f1ff62eec3` is nine commits later; discovery, directory-copy, plugin-discovery, and lock files are unchanged. The package declares Node `>=22.20.0`. [Codex release](https://github.com/openai/codex/releases/tag/rust-v0.159.3), [published npm metadata](https://registry.npmjs.org/skills/1.7.0), [source comparison](https://github.com/vercel-labs/skills/compare/7407f3893ad4dceab546ac002c3ef806e4000c73...3694740352eeef5cdd689af694c485f1ff62eec3), [skills package metadata](https://github.com/vercel-labs/skills/blob/7407f3893ad4dceab546ac002c3ef806e4000c73/package.json).

Executed temporary fixture checks, using the published skills tarball:

1. Ordinary discovery found two canonical skills; `--full-depth` additionally found a unique skill inside `.upstream/skills/`.
2. A names-only fixture containing the 25 upstream skills plus all nine proposed command stems found **34** skills normally, including `build`; the hidden raw-source skill increased full-depth discovery to **35**.
3. A project-only `--agent codex --copy --yes --skill demo` install copied `SKILL.md`, its local reference, and `LICENSE`. An assertion confirmed the repository-root reference was absent.
4. A read-only Codex listing with an ephemeral configuration override recognized the exact proposed manifest pair and returned `pluginId: addyosmani-skills@addyosmani-skills`, version `0.1.0`, and the fixture root as local source:

   ```sh
   codex plugin list --marketplace addyosmani-skills --available --json \
     -c 'marketplaces.addyosmani-skills={source_type="local",source="/absolute/temporary/fixture"}'
   ```

No real plugin installation, user configuration change, `HOME`/`CODEX_HOME` override, production package edit, or public installer success is claimed. After implementation is on the public default branch, run the following from a temporary project on a fresh disposable CI runner or OS account, using its normal home:

```sh
npx --yes skills@1.7.0 add Qiuyi-Hong/addyosmani-skills --list
npx --yes skills@1.7.0 add Qiuyi-Hong/addyosmani-skills \
  --agent codex --skill '*' --yes --copy
codex plugin marketplace add Qiuyi-Hong/addyosmani-skills --json
codex plugin add addyosmani-skills@addyosmani-skills --json
codex plugin list --marketplace addyosmani-skills --json
```

Assert both routes expose the same expected unique skill inventory; every required local reference, script, persona resource, and license survives installation; the plugin ID/version match the manifests; and a Codex skill-list check discovers each converted command. Repeat against a changed candidate version and marketplace refresh to prove the new contents replace the cache. Use the three literal user-requested commands for the final interactive smoke test. Public installation and runtime invocation remain implementation acceptance checks. Older-client compatibility is the only optional packaging scope choice; this recommendation targets the verified installed baseline.
