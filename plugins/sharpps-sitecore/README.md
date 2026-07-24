# sharpps-sitecore

Sitecore/SXA knowledge for solutions built on the [SharpPS](https://github.com/gweone) framework.

## Skills

| Skill | Covers |
|---|---|
| `sitecore` | Interacting with a running Sitecore instance via the `SharpPS.Shells` PowerShell module - bootstrapping, `SharpPS.config`, pipeline auto-resolution, logging in (`New-SitecoreSession`), reading the live merged config, and creating/manipulating CMS items (templates, renderings, datasources) via SPE (`New-SPESession`). |
| `sxa-search` | Sitecore XA's native search feature - `Sitecore.XA.Feature.Search.Controllers.SearchController` and the `/sxa/search/*` Web API endpoints: routes, query string params (including `sc_site`/`sc_database`/facet filtering), response shapes, and how Rendering Variants (`v`/`VariantID`) render each result's `Html`. |
| `feature-foundation-project` | Adding a new Feature or Foundation layer project to an existing SharpPS Sitecore solution via the "Sitecore MVC" VS project template - naming/location convention, what gets scaffolded, and Feature's automatic `ProjectReference` to the Foundation project. |
| `sxa-cshtml-controller-rendering` | Wiring a `.cshtml` up as a Controller Rendering with a custom view path (instead of a View Rendering) - reusing the existing `Sitecore.XA.Foundation.Mvc.Controllers.StandardController` with no new controller code, and converting an existing rendering item's template via SPE. |

## Source

These skills originate from `SharpPS.VisualStudio`'s `solutionitems.vstemplate`,
which scaffolds them into every new SharpPS Sitecore solution under
`.claude/skills/`. This plugin packages the same skills for installation into
any repository via the Claude Code plugin system, independent of that
template.
