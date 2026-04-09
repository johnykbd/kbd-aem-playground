# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AEM (Adobe Experience Manager) Cloud project built with the AEM Project Archetype. App ID is `kbd`. It is a multi-module Maven project targeting AEM as a Cloud Service.

## Build Commands

### Full Build
```bash
mvn clean install
```

### Deploy to Local AEM Author (localhost:4502)
```bash
mvn clean install -PautoInstallSinglePackage
```

### Deploy to Local AEM Publish (localhost:4503)
```bash
mvn clean install -PautoInstallSinglePackagePublish
```

### Deploy Only the Java Bundle
```bash
mvn clean install -PautoInstallBundle
```

### Deploy a Single Content Package (e.g. ui.apps)
```bash
cd ui.apps && mvn clean install -PautoInstallPackage
```

### Run Unit Tests Only
```bash
mvn clean test
```

### Integration Tests (requires running AEM instance)
```bash
mvn clean verify -Plocal
```

## Frontend (ui.frontend)

The frontend uses Webpack 5 with TypeScript and SCSS. The Maven build uses `frontend-maven-plugin` to download Node v20.19.0 / npm 10.8.2 and run the build automatically.

For local frontend development without Maven:
```bash
cd ui.frontend
npm ci
npm run dev      # One-shot dev build + clientlib generation
npm run prod     # Production build + clientlib generation
npm start        # Dev server with hot reload (proxies to localhost:4502)
npm run watch    # Dev server + aemsync live sync to AEM
```

**Build flow:** Webpack compiles `src/main/webpack/site/main.ts` and `main.scss` → outputs to `dist/` → `aem-clientlib-generator` transforms `dist/` into AEM ClientLibs placed in `ui.apps/src/main/content/jcr_root/apps/kbd/clientlibs/`.

The webpack dev config proxies API calls to `localhost:4502`, so a running AEM author instance is needed for `npm start` to work fully.

## Module Structure

| Module | Purpose |
|---|---|
| `core` | Java OSGi bundle: Sling Models, Servlets, Filters, Schedulers |
| `ui.apps` | Components (HTL), ClientLibs, templates, overlays |
| `ui.apps.structure` | Repository structure package (required for Cloud) |
| `ui.config` | OSGi configurations per runmode (`config/`, `config.author/`, `config.publish/`) |
| `ui.content` | Sample content, DAM assets, Experience Fragments |
| `ui.frontend` | Webpack/TypeScript/SCSS frontend source |
| `all` | Container package embedding all modules for deployment |
| `it.tests` | Integration tests (aem-cloud-testing-clients) |
| `ui.tests` | Selenium UI tests |

## Java Code Architecture (`core` module)

**Base package:** `com.kbd.playground.core`

**Subpackages and patterns:**

- `models/` — Sling Models annotated with `@Model(adaptables = Resource.class)`. Use `@ValueMapValue` for JCR properties, `@SlingObject` for Sling objects, `@OSGiService` for injecting services, `@PostConstruct` for initialization logic.
- `servlets/` — Extend `SlingSafeMethodsServlet` (GET) or `SlingAllMethodsServlet` (POST). Register via `@SlingServletResourceTypes`.
- `filters/` — Implement `javax.servlet.Filter`, register as `@Component(service = Filter.class)`. Control ordering with `@ServiceRanking`.
- `schedulers/` — Implement `Runnable`, use `@Designate` + `@ObjectClassDefinition` for OSGi configuration (cron expression, enabled flag). Configure via `/system/console/configMgr`.
- `listeners/` — Implement `ResourceChangeListener` for JCR change events.
- `testcontext/` — `AppAemContext` utility that provides a pre-configured `AemContext` (JUnit 5) with Core WCM Components and CAConfig plugins registered.

**OSGi config files** live in `ui.config/src/main/content/jcr_root/apps/kbd/osgiconfig/config[.author|.publish]/` as `.cfg.json` files. File naming: `<PID>~<factoryAlias>.cfg.json`.

## AEM Component Architecture (`ui.apps`)

Components live at `ui.apps/src/main/content/jcr_root/apps/kbd/components/`.

Most components are thin wrappers that delegate to Core WCM Components via `sling:resourceSuperType`. Custom behavior is added by creating a Sling Model in `core/` and referencing it from HTL via `data-sly-use`.

**HTL pattern:**
```html
<sly data-sly-use.model="com.kbd.playground.core.models.HelloWorldModel">
    <p data-sly-test="${model.message}">${model.message}</p>
</sly>
```

**Component group:** `KBD Playground Project - Content`

## Testing

Unit tests use JUnit 5, Mockito, and io.wcm AEM Mock. To write a model test:
1. Extend with `@ExtendWith(AemContextExtension.class)`
2. Get context from `AppAemContext.newAemContext()`
3. Create pages/resources via `context.create().page(...)` / `context.create().resource(...)`
4. Adapt resources to models and assert

## Key Properties / Defaults

| Property | Value |
|---|---|
| AEM Author | `localhost:4502` |
| AEM Publish | `localhost:4503` |
| Default credentials | `admin:admin` |
| AEM SDK API version | `2026.3.25194.20260330T181734Z-260300` |
| Core WCM Components | `2.19.0` |
| Java target | `1.8` |
| Node (frontend-maven-plugin) | `v20.19.0` |
