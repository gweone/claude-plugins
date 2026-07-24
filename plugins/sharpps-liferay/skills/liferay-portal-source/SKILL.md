---
name: liferay-portal-source
description: >
  Use this skill whenever the user wants to explore, understand, navigate, search, modify,
  build, debug, or reason about the Liferay Portal source code located at /opt/github/liferay-portal.
  Trigger this skill for any question about Liferay modules, portlets, OSGi components, service
  builder entities, themes, frontend JS/CSS, Gradle build tasks, upgrade steps, APIs, hooks,
  fragments, or customizations rooted in that codebase. Also trigger when the user says things
  like "in our Liferay source", "in the portal repo", "find the class for X in Liferay",
  "how does Liferay implement X", or asks to make changes to any file under /opt/github/liferay-portal.
---

# Liferay Portal Source Skill

## Source Root

```
/opt/github/liferay-portal
```

Always use this as the base path for all exploration, searches, and edits.

---

## First Step: Orient Before Acting

Before answering any question or making any change, run a quick orientation if you haven't already:

```bash
# Top-level structure
ls /opt/github/liferay-portal

# Identify the version
cat /opt/github/liferay-portal/release.properties 2>/dev/null \
  || cat /opt/github/liferay-portal/portal-impl/src/portal.properties 2>/dev/null | grep "^build.number" \
  || git -C /opt/github/liferay-portal log --oneline -1
```

Do NOT skip orientation — the layout differs between Liferay 7.0, 7.1, 7.2, 7.3, 7.4, and DXP versions.

---

## Canonical Directory Layout

```
liferay-portal/
├── modules/                    # OSGi bundles (the modern way, Liferay 7+)
│   ├── apps/                   # Product feature modules
│   │   ├── account/
│   │   ├── blogs/
│   │   ├── commerce/
│   │   ├── document-library/
│   │   ├── journal/            # Web Content
│   │   ├── portal-search/
│   │   └── ...
│   ├── core/                   # Framework / platform modules
│   │   ├── petra/
│   │   ├── portal-bootstrap/
│   │   └── ...
│   ├── dxp/                    # DXP-only (enterprise) modules
│   ├── private/                # Private/EE modules (may be absent in CE)
│   └── test/                   # Test infrastructure
│
├── portal-impl/                # Core portal implementation (legacy + shared)
│   └── src/
│       ├── com/liferay/portal/ # Portal services, utilities, legacy portlets
│       └── portal.properties   # Master configuration defaults
│
├── portal-kernel/              # Public API surface (interfaces, base classes)
│   └── src/com/liferay/portal/kernel/
│
├── portal-web/                 # WAR: JSPs, themes, static resources
│   └── docroot/
│       ├── html/               # Legacy JSP portlets
│       ├── WEB-INF/
│       └── errors/
│
├── portal-service/             # (Older repos) or merged into portal-kernel
│
├── util-bridges/               # JSF, Struts, Spring bridge utilities
├── util-java/                  # General Java utilities
├── util-taglib/                # JSP tag libraries
│
├── themes/                     # Classic/styled/admin base themes
│
├── sql/                        # DB schema, indexes, sequences
│   ├── portal-tables.sql
│   ├── indexes.sql
│   └── create/
│
├── lib/                        # Third-party JARs (pre-OSGi dependencies)
│
├── tools/                      # Build tools, SDK, Ant tasks
│   └── sdk/
│
├── build.xml                   # Root Ant build
├── build-common.xml
├── gradlew / settings.gradle   # Gradle workspace entry points
└── .gradle/
```

---

## Module Structure (inside `modules/`)

Each module follows OSGi bundle conventions:

```
modules/apps/<product>/<bundle-name>/
├── bnd.bnd                        # Bundle symbolic name, version, exports
├── build.gradle                   # Dependencies, tasks
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/liferay/...   # Java source
│   │   └── resources/
│   │       ├── META-INF/
│   │       │   └── resources/     # JSP, CSS, JS for the module
│   │       └── content/           # i18n Language.properties
│   └── test/
│       └── java/                  # Unit + integration tests
└── package-info.java (optional)
```

Service Builder modules add:

```
├── <bundle>-api/          # Generated interfaces + model classes
├── <bundle>-service/      # Generated service implementation
│   └── service.xml        # Entity definitions → triggers code gen
└── <bundle>-web/          # Portlet frontend
```

---

## Key Patterns & How to Find Things

### Finding a class

```bash
# Fast: use find + grep
find /opt/github/liferay-portal -name "ClassName.java" 2>/dev/null

# With partial name
find /opt/github/liferay-portal -name "*Journal*Service*.java" 2>/dev/null
```

### Finding where a portlet lives

```bash
# By portlet name keyword
grep -r "com.liferay.portlet.display-name" /opt/github/liferay-portal/modules \
  --include="*.properties" -l | grep -i "<keyword>"

# By portlet ID / javax.portlet.name
grep -r "javax.portlet.name=" /opt/github/liferay-portal/modules \
  --include="*.java" -l | grep -i "<keyword>"
```

### Finding OSGi component registration

```bash
grep -r "@Component" /opt/github/liferay-portal/modules/apps/<product> \
  --include="*.java" -l
```

### Finding Service Builder entities

```bash
find /opt/github/liferay-portal -name "service.xml" | xargs grep -l "<entity name"
```

### Searching configuration properties

```bash
grep -r "some.config.key" /opt/github/liferay-portal/portal-impl/src/portal.properties
grep -r "some.config.key" /opt/github/liferay-portal/modules --include="*.cfg" -l
```

---

## Build System

Liferay uses **both Ant and Gradle** depending on location:

| Area | Build tool | Command |
|---|---|---|
| Root / portal-impl / portal-web | Ant | `ant -f build.xml <target>` |
| modules/ (OSGi bundles) | Gradle | `./gradlew :<module-path>:deploy` |
| Single module deploy | Gradle | `cd modules/apps/X/Y && ../../../gradlew deploy` |
| Build all modules | Gradle | `./gradlew deploy` (slow) |
| Run tests | Gradle | `./gradlew <module>:test` |
| Clean | Gradle | `./gradlew clean` |

Common Gradle tasks:
- `deploy` — build JAR and copy to Liferay's `osgi/modules/`
- `buildService` — regenerate Service Builder code from `service.xml`
- `formatSource` — run code formatter (required before commits)
- `jar` — build without deploying

---

## OSGi & Declarative Services (DS)

Liferay 7+ is OSGi-based. Key annotations:

```java
@Component(
    immediate = true,
    property = {
        "javax.portlet.name=" + MyPortletKeys.MY_PORTLET
    },
    service = Portlet.class
)
public class MyPortlet extends MVCPortlet { ... }
```

- Services are wired via `@Reference` injection
- Configuration uses `@Meta.OCD` + `@ExtendedObjectClassDefinition`
- Component factories use `@Component(factory = "...")`

When modifying a component, check `bnd.bnd` to confirm the package is exported if it's part of the API.

---

## Frontend (JS / CSS)

Modern Liferay frontend lives in:

```
modules/apps/<product>/<bundle>-web/src/main/resources/META-INF/resources/
├── js/           # ES modules, Clay components, React
├── css/          # SCSS
└── *.jsp         # Server-side templates
```

Older portlets use:
```
portal-web/docroot/html/portlet/<portlet-name>/
```

JS build output is managed by `liferay-npm-bundler` or Webpack; check `package.json` in the module.

---

## Database / SQL

Schema lives in:
```
sql/portal-tables.sql      # Table definitions
sql/indexes.sql            # Index definitions
sql/sequences.sql          # Sequences (Oracle / PostgreSQL)
sql/create/               # Per-database DDL
```

Service Builder auto-generates `*ModelImpl.java` mapped to tables. Column → field mappings are in `service.xml`.

---

## Common Tasks Cheat Sheet

| Task | Where to look |
|---|---|
| Add a new portlet | `modules/apps/<product>/` — create `-web` module with `@Component(service = Portlet.class)` |
| Override a service | Create a module with `@Component(service = XService.class)` with higher `service.ranking` |
| Add a new entity | Edit `service.xml` → run `buildService` |
| Change portal.properties default | `portal-impl/src/portal.properties` (dev) or `portal-ext.properties` (runtime) |
| Add language keys | `src/main/resources/content/Language.properties` in the relevant module |
| Customize a JSP | Use Dynamic Include or JSP hook via `CustomJspBag` implementation |
| Add a configuration UI | Implement `ConfigurationScreen` or use System Settings via `@Meta.OCD` |
| Check if an asset type has its own native "view" widget on a page (vs. needing a generic/embedded preview) | `AssetRendererFactory.getURLView(LiferayPortletResponse, WindowState)` — a generic, type-agnostic hook (unrelated to Objects specifically). `BaseAssetRendererFactory`'s default returns `null`; `DLFileEntry`/`DLFolder`/`JournalArticle`/`JournalFolder`/`BlogsEntry`/`WikiPage`/`MBMessage`/`MBCategory`/Bookmarks override it for real. Objects/KBArticle don't, so they fall through to an embedded preview. |

---

## Tips

- **Never edit generated files** — classes in `*-service/src/main/java/.../service/impl/` that are prefixed with `Base` are generated. Edit only `*LocalServiceImpl.java` / `*ServiceImpl.java`.
- **Check `bnd.bnd` versions** — bump `Bundle-Version` when modifying a module or the OSGi container won't pick up changes.
- **Gradle wrapper** — always use `./gradlew` (not system `gradle`) to respect the project's Gradle version.
- **Format before committing** — run `./gradlew formatSource` to avoid CI failures.
- **DXP vs CE** — modules under `modules/dxp/` or `modules/private/` are enterprise-only and may not be present in Community Edition checkouts.
- **`ThemeDisplay.getURLCurrent()` vs `request.getRequestURL()`** — the friendly URL filter rewrites a request like `/web/guest/object-list` to `/c/portal/layout?p_l_id=...` *before* the portlet sees it, so `getRequestURL()` always returns the rewritten form, not what the browser's address bar shows. Use `getURLCurrent()` when you need the actual friendly URL (e.g. to detect what page/widget the user is currently on).