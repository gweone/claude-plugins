---
name: sxa-search
description: Use when working with Sitecore XA (SXA)'s native search feature - Sitecore.XA.Feature.Search.Controllers.SearchController and the /sxa/search/* Web API endpoints. Covers routes, query string params, and response shapes for results/facets/suggestions.
---

# Sitecore XA Feature Search (native)

`Sitecore.XA.Feature.Search.Controllers.SearchController` is SXA's own search
Web API controller, routed by SXA per-site as
`{site-virtual-folder}/sxa/{controller}/{action}`, e.g. `/sxa/search/results`.
Web API (not MVC) - query string only, JSON only.

| Action | Route | Model binder | Response |
|---|---|---|---|
| `GetResults` | `/sxa/search/results` | `QueryModel` | `ResultSet` |
| `GetFacets` | `/sxa/search/facets` | `FacetsModel` | `FacetSet` |
| `GetSuggestions` | `/sxa/search/suggestions` | `QueryModel` | `SuggestionsSet` |

## Query string parameters

| Param | Used by | Binds to | Meaning |
|---|---|---|---|
| `q` | all | `Query` | search term; server truncates to 100 chars |
| `s` | all | `ScopesIDs` | `,`/`\|`-separated item IDs whose `ScopeQuery` field defines the search scope |
| `l` | all | `Languages` | `,`/`\|`-separated language codes |
| `sig` | all | `Signature` | opaque client token, echoed back HTML-encoded |
| `sc_site` | all | `Site` | site name, drives home item / index resolution |
| `itemid` | all | `ItemID` | context item; drives `IIndexResolver` + scope tokens |
| `g` | all | `Coordinates` | `lat\|lng` (normalized to `lat,lng`); presence alone flips the request into geolocation mode |
| `o` | Results/Suggestions | `Sortings` | `\|`-separated sort fields, e.g. `field1\|-field2` |
| `e` | Results/Suggestions | `Offset` | paging offset (int) |
| `p` | Results/Suggestions | `PageSize` | page size (int), default `20` |
| `v` | Results/Suggestions | `VariantID` | rendering variant used to render each result's `Html` |
| `f` | Facets | `Facets` | `,`/`\|`-separated facet field keys to compute |

Facet filter widgets (checklist/range/radius/etc.) append their own query
params, read independently by `SearchService.GetQuery` via
`Context.Request.QueryString` - not part of the model binders above.

### How to specify site and database on a search request

Verified live against a real instance (`sitecorex41.dev.wsc`, Sitecore XM
10.x) - same query, only the session differed:

| Session | Resulting `Index` |
|---|---|
| Unauthenticated / bad credentials | `sitecore_sxa_web_index` |
| Authenticated as `admin` (Administrator) | `sitecore_sxa_master_index` |

```powershell
$sp = ConvertTo-SecureString $Password -AsPlainText -Force
$webSession = New-SitecoreSession -Url $Url -Username $Username -Password $sp

$query = "/sxa/search/results?sc_site=$SiteName&sc_database=$Database&itemid=$ItemId&v=$VariantId&p=5&e=0"
Invoke-WebRequest -Uri "$Url$query" -WebSession $webSession -UseBasicParsing
```

- `sc_site` - plain public param (`Site` on `BaseModel`), works for anyone,
  no authentication needed. Drives home-item/predicate scoping (see
  `PageOrMediaPredicate` below) - **not** the same thing as choosing a
  database.
- `sc_database` - **not** part of the SXA search request models at all; it's
  core Sitecore's own `sc_database` query string switch
  (`Sitecore.Pipelines.HttpRequest.DatabaseResolver`, part of the standard
  `httpRequestBegin` pipeline, evaluated before the search controller ever
  runs). It only takes effect when the request is authenticated as an
  Administrator or a member of `Sitecore Client Users` - an anonymous/
  unauthenticated call silently ignores it and the database stays whatever
  the site is configured for (typically `web`). That's why the example above
  goes through `New-SitecoreSession`'s `$webSession` rather than a bare
  `Invoke-WebRequest` - a plain anonymous fetch (browser JS, curl, etc.)
  cannot exercise `sc_database` at all.

Full details on both mechanisms - including why there's no equivalent
`?index=` override - are below.

### No way to specify the index name directly - it's always server-resolved

`QueryModel`/`FacetsModel`/`BaseModel` (see the param table above) have no
`Index`/`IndexName` property, and `BaseModelBinder<T>` only ever binds `q`,
`s`, `l`, `sig`, `sc_site`, `itemid`, `g` - there's no query string key that
picks an index directly. Resolution is entirely server-side, via
`IIndexResolver.ResolveIndex(contextItem)` (`Sitecore.XA.Foundation.Search.
Services.IndexResolver`), driven only by `itemid`/`sc_site`:

1. If the `itemid` item's site has an `indexes` site-definition property
   (`db/lang/{shortid}` or `db/*/{shortid}`), that named index wins.
2. Else `sitecore_sxa_{db}_index` if it exists.
3. Else a page-mode-based default: `sitecore_sxa_web_index` (normal/live
   mode) or `sitecore_sxa_master_index` (edit/preview mode), falling back
   further to the plain Sitecore content index for that database if even
   those don't exist.

So the only *indirect* lever a client has over which index gets queried is
`itemid` (via that item's site config) and `sc_site` - there's no `?index=`
override, and building one would mean overriding `IIndexResolver` yourself
(the kind of thing `SharpPS.Sitecore.ContentSearch.XA`'s
`BucketableIndexResolver` does - out of scope for this skill, see its own
project if that's actually needed).

**The database (`{db}` above) isn't part of the search request models at
all** - `QueryModel`/`FacetsModel`/`BaseModel` have no database property, and
`SearchService.GetContextItem` just does `Context.Database.GetItem(itemId)`,
reading whatever `Context.Database` already is for the current request (also
where `contextItem.Database.Name`, the `{db}` in `sitecore_sxa_{db}_index`,
comes from). `sc_site` doesn't touch it either - it only feeds
`SearchContextService.GetHomeItem(siteName)` for scoping the search
predicate (home item, associated content).

That said, `Context.Database` for the request *can* be switched before the
search controller ever runs - by core Sitecore's own `sc_database` query
string switch (`Sitecore.Pipelines.HttpRequest.DatabaseResolver`, part of
the standard `httpRequestBegin` pipeline, not anything SXA/search-specific -
`sc_content` is **not** this key, that's a red herring for a different,
unrelated client/page-state string). It's gated, though:

```csharp
protected virtual bool CanSetDatabaseByQueryString(HttpRequestArgs args, string databaseName)
{
    if (string.IsNullOrEmpty(databaseName)) return false;
    User user = args.SitecoreContext.User;
    if (user != null)
    {
        if (!user.IsAdministrator) return user.IsInRole(Constants.SitecoreClientUsersRole);
        return true;
    }
    return false;
}
```

So `?sc_database=master&...` on a `/sxa/search/results` call **only** takes
effect if the request is authenticated as an Administrator or a member of
`Sitecore Client Users` - for an anonymous front-end visitor it's silently
ignored (`Context.Database` stays whatever the site's configured database
is). Not something a public-facing search widget can rely on; only useful
for an authenticated CM-side/admin request.

**Practical implication for testing `sc_database`:** a plain anonymous call
(browser fetch, unauthenticated `Invoke-WebRequest`) won't carry Sitecore
auth, so `sc_database` gets silently dropped and it looks like the parameter
does nothing. To actually exercise it, issue the request through the
authenticated session from "Logging into a Sitecore instance" in the
`sitecore` skill - reuse its `$webSession` (from `New-SitecoreSession`) via
`Invoke-WebRequest -WebSession $webSession -Uri "$Url/sxa/search/results?...
&sc_database=master"` in PowerShell, not an anonymous request. `sc_site` is
unrelated to this - it isn't gated by authentication at all (see above), so
don't conflate the two: only `sc_database` needs the authenticated session,
because only `sc_database` is permission-checked.

### Filtering by a facet: the query string key is the Facet item's own name

`SearchService.GetQuery` calls `query.ApplyFacetFilters(Context.Request.
QueryString, ...)` (`FacetExtensions.cs`) for **every** request (`GetResults`
and `GetFacets` both), not just `GetFacets` - so a facet filter param on
`/sxa/search/results` narrows the actual results, not only the facet counts.

`ApplyFacetFilters` matches query string keys against Facet items under
`{sxa-site}/Settings/Facets/{facet}` (`FacetService.GetFacetItems`, via
`MultisiteContext.GetSettingsItem(site).FirstChildInheritingFrom(Templates.
FacetsGrouping.ID)`) by the Facet item's **own Sitecore item name** -
`facetItem.Name` - then reads the filter value with `queryString[facetItem.
Name]`. That's the content-tree node name of the Facet item itself, **not**
its `Name` field (`Templates.Facet.Fields.Name` is only used as the
`FriendlyName`/`Facet.Name` label shown in `GetFacets`' aggregation
response - a different value). So to filter by a Facet item named `Category`,
send `?...&Category=SomeValue`, regardless of what that item's `Name` field
is set to.

Value format depends on the Facet's type (checked via `DoesItemInheritFrom`):
- Plain match: exact value, or `|`-separated to build an OR-set (`_empty_`
  represents an empty value)
- Range (`FloatFacet`/`IntegerFacet`/`DateFacet`/generic): `from|to` (either
  side omittable for an open-ended range)
- `DistanceFacet`: a distance string (e.g. `10km`) evaluated against the `g`
  coordinate as the query's center

## Response shapes

All three carry `TotalTime`, `QueryTime`, `Signature`, `Index`, plus their own results array.

| Type | Extra fields |
|---|---|
| `ResultSet` | `Count`, `Results: Result[]` |
| `FacetSet` | `Facets: Facet[]` |
| `SuggestionsSet` | `Results: Suggestion[]` |

| Model | Fields |
|---|---|
| `Result` | `Id`, `Language`, `Path`, `Url`, `Name`, `Html` (see `v`/`VariantID` section above) |
| `Facet` | `Key`, `Name`, `DisplayName`, `Values: FacetValue[]` |
| `FacetValue` | `Name`, `Count` |
| `Suggestion` | `Term`, `Payload`, `Html` (mirrors `Term`) |

Geolocation requests (`g` param present) return a `Result` with an extra
`Geospatial` property (distance/coordinate info relative to the request's
`g` center) instead of the plain shape above.

## Suggestions behavior

`GetSuggestions` behavior depends on the `search:define` appSetting: on
Lucene it's just `GetResults` with each result's rendered `Html` returned as
a `Suggestion.Term`; on Solr it calls the native Solr Suggest handler
(`sxaSuggester`, top 5) for real prefix suggestions.

## How `v`/`VariantID` renders each result's `Html`

Each result's `Html` is produced by SXA's general-purpose **Rendering
Variants** mechanism (`Sitecore.XA.Foundation.Variants.Abstractions` /
`Sitecore.XA.Foundation.RenderingVariants` - the same engine used by List
renderings etc.), just applied per search-result item instead of per-list-item:

1. A Rendering Variant is a Sitecore item tree of typed field nodes -
   `VariantField`, `VariantText`, `VariantSection`, `VariantPlaceholder`,
   `VariantScriban`, `VariantDate`, `VariantReference`,
   `VariantResponsiveImage`, `VariantHtmlTag`, `VariantToken`, `VariantModel`,
   `VariantQuery`, `VariantEditFrame`, etc. (each with its own template ID).
2. `SearchController.GetResults` resolves `Item variant =
   Context.Database.GetItem(model.VariantID)` - i.e. the item ID passed via
   the `v` query param.
3. For each result item, `new Result(searchItem, variant)` calls
   `IVariantRenderer.RenderVariant(variantItem, dataItem)`, which parses the
   variant item into `BaseVariantField`s (`IVariantFieldParser.ParseVariantFields`)
   and, for each field, runs the `renderVariantField` core pipeline (one
   processor per field type - `RenderText`, `RenderHtmlTag`, `RenderScriban`,
   `RenderPlaceholder`, etc. in
   `RenderingVariants.Pipelines.RenderVariantField`) - concatenating each
   field's output into `Result.Html`.

So `v` must be the item ID of a Rendering Variant definition item (built the
same way as any other SXA rendering variant), not something search-specific.

### Workflow: Spec -> Plan -> Execution (creating a search result variant)

**Spec - ask the user if any of this is missing:**
- Tenant/Site (same as the `sitecore` skill's CMS section - never assume)
- Variant name
- Whether it's plain field-based (`VariantText`/`VariantField`/etc.) or a
  **Component Variant Field** reusing a Rendering - if the latter, it must be
  a Controller Rendering, not a View Rendering and not a new custom
  controller (see the `sxa-cshtml-controller-rendering` skill for the exact
  fields/why), and check whether that rendering (and its backing
  Feature-layer template/rendering item, per the `sitecore` skill's CMS
  workflow) already exists or needs to be created first

**Plan - resolve in dependency order:**
1. If using a Component Variant Field and the rendering doesn't exist yet:
   run the `sitecore` skill's template/rendering/datasource workflow first,
   creating it directly as a Controller Rendering per the
   `sxa-cshtml-controller-rendering` skill - the Component Variant Field's
   `RenderingItem` field needs a real rendering item ID to point at
2. `.cshtml` location (if Component Variant Field): `~/Areas/$mvc_area$/
   Views/{FeatureName}/{RenderingName}.cshtml` - see below
3. Variant item location: `{sxa-site}/Presentation/Rendering Variants/Search
   Results/{variant name}` - see below
4. Child field node(s) under the variant item - a `VariantComponentField`
   (`{1151B2A9-08AF-4F4D-A892-C2CC9A92EA6A}`) with `RenderingItem` set to the
   rendering from step 1, or plain `VariantText`/`VariantField`/etc. nodes

**Execution:**
1. Create the `.cshtml` file and register it in the `.csproj` (see below)
2. Create the variant item and its field node(s) via `New-SPESession` +
   `Invoke-RemoteScript` (same mechanism as the `sitecore` skill's CMS
   example - don't just describe the Content Editor steps)
3. Report back the variant item's ID - that's the `v` value to pass to
   `/sxa/search/results`

### Where these variant items live in the content tree

`{sxa-site}/Presentation/Rendering Variants/Search Results` - i.e.
`/sitecore/content/{Tenant}/{Site}/Presentation/Rendering Variants/Search
Results/{variant name}`. Author/edit the variant's field nodes there in the
Rendering Variants editor; its item ID is what gets passed as `v` (or listed
in a Results Variant Selector's `AvailableVariants`).

### Component Variant Field: reusing a Rendering for each result

The `RenderComponentField` processor (`SupportedType = VariantComponentField`)
runs the referenced Rendering through the normal `mvc.getRenderer` /
`mvc.renderRendering` pipelines, and pushes a `PageContext` whose `Item` is
the current variant data item (the search result) before rendering it.

**Use a Controller Rendering, not a View Rendering, for the target of a
Component Variant Field used in search.** `/sxa/search/results` is served by
`SearchController : ApiController` - a Web API request, not a normal MVC
page-rendering pipeline request - and rendering a View Rendering outside a
page request is fragile (can throw `InvalidOperationException: ViewContext
from empty stack`, confirmed live, and the exception gets silently
swallowed into an empty result rather than surfacing as an error). See the
`sxa-cshtml-controller-rendering` skill for the full why/how - it covers reusing
the existing `Sitecore.XA.Foundation.Mvc.Controllers.StandardController`
(no new controller code) with the Controller Rendering's `RenderingViewPath`
field, and the exact steps to convert an existing rendering item via SPE.
That skill's guidance isn't search-specific, but it's exactly what a
Component Variant Field's target rendering needs here.

If you do use a View Rendering anyway, inside its `.cshtml`:

- `RenderingContext.Current.Rendering.Item` - the rendering's own
  **Datasource** item (whatever was set on the Component Variant Field's
  `Data Source` rendering parameter - typically a shared config/template item
  holding fallback values, link targets, etc., not the search result itself)
- `RenderingContext.Current.PageContext.Item` - the **search result item**
  (`args.Item` from `RenderVariant(variantItem, item)` - each entry from the
  search results, i.e. what `PageContext.Item` gets pushed to)

The `.cshtml` file itself belongs under that Feature project's own Views
folder, not a generic `Sitecore` subfolder:
`~/Areas/$mvc_area$/Views/{FeatureName}/{RenderingName}.cshtml` (e.g.
`~/Areas/v1/Views/Search/SearchResultVariant.cshtml`) - matching where the
View Rendering item's `Path` field points. Register it as an explicit
`Content` item in the Feature project's `.csproj` if no existing wildcard
`Content Include` already covers `.cshtml` files under that Views subfolder.

Example `.cshtml` (search result card, reused via a Component Variant Field):

```cshtml
@using Sitecore.Mvc
@using Sitecore.Mvc.Presentation
@using Sitecore.Data.Items
@using Sitecore.Links
@{
    // Datasource of the View Rendering itself (set via the Component Variant Field's
    // "Data Source" rendering parameter) - shared fallback config, not the result item.
    Item datasource = RenderingContext.Current.Rendering.Item;

    // The search result item for this row, pushed by RenderVariant(variantItem, item).
    Item result = RenderingContext.Current.PageContext.Item;
    string resultUrl = LinkManager.GetItemUrl(result);
}
<div class="search-result-variant">
    @Html.Sitecore().Field("Image", result)
    <div class="search-result-variant__body">
        <h3 class="search-result-variant__title">
            <a href="@resultUrl">@Html.Sitecore().Field("Title", result)</a>
        </h3>
        <p class="search-result-variant__snippet">@Html.Sitecore().Field("Description", result)</p>
        <span class="search-result-variant__url">@resultUrl</span>
    </div>
</div>
```

This is why `ResultSet.Results[].Html` in the JSON response is a plain
**string of pre-rendered markup**, not structured JSON - the server executes
this `.cshtml` per result (via the Component Variant Field -> View Rendering
chain above) and embeds its output verbatim:

```json
{
  "TotalTime": 0,
  "QueryTime": 0,
  "Signature": "string",
  "Index": "string",
  "Count": 0,
  "Results": [
    {
      "Id": "string",
      "Language": "string",
      "Path": "string",
      "Url": "string",
      "Name": "string",
      "Html": "string"
    }
  ]
}
```

(`RenderVariantField.JSON` processors exist as a separate pipeline group for
headless/JSS scenarios that need structured JSON per field instead of an
HTML string - not what the native `/sxa/search/results` endpoint uses.)

### Results Variant Selector (picking among multiple result variants)

`ResultsVariantSelectorController`/`ResultsVariantSelectorRepository` is a
separate component that renders a UI widget letting a visitor pick among
several result-display variants (e.g. grid vs. list). Its `AvailableVariants`
rendering parameter is a pipe (`|`) separated list of Rendering Variant item
IDs; the repository resolves each into a `ResultsVariantModel`
(`DataVariant` = item ID, `DataVariantIndex`, a `CssClasses` marking the
active one) for the widget to render `data-variant="{id}"` options that
client-side JS uses to drive the `v` param on subsequent
`/sxa/search/results` calls.
