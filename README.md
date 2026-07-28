# claude-plugins

A Claude Code plugin marketplace for SharpPS framework tooling and knowledge.

## Structure

- **`/plugins`** - plugins developed and maintained here

## Installation

Add this marketplace, then install a plugin from it:

```
/plugin marketplace add gweone/claude-plugins
/plugin install sharpps-sitecore@sharpps-plugins
/plugin install sharpps-liferay@sharpps-plugins
/plugin install sharpps-dotnet@sharpps-plugins
/plugin install sharpps-work@sharpps-plugins
```

or browse for the plugin in `/plugin > Discover`.

## Plugins

| Plugin | Description |
|---|---|
| [`sharpps-sitecore`](plugins/sharpps-sitecore) | Sitecore/SXA knowledge for solutions built on the SharpPS framework |
| [`sharpps-liferay`](plugins/sharpps-liferay) | Liferay knowledge and workflow tooling for solutions built on the SharpPS framework |
| [`sharpps-dotnet`](plugins/sharpps-dotnet) | .NET tooling knowledge for solutions built on the SharpPS framework |
| [`sharpps-work`](plugins/sharpps-work) | General office/documentation tooling (screen capture, Word document generation) — not tied to any one platform |

## Plugin structure

Each plugin follows the standard Claude Code plugin layout:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # Plugin metadata (required)
├── skills/               # Skill definitions
└── README.md             # Documentation
```

## Plugin names are immutable

The `name` field in a marketplace entry is an immutable slug - once a plugin
has been published, its `name` must not change. If a rename is genuinely
unavoidable, add an entry to the top-level `renames` map in
`.claude-plugin/marketplace.json` so existing installs auto-migrate.
