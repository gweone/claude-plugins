# sharpps-liferay

Liferay knowledge and workflow tooling for solutions built on the [SharpPS](https://github.com/gweone) framework.

## Skills

| Skill | Covers |
|---|---|
| `build` | Running a full Gradle `clean build` from the workspace root and reporting the resulting module `.jar` / client-extension `.zip` output. |
| `deploy` | Copying built module `.jar` / client-extension `.zip` artifacts into the Liferay dev environment's hot-deploy folder. |
| `clean-logs` | Truncating a Docker container's logs to zero without restarting it. |
| `liferay-logs` | Streaming/tailing the `liferay-dev` container's logs via Docker (Compose) and flagging startup, bundle-deploy, and error events. |
| `liferay-batch-client-extension` | Creating/debugging a Liferay "Batch" Client Extension (`type: batch`, `*.batch-engine-data.json`) that imports Object Definitions/Entries/Relationships, List Types, Workflow Definitions, Roles, or User Accounts via the Headless Batch Engine - including the workaround it provides when a Site Initializer can't target a system-required site. |
| `liferay-ddm-field-validation` | Adding/debugging validation rules on a Dynamic Data Mapping (DDM) field shared by Document Types, Objects, and Forms - declarative expression validation vs. a custom `DDMValidation` type. |
| `liferay-management-toolbar-search-container` | Wiring `<clay:management-toolbar selectable="true">` with `<liferay-ui:search-container>` + `RowChecker` for bulk-select checkboxes, "select all", and bulk actions in admin JSPs. |
| `liferay-portal-source` | Exploring, navigating, and reasoning about the Liferay Portal source tree - modules, portlets, OSGi components, service builder entities, themes, frontend assets, and Gradle build tasks. |
| `liferay-site-initializer` | Creating/debugging a Site Initializer that provisions a new Site with pre-built content (pages, web content, documents, objects, blueprints, taxonomies, fragments, roles). |
| `liferay-taglibs` | Choosing and using Liferay JSP taglibs and FreeMarker (FTL) macros, including building a custom taglib from scratch. |
| `liferay-widget-config-erc` | The rule that portlet/widget configuration referencing another Liferay entity must store its ERC, never a raw numeric ID. |

## Source

These skills originate from the Liferay workspace at `/opt/github/PISLIB/liferay/.claude/skills`.
This plugin packages the same skills for installation into any repository via
the Claude Code plugin system, independent of that workspace.
