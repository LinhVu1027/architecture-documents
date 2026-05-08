# my-blog v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `https://blogs.linhvu.dev` v1 — a Hugo-style SSG built on Spring Boot 4 + Java 25, with the first article live, deployed by `git push origin main`.

**Architecture:** Build-time SSG. Five Spring Application Modules (`content`, `render`, `output`, `assets`, `engine`) under `dev.linhvu.my_blog`. Modules communicate by passing immutable data records. Spring Modulith verifies boundaries in tests. AWS CDK (Java) provisions S3 + CloudFront + ACM + Route 53. GitHub Actions deploys on push to main via OIDC (no static AWS keys).

**Tech Stack:** Spring Boot 4.0.6, Java 25, Spring Modulith 2.0.5, JTE 3.2.3 (`jte-spring-boot-starter-4`), commonmark-java 0.24.0 + commonmark-ext-yaml-front-matter 0.24.0, JUnit 5 + AssertJ, AWS CDK 2.x (Java), GitHub Actions.

**Phases (each ends in a verifiable milestone):**
- **Phase A — SSG foundation** (8 tasks) → `./gradlew buildSite` produces `dist/index.html` with sample article
- **Phase B — Dev server & hot reload** (2 tasks) → `./gradlew bootRun` serves http://localhost:8080, edit-reload < 500 ms
- **Phase C — Visual design via DESIGN.md flow** (2 tasks) → site has cohesive visual identity
- **Phase D — AWS infrastructure (CDK Java)** (5 tasks) → `cdk deploy` provisions empty stack at `blogs.linhvu.dev`
- **Phase E — CI/CD & first deploy** (3 tasks) → push to `main` deploys, article visible publicly

---

## File Structure

### Created in this plan

```
build.gradle                                     # extended: deps, JTE plugin, buildSite task
settings.gradle                                  # extended: include('infra')
content/                                         # user-authored markdown
├── _index.md
├── posts/
│   └── 2026-05-07-hello-world.md
└── pages/
    └── about.md
static/
└── prism.css                                    # client-side syntax highlighter style
src/main/java/dev/linhvu/my_blog/
├── MyBlogApplication.java                       # existing — minor change
├── content/
│   ├── package-info.java                        # @ApplicationModule
│   ├── ContentApi.java                          # facade
│   ├── model/
│   │   ├── package-info.java                    # @NamedInterface("model")
│   │   ├── Site.java
│   │   ├── SiteConfig.java
│   │   ├── Page.java                            # sealed
│   │   ├── Post.java
│   │   ├── StaticPage.java
│   │   ├── IndexPage.java
│   │   ├── Frontmatter.java
│   │   ├── PageId.java
│   │   ├── SectionId.java
│   │   ├── Section.java
│   │   ├── Tag.java
│   │   └── TaxonomyTerm.java
│   └── internal/
│       ├── DefaultContentApi.java
│       ├── FrontmatterParser.java
│       ├── MarkdownLoader.java
│       └── SiteAssembler.java
├── render/
│   ├── package-info.java                        # @ApplicationModule(allowedDependencies = "content::model")
│   ├── RenderApi.java
│   ├── model/
│   │   ├── package-info.java                    # @NamedInterface("model")
│   │   └── RenderedPage.java
│   └── internal/
│       ├── DefaultRenderApi.java
│       ├── LayoutResolver.java
│       ├── MarkdownRenderer.java
│       ├── SyntaxHighlighter.java
│       └── JteRenderer.java
├── output/
│   ├── package-info.java
│   ├── OutputApi.java
│   └── internal/
│       ├── DefaultOutputApi.java
│       ├── HtmlEmitter.java
│       ├── RssEmitter.java
│       ├── SitemapEmitter.java
│       └── RobotsEmitter.java
├── assets/
│   ├── package-info.java
│   ├── AssetsApi.java
│   └── internal/
│       └── DefaultAssetsApi.java
└── engine/
    ├── package-info.java
    ├── SiteBuilder.java
    ├── DevServer.java
    ├── BuildResult.java
    ├── live/
    │   └── LiveReloadEndpoint.java
    └── cli/
        └── BuildSiteCommand.java
src/main/jte/
├── base.jte
├── post.jte
├── page.jte
├── list.jte
└── tags/.gitkeep
src/main/resources/
├── application.yaml                              # existing — extended
└── static/
    └── livereload.js                             # SSE client
src/test/java/dev/linhvu/my_blog/
├── MyBlogApplicationTests.java                  # existing
├── ModuleStructureTests.java
├── content/
│   ├── FrontmatterParserTest.java
│   ├── MarkdownLoaderTest.java
│   └── SiteAssemblerTest.java
├── render/
│   ├── LayoutResolverTest.java
│   ├── MarkdownRendererTest.java
│   └── DefaultRenderApiTest.java
├── output/
│   ├── HtmlEmitterTest.java
│   ├── RssEmitterTest.java
│   └── SitemapEmitterTest.java
├── assets/
│   └── DefaultAssetsApiTest.java
└── engine/
    ├── SiteBuilderTest.java
    └── BuildSiteSmokeTest.java
src/test/resources/
└── fixtures/
    ├── _index.md
    ├── posts/
    │   └── 2026-01-01-fixture.md
    └── pages/
        └── fixture-page.md
infra/                                            # CDK Java subproject
├── build.gradle
├── cdk.json
└── src/main/java/dev/linhvu/my_blog/infra/
    ├── InfraApp.java
    └── BlogStack.java
.github/workflows/
└── deploy.yml
DESIGN.md                                         # Phase C
docs/superpowers/plans/2026-05-07-blog-v1.md     # this file
```

### Modified in this plan

- `build.gradle` — JTE plugin + Modulith BOM + commonmark deps + `buildSite` task
- `settings.gradle` — `include('infra')`
- `.gitignore` — add `dist/`, `infra/cdk.out/`
- `src/main/resources/application.yaml` — JTE config + dev port
- `src/main/java/dev/linhvu/my_blog/MyBlogApplication.java` — exclude DataSource auto-config

---

# Phase A — SSG foundation

## Task A1: Add dependencies, configure JTE, verify Spring Boot still boots

**Files:**
- Modify: `build.gradle`
- Modify: `src/main/java/dev/linhvu/my_blog/MyBlogApplication.java`
- Modify: `src/main/resources/application.yaml`
- Modify: `.gitignore`

- [ ] **Step 1: Update `build.gradle` with dependencies and JTE plugin**

Replace entire file with:

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
    id 'gg.jte.gradle' version '3.2.3'
}

group = 'dev.linhvu'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

repositories {
    mavenCentral()
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.modulith:spring-modulith-bom:2.0.5'
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'org.springframework.modulith:spring-modulith-starter-core'
    implementation 'gg.jte:jte:3.2.3'
    implementation 'gg.jte:jte-spring-boot-starter-4:3.2.3'
    implementation 'org.commonmark:commonmark:0.24.0'
    implementation 'org.commonmark:commonmark-ext-yaml-front-matter:0.24.0'

    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
    testImplementation 'org.springframework.modulith:spring-modulith-starter-test'
    testImplementation 'org.assertj:assertj-core:3.26.3'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

jte {
    sourceDirectory = file("src/main/jte").toPath()
    contentType = gg.jte.ContentType.Html
    generate()
}

tasks.named('test') {
    useJUnitPlatform()
}
```

- [ ] **Step 2: Add `dist/` and `infra/cdk.out/` to `.gitignore`**

Append to `.gitignore`:

```
### Site build output ###
dist/
infra/cdk.out/
infra/build/
```

- [ ] **Step 3: Update `MyBlogApplication.java` to opt out of unused auto-config**

Replace contents:

```java
package dev.linhvu.my_blog;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.modulith.Modulithic;

@SpringBootApplication
@Modulithic(systemName = "my-blog")
public class MyBlogApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyBlogApplication.class, args);
    }
}
```

- [ ] **Step 4: Update `application.yaml`**

Replace contents:

```yaml
spring:
  application:
    name: my-blog
  main:
    web-application-type: servlet
server:
  port: 8080

gg:
  jte:
    development-mode: true
    template-location: classpath:/jte/

myblog:
  content-root: ./content
  static-root: ./static
  dist-root: ./dist
  base-url: http://localhost:8080
  site-title: "linhvu — notes on Java"
```

- [ ] **Step 5: Create empty `src/main/jte/` directory**

```bash
mkdir -p src/main/jte
echo "" > src/main/jte/.gitkeep
```

(JTE plugin fails if the source dir is missing.)

- [ ] **Step 6: Verify build still passes**

Run: `./gradlew build -x test`
Expected: `BUILD SUCCESSFUL`. (Tests skipped — we'll add real tests next.)

- [ ] **Step 7: Verify the existing test still passes**

Run: `./gradlew test`
Expected: `BUILD SUCCESSFUL`, `MyBlogApplicationTests.contextLoads PASSED`.

- [ ] **Step 8: Commit**

```bash
git add build.gradle .gitignore src/main/java/dev/linhvu/my_blog/MyBlogApplication.java src/main/resources/application.yaml src/main/jte/.gitkeep
git commit -m "build: add JTE, Modulith, commonmark deps; baseline config"
```

---

## Task A2: Module skeleton + Spring Modulith boundary test

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/content/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/assets/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/engine/package-info.java`
- Test: `src/test/java/dev/linhvu/my_blog/ModuleStructureTests.java`

- [ ] **Step 1: Write the failing module-boundary test**

Create `src/test/java/dev/linhvu/my_blog/ModuleStructureTests.java`:

```java
package dev.linhvu.my_blog;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;

class ModuleStructureTests {

    @Test
    void verifiesModularStructure() {
        ApplicationModules modules = ApplicationModules.of(MyBlogApplication.class);
        modules.verify();
    }

    @Test
    void writesDocumentation() {
        ApplicationModules modules = ApplicationModules.of(MyBlogApplication.class);
        modules.forEach(m -> System.out.println(m.getName() + " : " + m.getDisplayName()));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: FAIL — no application modules detected (no `package-info.java` files exist yet).

- [ ] **Step 3: Create the five module `package-info.java` files**

`src/main/java/dev/linhvu/my_blog/content/package-info.java`:

```java
@org.springframework.modulith.ApplicationModule(displayName = "Content")
package dev.linhvu.my_blog.content;
```

`src/main/java/dev/linhvu/my_blog/render/package-info.java`:

```java
@org.springframework.modulith.ApplicationModule(
    displayName = "Render",
    allowedDependencies = {"content::model"}
)
package dev.linhvu.my_blog.render;
```

`src/main/java/dev/linhvu/my_blog/output/package-info.java`:

```java
@org.springframework.modulith.ApplicationModule(
    displayName = "Output",
    allowedDependencies = {"content::model", "render", "render::model"}
)
package dev.linhvu.my_blog.output;
```

`src/main/java/dev/linhvu/my_blog/assets/package-info.java`:

```java
@org.springframework.modulith.ApplicationModule(displayName = "Assets")
package dev.linhvu.my_blog.assets;
```

`src/main/java/dev/linhvu/my_blog/engine/package-info.java`:

```java
@org.springframework.modulith.ApplicationModule(
    displayName = "Engine",
    allowedDependencies = {
        "content", "content::model",
        "render", "render::model",
        "output",
        "assets"
    }
)
package dev.linhvu.my_blog.engine;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: PASS. The "writesDocumentation" test prints all 5 module names.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/content src/main/java/dev/linhvu/my_blog/render src/main/java/dev/linhvu/my_blog/output src/main/java/dev/linhvu/my_blog/assets src/main/java/dev/linhvu/my_blog/engine src/test/java/dev/linhvu/my_blog/ModuleStructureTests.java
git commit -m "feat(modulith): wire 5 application modules with verified boundaries"
```

---

## Task A3: Content model — records and sealed types

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/content/model/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/PageId.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/SectionId.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Tag.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Frontmatter.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/SiteConfig.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Section.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/TaxonomyTerm.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Page.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Post.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/StaticPage.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/IndexPage.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/model/Site.java`
- Test: `src/test/java/dev/linhvu/my_blog/content/SiteModelTests.java`

- [ ] **Step 1: Write the failing test for the sealed Page hierarchy**

Create `src/test/java/dev/linhvu/my_blog/content/SiteModelTests.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.model.*;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class SiteModelTests {

    @Test
    void postIsAPage() {
        var fm = new Frontmatter("Hello", Optional.empty(), Optional.of(LocalDate.of(2026, 5, 7)),
                                 false, List.of("java25"), Map.of());
        Page page = new Post(new PageId("/posts/hello"), fm, "<p>x</p>", "/posts/hello/",
                             new Section(new SectionId("/posts"), "posts", "/posts/", List.of()),
                             LocalDate.of(2026, 5, 7), List.of(new Tag("java25", "Java 25")));
        assertThat(page.url()).isEqualTo("/posts/hello/");
        assertThat(page.frontmatter().title()).isEqualTo("Hello");
    }

    @Test
    void layoutDispatchByPatternMatch() {
        Page p = new Post(new PageId("/p"), emptyFm("X"), "", "/p/",
                          new Section(new SectionId("/posts"), "posts", "/posts/", List.of()),
                          LocalDate.now(), List.of());
        Page s = new StaticPage(new PageId("/about"), emptyFm("About"), "", "/about/",
                                new Section(new SectionId("/pages"), "pages", "/pages/", List.of()));
        Page i = new IndexPage(new PageId("/"), emptyFm("Home"), "", "/",
                               new Section(new SectionId("/"), "root", "/", List.of()),
                               List.of());

        assertThat(layoutFor(p)).isEqualTo("post");
        assertThat(layoutFor(s)).isEqualTo("page");
        assertThat(layoutFor(i)).isEqualTo("list");
    }

    private static String layoutFor(Page page) {
        return switch (page) {
            case Post p      -> "post";
            case StaticPage s -> "page";
            case IndexPage i  -> "list";
        };
    }

    private static Frontmatter emptyFm(String title) {
        return new Frontmatter(title, Optional.empty(), Optional.empty(), false, List.of(), Map.of());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests SiteModelTests`
Expected: FAIL — model classes do not exist.

- [ ] **Step 3: Create model package marker**

`src/main/java/dev/linhvu/my_blog/content/model/package-info.java`:

```java
@org.springframework.modulith.NamedInterface("model")
package dev.linhvu.my_blog.content.model;
```

- [ ] **Step 4: Create simple ID and value records**

`src/main/java/dev/linhvu/my_blog/content/model/PageId.java`:

```java
package dev.linhvu.my_blog.content.model;

public record PageId(String path) {}
```

`src/main/java/dev/linhvu/my_blog/content/model/SectionId.java`:

```java
package dev.linhvu.my_blog.content.model;

public record SectionId(String path) {}
```

`src/main/java/dev/linhvu/my_blog/content/model/Tag.java`:

```java
package dev.linhvu.my_blog.content.model;

public record Tag(String slug, String label) {}
```

- [ ] **Step 5: Create Frontmatter, SiteConfig records**

`src/main/java/dev/linhvu/my_blog/content/model/Frontmatter.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;
import java.util.Optional;

public record Frontmatter(
    String title,
    Optional<String> description,
    Optional<LocalDate> date,
    boolean draft,
    List<String> tags,
    Map<String, Object> extra
) {}
```

`src/main/java/dev/linhvu/my_blog/content/model/SiteConfig.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.net.URI;
import java.time.ZoneId;
import java.util.Locale;

public record SiteConfig(
    URI baseUrl,
    String title,
    String description,
    ZoneId timezone,
    Locale locale
) {}
```

- [ ] **Step 6: Create Section, TaxonomyTerm, sealed Page hierarchy**

`src/main/java/dev/linhvu/my_blog/content/model/Section.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.util.List;

public record Section(SectionId id, String name, String url, List<Page> pages) {}
```

`src/main/java/dev/linhvu/my_blog/content/model/TaxonomyTerm.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.util.List;

public record TaxonomyTerm(String taxonomy, Tag tag, List<Page> pages) {}
```

`src/main/java/dev/linhvu/my_blog/content/model/Page.java`:

```java
package dev.linhvu.my_blog.content.model;

public sealed interface Page permits Post, StaticPage, IndexPage {
    PageId id();
    Frontmatter frontmatter();
    String renderedHtml();
    String url();
    Section section();
}
```

`src/main/java/dev/linhvu/my_blog/content/model/Post.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.time.LocalDate;
import java.util.List;

public record Post(
    PageId id,
    Frontmatter frontmatter,
    String renderedHtml,
    String url,
    Section section,
    LocalDate date,
    List<Tag> tags
) implements Page {}
```

`src/main/java/dev/linhvu/my_blog/content/model/StaticPage.java`:

```java
package dev.linhvu.my_blog.content.model;

public record StaticPage(
    PageId id,
    Frontmatter frontmatter,
    String renderedHtml,
    String url,
    Section section
) implements Page {}
```

`src/main/java/dev/linhvu/my_blog/content/model/IndexPage.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.util.List;

public record IndexPage(
    PageId id,
    Frontmatter frontmatter,
    String renderedHtml,
    String url,
    Section section,
    List<Page> children
) implements Page {}
```

- [ ] **Step 7: Create top-level Site record**

`src/main/java/dev/linhvu/my_blog/content/model/Site.java`:

```java
package dev.linhvu.my_blog.content.model;

import java.util.List;
import java.util.Map;

public record Site(
    SiteConfig config,
    List<Section> sections,
    List<TaxonomyTerm> taxonomies,
    Map<String, Page> byPath
) {}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `./gradlew test --tests SiteModelTests`
Expected: PASS. Both `postIsAPage` and `layoutDispatchByPatternMatch` green.

- [ ] **Step 9: Run module-boundary test to verify nothing broke**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: PASS.

- [ ] **Step 10: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/content/model src/test/java/dev/linhvu/my_blog/content/SiteModelTests.java
git commit -m "feat(content): add Site model with sealed Page hierarchy"
```

---

## Task A4: Content module — parser, loader, assembler, ContentApi

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/content/ContentApi.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/internal/FrontmatterParser.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/internal/MarkdownLoader.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/internal/SiteAssembler.java`
- Create: `src/main/java/dev/linhvu/my_blog/content/internal/DefaultContentApi.java`
- Test: `src/test/java/dev/linhvu/my_blog/content/FrontmatterParserTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/content/MarkdownLoaderTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/content/SiteAssemblerTest.java`
- Create: `src/test/resources/fixtures/_index.md`
- Create: `src/test/resources/fixtures/posts/2026-01-01-fixture.md`
- Create: `src/test/resources/fixtures/pages/fixture-page.md`

- [ ] **Step 1: Create the test fixtures**

`src/test/resources/fixtures/_index.md`:

```markdown
---
title: "Home"
description: "Notes on Java"
---

Welcome.
```

`src/test/resources/fixtures/posts/2026-01-01-fixture.md`:

```markdown
---
title: "Hello, world"
date: 2026-01-01
tags: [java25, intro]
description: "First post fixture"
draft: false
---

# Hello

This is the **fixture** post body. With `code`.
```

`src/test/resources/fixtures/pages/fixture-page.md`:

```markdown
---
title: "About"
---

This is the about page.
```

- [ ] **Step 2: Write failing test for FrontmatterParser**

`src/test/java/dev/linhvu/my_blog/content/FrontmatterParserTest.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.internal.FrontmatterParser;
import dev.linhvu.my_blog.content.internal.FrontmatterParser.ParseResult;
import dev.linhvu.my_blog.content.model.Frontmatter;
import org.junit.jupiter.api.Test;

import java.time.LocalDate;

import static org.assertj.core.api.Assertions.assertThat;

class FrontmatterParserTest {

    private final FrontmatterParser parser = new FrontmatterParser();

    @Test
    void extractsTitleAndDateAndTags() {
        String md = """
                ---
                title: "Hello"
                date: 2026-05-07
                tags: [java25, intro]
                draft: false
                ---
                Body here.
                """;

        ParseResult r = parser.parse(md);
        Frontmatter fm = r.frontmatter();

        assertThat(fm.title()).isEqualTo("Hello");
        assertThat(fm.date()).contains(LocalDate.of(2026, 5, 7));
        assertThat(fm.tags()).containsExactly("java25", "intro");
        assertThat(fm.draft()).isFalse();
        assertThat(r.body().trim()).isEqualTo("Body here.");
    }

    @Test
    void missingFrontmatterReturnsEmptyDefaults() {
        ParseResult r = parser.parse("Just a body.\n");
        assertThat(r.frontmatter().title()).isEqualTo("");
        assertThat(r.frontmatter().tags()).isEmpty();
        assertThat(r.body().trim()).isEqualTo("Just a body.");
    }

    @Test
    void draftFlagDefaultsFalse() {
        String md = """
                ---
                title: "X"
                ---
                Body.
                """;
        assertThat(parser.parse(md).frontmatter().draft()).isFalse();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `./gradlew test --tests FrontmatterParserTest`
Expected: FAIL — `FrontmatterParser` does not exist.

- [ ] **Step 4: Implement FrontmatterParser**

`src/main/java/dev/linhvu/my_blog/content/internal/FrontmatterParser.java`:

```java
package dev.linhvu.my_blog.content.internal;

import dev.linhvu.my_blog.content.model.Frontmatter;
import org.commonmark.ext.front.matter.YamlFrontMatterExtension;
import org.commonmark.ext.front.matter.YamlFrontMatterVisitor;
import org.commonmark.parser.Parser;

import java.time.LocalDate;
import java.time.format.DateTimeParseException;
import java.util.*;

public class FrontmatterParser {

    public record ParseResult(Frontmatter frontmatter, String body) {}

    private final Parser parser = Parser.builder()
            .extensions(List.of(YamlFrontMatterExtension.create()))
            .build();

    public ParseResult parse(String raw) {
        var visitor = new YamlFrontMatterVisitor();
        parser.parse(raw).accept(visitor);
        Map<String, List<String>> data = visitor.getData();

        String body = stripFrontmatter(raw);

        Frontmatter fm = new Frontmatter(
                first(data.get("title")).orElse(""),
                first(data.get("description")),
                first(data.get("date")).flatMap(FrontmatterParser::parseDate),
                first(data.get("draft")).map(Boolean::parseBoolean).orElse(false),
                Optional.ofNullable(data.get("tags")).orElse(List.of()),
                Map.of()
        );
        return new ParseResult(fm, body);
    }

    private static Optional<String> first(List<String> values) {
        return (values == null || values.isEmpty()) ? Optional.empty() : Optional.of(values.get(0));
    }

    private static Optional<LocalDate> parseDate(String s) {
        try {
            return Optional.of(LocalDate.parse(s));
        } catch (DateTimeParseException e) {
            return Optional.empty();
        }
    }

    private static String stripFrontmatter(String raw) {
        if (!raw.startsWith("---")) return raw;
        int second = raw.indexOf("\n---", 3);
        if (second < 0) return raw;
        int newline = raw.indexOf('\n', second + 4);
        return newline < 0 ? "" : raw.substring(newline + 1);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `./gradlew test --tests FrontmatterParserTest`
Expected: PASS — all three tests green.

- [ ] **Step 6: Commit FrontmatterParser**

```bash
git add src/main/java/dev/linhvu/my_blog/content/internal/FrontmatterParser.java src/test/java/dev/linhvu/my_blog/content/FrontmatterParserTest.java src/test/resources/fixtures
git commit -m "feat(content): YAML frontmatter parser"
```

- [ ] **Step 7: Write failing test for MarkdownLoader**

`src/test/java/dev/linhvu/my_blog/content/MarkdownLoaderTest.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.internal.MarkdownLoader;
import dev.linhvu.my_blog.content.internal.MarkdownLoader.LoadedFile;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class MarkdownLoaderTest {

    private final MarkdownLoader loader = new MarkdownLoader();

    @Test
    void loadsAllMarkdownFilesUnderRoot() {
        Path root = Path.of("src/test/resources/fixtures");
        List<LoadedFile> files = loader.loadAll(root);

        assertThat(files).hasSizeGreaterThanOrEqualTo(3);
        assertThat(files).anyMatch(f -> f.relativePath().toString().endsWith("_index.md"));
        assertThat(files).anyMatch(f -> f.relativePath().toString().contains("posts"));
        assertThat(files).anyMatch(f -> f.relativePath().toString().contains("pages"));
    }

    @Test
    void capturesRawContent() {
        Path root = Path.of("src/test/resources/fixtures");
        List<LoadedFile> files = loader.loadAll(root);
        var post = files.stream()
                .filter(f -> f.relativePath().toString().contains("2026-01-01-fixture"))
                .findFirst().orElseThrow();
        assertThat(post.rawContent()).contains("title: \"Hello, world\"");
        assertThat(post.rawContent()).contains("# Hello");
    }
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `./gradlew test --tests MarkdownLoaderTest`
Expected: FAIL — `MarkdownLoader` does not exist.

- [ ] **Step 9: Implement MarkdownLoader**

`src/main/java/dev/linhvu/my_blog/content/internal/MarkdownLoader.java`:

```java
package dev.linhvu.my_blog.content.internal;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.stream.Stream;

public class MarkdownLoader {

    public record LoadedFile(Path absolutePath, Path relativePath, String rawContent) {}

    public List<LoadedFile> loadAll(Path root) {
        if (!Files.exists(root)) return List.of();
        try (var exec = Executors.newVirtualThreadPerTaskExecutor();
             Stream<Path> stream = Files.walk(root)) {
            List<Future<LoadedFile>> futures = stream
                    .filter(Files::isRegularFile)
                    .filter(p -> p.getFileName().toString().endsWith(".md"))
                    .map(p -> exec.submit(() -> read(root, p)))
                    .toList();
            return futures.stream().map(MarkdownLoader::join).toList();
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private static LoadedFile read(Path root, Path file) throws IOException {
        return new LoadedFile(file, root.relativize(file), Files.readString(file));
    }

    private static <T> T join(Future<T> f) {
        try { return f.get(); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); throw new RuntimeException(e); }
        catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `./gradlew test --tests MarkdownLoaderTest`
Expected: PASS.

- [ ] **Step 11: Write failing test for SiteAssembler + DefaultContentApi**

`src/test/java/dev/linhvu/my_blog/content/SiteAssemblerTest.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.model.*;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.nio.file.Path;
import java.time.ZoneId;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class SiteAssemblerTest {

    @Test
    void buildsSiteFromFixtures() {
        ContentApi api = ContentApiTestFactory.create();
        SiteConfig cfg = new SiteConfig(URI.create("http://localhost:8080"),
                "Test", "desc", ZoneId.of("UTC"), Locale.ENGLISH);

        Site site = api.loadSite(Path.of("src/test/resources/fixtures"), cfg);

        assertThat(site.byPath()).isNotEmpty();
        assertThat(site.byPath().values()).anyMatch(p -> p instanceof Post);
        assertThat(site.byPath().values()).anyMatch(p -> p instanceof StaticPage);
        assertThat(site.byPath().values()).anyMatch(p -> p instanceof IndexPage);

        Post post = site.byPath().values().stream()
                .filter(Post.class::isInstance).map(Post.class::cast).findFirst().orElseThrow();
        assertThat(post.frontmatter().title()).isEqualTo("Hello, world");
        assertThat(post.url()).startsWith("/posts/");
    }
}
```

`src/test/java/dev/linhvu/my_blog/content/ContentApiTestFactory.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.internal.DefaultContentApi;
import dev.linhvu.my_blog.content.internal.FrontmatterParser;
import dev.linhvu.my_blog.content.internal.MarkdownLoader;
import dev.linhvu.my_blog.content.internal.SiteAssembler;

final class ContentApiTestFactory {
    static ContentApi create() {
        return new DefaultContentApi(new MarkdownLoader(), new FrontmatterParser(), new SiteAssembler());
    }
    private ContentApiTestFactory() {}
}
```

- [ ] **Step 12: Run test to verify it fails**

Run: `./gradlew test --tests SiteAssemblerTest`
Expected: FAIL — `ContentApi`, `SiteAssembler`, `DefaultContentApi` do not exist.

- [ ] **Step 13: Create ContentApi facade**

`src/main/java/dev/linhvu/my_blog/content/ContentApi.java`:

```java
package dev.linhvu.my_blog.content;

import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.content.model.SiteConfig;

import java.nio.file.Path;

public interface ContentApi {
    Site loadSite(Path contentRoot, SiteConfig config);
}
```

- [ ] **Step 14: Implement SiteAssembler**

`src/main/java/dev/linhvu/my_blog/content/internal/SiteAssembler.java`:

```java
package dev.linhvu.my_blog.content.internal;

import dev.linhvu.my_blog.content.internal.FrontmatterParser.ParseResult;
import dev.linhvu.my_blog.content.internal.MarkdownLoader.LoadedFile;
import dev.linhvu.my_blog.content.model.*;

import java.nio.file.Path;
import java.time.LocalDate;
import java.util.*;

public class SiteAssembler {

    public Site assemble(SiteConfig config, List<LoadedFile> files, FrontmatterParser parser) {
        LocalDate today = LocalDate.now(config.timezone());
        List<ParsedDoc> parsed = files.stream()
                .map(f -> new ParsedDoc(f, parser.parse(f.rawContent())))
                .filter(d -> !d.parse.frontmatter().draft())
                .filter(d -> d.parse.frontmatter().date().map(date -> !date.isAfter(today)).orElse(true))
                .toList();

        Map<String, List<ParsedDoc>> bySection = new LinkedHashMap<>();
        for (ParsedDoc d : parsed) {
            String section = sectionFor(d.file.relativePath());
            bySection.computeIfAbsent(section, k -> new ArrayList<>()).add(d);
        }

        Map<String, Page> byPath = new LinkedHashMap<>();
        List<Section> sections = new ArrayList<>();

        for (var entry : bySection.entrySet()) {
            String name = entry.getKey();
            String url = name.equals("root") ? "/" : "/" + name + "/";
            Section section = new Section(new SectionId(url), name, url, List.of());
            List<Page> pages = new ArrayList<>();
            for (ParsedDoc d : entry.getValue()) {
                Page p = toPage(d, section);
                pages.add(p);
                byPath.put(p.url(), p);
            }
            Section withPages = new Section(section.id(), section.name(), section.url(), pages);
            sections.add(withPages);

            // homepage IndexPage for /
            if (name.equals("root")) {
                pages.stream().filter(IndexPage.class::isInstance).findFirst()
                        .ifPresent(idx -> byPath.put("/", idx));
            }
        }

        // ensure / exists even if no _index.md
        byPath.computeIfAbsent("/", k -> defaultHome(byPath));

        return new Site(config, sections, List.of(), Map.copyOf(byPath));
    }

    private static String sectionFor(Path rel) {
        if (rel.getNameCount() == 1) return "root";
        return rel.getName(0).toString();
    }

    private static Page toPage(ParsedDoc d, Section section) {
        String slug = slug(d.file.relativePath().getFileName().toString());
        String url = section.url().equals("/") && slug.equals("index")
                ? "/" : section.url() + slug + "/";
        Frontmatter fm = d.parse.frontmatter();
        if (slug.equals("index") || d.file.relativePath().getFileName().toString().equals("_index.md")) {
            return new IndexPage(new PageId(url), fm, d.parse.body(), url, section, List.of());
        }
        if (section.name().equals("posts")) {
            LocalDate date = fm.date().orElse(LocalDate.now());
            List<Tag> tags = fm.tags().stream().map(t -> new Tag(t, t)).toList();
            return new Post(new PageId(url), fm, d.parse.body(), url, section, date, tags);
        }
        return new StaticPage(new PageId(url), fm, d.parse.body(), url, section);
    }

    private static String slug(String filename) {
        String name = filename.replaceFirst("\\.md$", "");
        if (name.equals("_index")) return "index";
        // strip leading "YYYY-MM-DD-"
        return name.replaceFirst("^\\d{4}-\\d{2}-\\d{2}-", "");
    }

    private static IndexPage defaultHome(Map<String, Page> byPath) {
        Frontmatter fm = new Frontmatter("Home", Optional.empty(), Optional.empty(),
                false, List.of(), Map.of());
        Section root = new Section(new SectionId("/"), "root", "/", List.of());
        return new IndexPage(new PageId("/"), fm, "", "/", root, List.copyOf(byPath.values()));
    }

    private record ParsedDoc(LoadedFile file, ParseResult parse) {}
}
```

- [ ] **Step 15: Implement DefaultContentApi**

`src/main/java/dev/linhvu/my_blog/content/internal/DefaultContentApi.java`:

```java
package dev.linhvu.my_blog.content.internal;

import dev.linhvu.my_blog.content.ContentApi;
import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.content.model.SiteConfig;
import org.springframework.stereotype.Component;

import java.nio.file.Path;

@Component
public class DefaultContentApi implements ContentApi {

    private final MarkdownLoader loader;
    private final FrontmatterParser parser;
    private final SiteAssembler assembler;

    public DefaultContentApi(MarkdownLoader loader, FrontmatterParser parser, SiteAssembler assembler) {
        this.loader = loader;
        this.parser = parser;
        this.assembler = assembler;
    }

    @Override
    public Site loadSite(Path contentRoot, SiteConfig config) {
        var files = loader.loadAll(contentRoot);
        return assembler.assemble(config, files, parser);
    }
}
```

- [ ] **Step 16: Add Spring components for the helpers**

Edit `MarkdownLoader.java`, `FrontmatterParser.java`, `SiteAssembler.java` — add `@Component` annotation at the class level with import:

```java
import org.springframework.stereotype.Component;

@Component
public class MarkdownLoader { ... }
```

(Same for the other two.)

- [ ] **Step 17: Run all content tests**

Run: `./gradlew test --tests "dev.linhvu.my_blog.content.*"`
Expected: PASS — `FrontmatterParserTest`, `MarkdownLoaderTest`, `SiteAssemblerTest`, `SiteModelTests` all green.

- [ ] **Step 18: Run the module structure test**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: PASS.

- [ ] **Step 19: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/content src/test/java/dev/linhvu/my_blog/content
git commit -m "feat(content): MarkdownLoader, SiteAssembler, ContentApi"
```

---

## Task A5: Render module — markdown→HTML, JTE templates, layout dispatch

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/render/RenderApi.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/model/package-info.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/model/RenderedPage.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/internal/MarkdownRenderer.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/internal/SyntaxHighlighter.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/internal/LayoutResolver.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/internal/JteRenderer.java`
- Create: `src/main/java/dev/linhvu/my_blog/render/internal/DefaultRenderApi.java`
- Create: `src/main/jte/base.jte`
- Create: `src/main/jte/post.jte`
- Create: `src/main/jte/page.jte`
- Create: `src/main/jte/list.jte`
- Test: `src/test/java/dev/linhvu/my_blog/render/MarkdownRendererTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/render/LayoutResolverTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/render/DefaultRenderApiTest.java`

- [ ] **Step 1: Create render::model named interface**

`src/main/java/dev/linhvu/my_blog/render/model/package-info.java`:

```java
@org.springframework.modulith.NamedInterface("model")
package dev.linhvu.my_blog.render.model;
```

`src/main/java/dev/linhvu/my_blog/render/model/RenderedPage.java`:

```java
package dev.linhvu.my_blog.render.model;

import dev.linhvu.my_blog.content.model.Page;

public record RenderedPage(Page page, String html) {}
```

- [ ] **Step 2: Write failing test for MarkdownRenderer**

`src/test/java/dev/linhvu/my_blog/render/MarkdownRendererTest.java`:

```java
package dev.linhvu.my_blog.render;

import dev.linhvu.my_blog.render.internal.MarkdownRenderer;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MarkdownRendererTest {

    private final MarkdownRenderer renderer = new MarkdownRenderer();

    @Test
    void rendersBasicMarkdown() {
        String html = renderer.render("# Title\n\nBody text.");
        assertThat(html).contains("<h1>Title</h1>");
        assertThat(html).contains("<p>Body text.</p>");
    }

    @Test
    void rendersFencedCodeBlocksWithLanguageClass() {
        String md = """
                ```java
                int x = 1;
                ```
                """;
        String html = renderer.render(md);
        assertThat(html).contains("<pre><code class=\"language-java\">");
        assertThat(html).contains("int x = 1;");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `./gradlew test --tests MarkdownRendererTest`
Expected: FAIL — `MarkdownRenderer` not found.

- [ ] **Step 4: Implement MarkdownRenderer**

`src/main/java/dev/linhvu/my_blog/render/internal/MarkdownRenderer.java`:

```java
package dev.linhvu.my_blog.render.internal;

import org.commonmark.parser.Parser;
import org.commonmark.renderer.html.HtmlRenderer;
import org.springframework.stereotype.Component;

@Component
public class MarkdownRenderer {

    private final Parser parser = Parser.builder().build();
    private final HtmlRenderer renderer = HtmlRenderer.builder().build();

    public String render(String markdownBody) {
        return renderer.render(parser.parse(markdownBody));
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `./gradlew test --tests MarkdownRendererTest`
Expected: PASS.

- [ ] **Step 6: Implement SyntaxHighlighter (no-op v1, Prism handles client-side)**

`src/main/java/dev/linhvu/my_blog/render/internal/SyntaxHighlighter.java`:

```java
package dev.linhvu.my_blog.render.internal;

import org.springframework.stereotype.Component;

@Component
public class SyntaxHighlighter {
    // v1: no-op. CommonMark already emits class="language-xxx" on <code>;
    // Prism.js (client-side) reads those at page load and highlights.
    // Phase 1.5 replaces this with build-time Shiki/Chroma.
    public String highlight(String htmlWithCodeBlocks) {
        return htmlWithCodeBlocks;
    }
}
```

- [ ] **Step 7: Write failing test for LayoutResolver**

`src/test/java/dev/linhvu/my_blog/render/LayoutResolverTest.java`:

```java
package dev.linhvu.my_blog.render;

import dev.linhvu.my_blog.content.model.*;
import dev.linhvu.my_blog.render.internal.LayoutResolver;
import org.junit.jupiter.api.Test;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class LayoutResolverTest {

    private final LayoutResolver r = new LayoutResolver();
    private final Section sec = new Section(new SectionId("/posts"), "posts", "/posts/", List.of());
    private final Frontmatter fm = new Frontmatter("X", Optional.empty(), Optional.empty(),
            false, List.of(), Map.of());

    @Test
    void postUsesPostTemplate() {
        Page p = new Post(new PageId("/p"), fm, "", "/p/", sec, LocalDate.now(), List.of());
        assertThat(r.layoutFor(p)).isEqualTo("post.jte");
    }

    @Test
    void staticPageUsesPageTemplate() {
        Page p = new StaticPage(new PageId("/about"), fm, "", "/about/", sec);
        assertThat(r.layoutFor(p)).isEqualTo("page.jte");
    }

    @Test
    void indexPageUsesListTemplate() {
        Page p = new IndexPage(new PageId("/"), fm, "", "/", sec, List.of());
        assertThat(r.layoutFor(p)).isEqualTo("list.jte");
    }
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `./gradlew test --tests LayoutResolverTest`
Expected: FAIL.

- [ ] **Step 9: Implement LayoutResolver with sealed-type pattern match**

`src/main/java/dev/linhvu/my_blog/render/internal/LayoutResolver.java`:

```java
package dev.linhvu.my_blog.render.internal;

import dev.linhvu.my_blog.content.model.*;
import org.springframework.stereotype.Component;

@Component
public class LayoutResolver {

    public String layoutFor(Page page) {
        return switch (page) {
            case Post p       -> "post.jte";
            case StaticPage s -> "page.jte";
            case IndexPage i  -> "list.jte";
        };
    }
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `./gradlew test --tests LayoutResolverTest`
Expected: PASS.

- [ ] **Step 11: Create the four JTE templates (minimal, no styling — Phase C polishes them)**

`src/main/jte/base.jte`:

```jte
@import dev.linhvu.my_blog.content.model.Site
@param Site site
@param String title
@param gg.jte.Content content

<!DOCTYPE html>
<html lang="${site.config().locale().toLanguageTag()}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${title} — ${site.config().title()}</title>
  <link rel="stylesheet" href="/prism.css">
  <link rel="alternate" type="application/rss+xml" title="RSS" href="/index.xml">
</head>
<body>
  <header>
    <a href="/"><strong>${site.config().title()}</strong></a>
    <nav>
      <a href="/posts/">posts</a>
      <a href="/about/">about</a>
    </nav>
  </header>
  <main>
    ${content}
  </main>
  <footer>
    <small>© ${java.time.Year.now().toString()} ${site.config().title()}</small>
  </footer>
  <script src="/prism.js" defer></script>
</body>
</html>
```

`src/main/jte/post.jte`:

```jte
@import dev.linhvu.my_blog.content.model.Site
@import dev.linhvu.my_blog.content.model.Post
@param Site site
@param Post page
@param String bodyHtml

@template.base(site = site, title = page.frontmatter().title(), content = @`
<article>
  <h1>${page.frontmatter().title()}</h1>
  <p><time datetime="${page.date().toString()}">${page.date().toString()}</time></p>
  $unsafe{bodyHtml}
</article>
`)
```

`src/main/jte/page.jte`:

```jte
@import dev.linhvu.my_blog.content.model.Site
@import dev.linhvu.my_blog.content.model.StaticPage
@param Site site
@param StaticPage page
@param String bodyHtml

@template.base(site = site, title = page.frontmatter().title(), content = @`
<article>
  <h1>${page.frontmatter().title()}</h1>
  $unsafe{bodyHtml}
</article>
`)
```

`src/main/jte/list.jte`:

```jte
@import dev.linhvu.my_blog.content.model.Site
@import dev.linhvu.my_blog.content.model.IndexPage
@import dev.linhvu.my_blog.content.model.Post
@param Site site
@param IndexPage page
@param String bodyHtml

@template.base(site = site, title = page.frontmatter().title(), content = @`
<article>
  <h1>${page.frontmatter().title()}</h1>
  $unsafe{bodyHtml}
  <ul>
    @for(var p : site.byPath().values())
      @if(p instanceof Post post)
        <li>
          <a href="${post.url()}">${post.frontmatter().title()}</a>
          <small>— ${post.date().toString()}</small>
        </li>
      @endif
    @endfor
  </ul>
</article>
`)
```

- [ ] **Step 12: Implement JteRenderer**

`src/main/java/dev/linhvu/my_blog/render/internal/JteRenderer.java`:

```java
package dev.linhvu.my_blog.render.internal;

import dev.linhvu.my_blog.content.model.*;
import gg.jte.TemplateEngine;
import gg.jte.output.StringOutput;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class JteRenderer {

    private final TemplateEngine engine;
    private final LayoutResolver resolver;

    public JteRenderer(TemplateEngine engine, LayoutResolver resolver) {
        this.engine = engine;
        this.resolver = resolver;
    }

    public String render(Page page, Site site, String bodyHtml) {
        String template = resolver.layoutFor(page);
        var out = new StringOutput();
        engine.render(template, Map.of(
                "site", site,
                "page", page,
                "bodyHtml", bodyHtml
        ), out);
        return out.toString();
    }
}
```

- [ ] **Step 13: Implement DefaultRenderApi**

`src/main/java/dev/linhvu/my_blog/render/RenderApi.java`:

```java
package dev.linhvu.my_blog.render;

import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.render.model.RenderedPage;

import java.util.Map;

public interface RenderApi {
    Map<String, RenderedPage> renderAll(Site site);
}
```

`src/main/java/dev/linhvu/my_blog/render/internal/DefaultRenderApi.java`:

```java
package dev.linhvu.my_blog.render.internal;

import dev.linhvu.my_blog.content.model.Page;
import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.render.RenderApi;
import dev.linhvu.my_blog.render.model.RenderedPage;
import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

@Component
public class DefaultRenderApi implements RenderApi {

    private final MarkdownRenderer markdown;
    private final SyntaxHighlighter highlighter;
    private final JteRenderer jte;

    public DefaultRenderApi(MarkdownRenderer markdown, SyntaxHighlighter highlighter, JteRenderer jte) {
        this.markdown = markdown;
        this.highlighter = highlighter;
        this.jte = jte;
    }

    @Override
    public Map<String, RenderedPage> renderAll(Site site) {
        try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
            Map<String, Future<RenderedPage>> jobs = new LinkedHashMap<>();
            for (var entry : site.byPath().entrySet()) {
                Page page = entry.getValue();
                jobs.put(entry.getKey(), exec.submit(() -> renderOne(site, page)));
            }
            Map<String, RenderedPage> result = new LinkedHashMap<>();
            jobs.forEach((url, fut) -> result.put(url, get(fut)));
            return result;
        }
    }

    private RenderedPage renderOne(Site site, Page page) {
        String body = highlighter.highlight(markdown.render(page.renderedHtml()));
        String html = jte.render(page, site, body);
        return new RenderedPage(page, html);
    }

    private static <T> T get(Future<T> f) {
        try { return f.get(); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); throw new RuntimeException(e); }
        catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

(Note: in `Page.renderedHtml()` we currently store the raw markdown body. The field name reflects the *eventual* state; before the renderer runs, it holds raw md. We could rename later — for now this matches the spec.)

- [ ] **Step 14: Write failing test for DefaultRenderApi end-to-end**

`src/test/java/dev/linhvu/my_blog/render/DefaultRenderApiTest.java`:

```java
package dev.linhvu.my_blog.render;

import dev.linhvu.my_blog.content.ContentApi;
import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.content.model.SiteConfig;
import dev.linhvu.my_blog.render.model.RenderedPage;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.net.URI;
import java.nio.file.Path;
import java.time.ZoneId;
import java.util.Locale;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class DefaultRenderApiTest {

    @Autowired ContentApi content;
    @Autowired RenderApi render;

    @Test
    void rendersAllFixtures() {
        SiteConfig cfg = new SiteConfig(URI.create("http://localhost:8080"), "Test", "x",
                ZoneId.of("UTC"), Locale.ENGLISH);
        Site site = content.loadSite(Path.of("src/test/resources/fixtures"), cfg);

        Map<String, RenderedPage> rendered = render.renderAll(site);

        assertThat(rendered).isNotEmpty();
        rendered.values().forEach(rp -> {
            assertThat(rp.html()).contains("<!DOCTYPE html>");
            assertThat(rp.html()).contains(rp.page().frontmatter().title());
        });
    }
}
```

- [ ] **Step 15: Run test to verify it passes**

Run: `./gradlew test --tests DefaultRenderApiTest`
Expected: PASS — every fixture renders to HTML containing its title and a doctype.

- [ ] **Step 16: Run module structure test**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: PASS.

- [ ] **Step 17: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/render src/main/jte src/test/java/dev/linhvu/my_blog/render
git commit -m "feat(render): markdown→HTML, JTE templates, sealed-type layout dispatch"
```

---

## Task A6: Output module — HTML, RSS, sitemap, robots.txt

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/output/OutputApi.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/internal/HtmlEmitter.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/internal/RssEmitter.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/internal/SitemapEmitter.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/internal/RobotsEmitter.java`
- Create: `src/main/java/dev/linhvu/my_blog/output/internal/DefaultOutputApi.java`
- Test: `src/test/java/dev/linhvu/my_blog/output/HtmlEmitterTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/output/RssEmitterTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/output/SitemapEmitterTest.java`

- [ ] **Step 1: Create OutputApi facade**

`src/main/java/dev/linhvu/my_blog/output/OutputApi.java`:

```java
package dev.linhvu.my_blog.output;

import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.render.model.RenderedPage;

import java.nio.file.Path;
import java.util.Map;

public interface OutputApi {
    void emitAll(Site site, Map<String, RenderedPage> rendered, Path distRoot);
}
```

- [ ] **Step 2: Write failing test for HtmlEmitter**

`src/test/java/dev/linhvu/my_blog/output/HtmlEmitterTest.java`:

```java
package dev.linhvu.my_blog.output;

import dev.linhvu.my_blog.content.model.*;
import dev.linhvu.my_blog.output.internal.HtmlEmitter;
import dev.linhvu.my_blog.render.model.RenderedPage;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.LocalDate;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class HtmlEmitterTest {

    private final HtmlEmitter emitter = new HtmlEmitter();

    @Test
    void writesIndexHtmlAtUrl(@TempDir Path dist) throws Exception {
        Section sec = new Section(new SectionId("/posts"), "posts", "/posts/", List.of());
        Frontmatter fm = new Frontmatter("Hello", Optional.empty(), Optional.empty(),
                false, List.of(), Map.of());
        Post p = new Post(new PageId("/posts/hello/"), fm, "", "/posts/hello/", sec,
                LocalDate.of(2026, 1, 1), List.of());
        emitter.emit(Map.of("/posts/hello/", new RenderedPage(p, "<html>HI</html>")), dist);

        Path file = dist.resolve("posts/hello/index.html");
        assertThat(Files.exists(file)).isTrue();
        assertThat(Files.readString(file)).contains("HI");
    }

    @Test
    void writesRootIndexHtml(@TempDir Path dist) throws Exception {
        Section sec = new Section(new SectionId("/"), "root", "/", List.of());
        Frontmatter fm = new Frontmatter("Home", Optional.empty(), Optional.empty(),
                false, List.of(), Map.of());
        IndexPage i = new IndexPage(new PageId("/"), fm, "", "/", sec, List.of());
        emitter.emit(Map.of("/", new RenderedPage(i, "<html>Home</html>")), dist);

        assertThat(Files.exists(dist.resolve("index.html"))).isTrue();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `./gradlew test --tests HtmlEmitterTest`
Expected: FAIL — `HtmlEmitter` not found.

- [ ] **Step 4: Implement HtmlEmitter**

`src/main/java/dev/linhvu/my_blog/output/internal/HtmlEmitter.java`:

```java
package dev.linhvu.my_blog.output.internal;

import dev.linhvu.my_blog.render.model.RenderedPage;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

@Component
public class HtmlEmitter {

    public void emit(Map<String, RenderedPage> rendered, Path distRoot) {
        rendered.forEach((url, rp) -> {
            String relative = url.startsWith("/") ? url.substring(1) : url;
            Path out = relative.isEmpty()
                    ? distRoot.resolve("index.html")
                    : distRoot.resolve(relative).resolve("index.html");
            try {
                Files.createDirectories(out.getParent());
                Files.writeString(out, rp.html());
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        });
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `./gradlew test --tests HtmlEmitterTest`
Expected: PASS.

- [ ] **Step 6: Write failing test for RssEmitter**

`src/test/java/dev/linhvu/my_blog/output/RssEmitterTest.java`:

```java
package dev.linhvu.my_blog.output;

import dev.linhvu.my_blog.content.model.*;
import dev.linhvu.my_blog.output.internal.RssEmitter;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.net.URI;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

class RssEmitterTest {

    private final RssEmitter emitter = new RssEmitter();

    @Test
    void writesIndexXmlWithChannelAndItem(@TempDir Path dist) throws Exception {
        Section sec = new Section(new SectionId("/posts"), "posts", "/posts/", List.of());
        Frontmatter fm = new Frontmatter("Hello", Optional.of("desc"), Optional.empty(),
                false, List.of(), Map.of());
        Post p = new Post(new PageId("/posts/hello/"), fm, "<p>body</p>", "/posts/hello/", sec,
                LocalDate.of(2026, 1, 1), List.of());
        SiteConfig cfg = new SiteConfig(URI.create("https://blogs.linhvu.dev"),
                "linhvu", "desc", ZoneId.of("UTC"), Locale.ENGLISH);
        Site site = new Site(cfg, List.of(sec), List.of(), Map.of("/posts/hello/", p));

        emitter.emit(site, dist);

        String xml = Files.readString(dist.resolve("index.xml"));
        assertThat(xml).contains("<rss");
        assertThat(xml).contains("<title>linhvu</title>");
        assertThat(xml).contains("<link>https://blogs.linhvu.dev</link>");
        assertThat(xml).contains("https://blogs.linhvu.dev/posts/hello/");
        assertThat(xml).contains("Hello");
    }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `./gradlew test --tests RssEmitterTest`
Expected: FAIL.

- [ ] **Step 8: Implement RssEmitter**

`src/main/java/dev/linhvu/my_blog/output/internal/RssEmitter.java`:

```java
package dev.linhvu.my_blog.output.internal;

import dev.linhvu.my_blog.content.model.Post;
import dev.linhvu.my_blog.content.model.Site;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.format.DateTimeFormatter;
import java.util.Comparator;
import java.util.Locale;

@Component
public class RssEmitter {

    private static final DateTimeFormatter RFC822 =
            DateTimeFormatter.ofPattern("EEE, dd MMM yyyy 00:00:00 Z", Locale.ENGLISH);

    public void emit(Site site, Path distRoot) {
        StringBuilder b = new StringBuilder();
        b.append("<?xml version=\"1.0\" encoding=\"utf-8\"?>\n");
        b.append("<rss version=\"2.0\"><channel>\n");
        b.append("<title>").append(escape(site.config().title())).append("</title>\n");
        b.append("<link>").append(site.config().baseUrl()).append("</link>\n");
        b.append("<description>").append(escape(site.config().description())).append("</description>\n");

        site.byPath().values().stream()
                .filter(Post.class::isInstance).map(Post.class::cast)
                .sorted(Comparator.comparing(Post::date).reversed())
                .forEach(p -> {
                    b.append("<item>\n");
                    b.append("<title>").append(escape(p.frontmatter().title())).append("</title>\n");
                    b.append("<link>").append(site.config().baseUrl()).append(p.url()).append("</link>\n");
                    b.append("<guid>").append(site.config().baseUrl()).append(p.url()).append("</guid>\n");
                    b.append("<pubDate>").append(p.date().atStartOfDay(site.config().timezone()).format(RFC822))
                            .append("</pubDate>\n");
                    p.frontmatter().description().ifPresent(d ->
                            b.append("<description>").append(escape(d)).append("</description>\n"));
                    b.append("</item>\n");
                });

        b.append("</channel></rss>\n");
        try {
            Files.createDirectories(distRoot);
            Files.writeString(distRoot.resolve("index.xml"), b.toString());
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private static String escape(String s) {
        return s.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;");
    }
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `./gradlew test --tests RssEmitterTest`
Expected: PASS.

- [ ] **Step 10: Write failing test for SitemapEmitter**

`src/test/java/dev/linhvu/my_blog/output/SitemapEmitterTest.java`:

```java
package dev.linhvu.my_blog.output;

import dev.linhvu.my_blog.content.model.*;
import dev.linhvu.my_blog.output.internal.SitemapEmitter;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.net.URI;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

class SitemapEmitterTest {

    private final SitemapEmitter emitter = new SitemapEmitter();

    @Test
    void writesSitemapXmlWithEveryUrl(@TempDir Path dist) throws Exception {
        Section sec = new Section(new SectionId("/posts"), "posts", "/posts/", List.of());
        Frontmatter fm = new Frontmatter("X", Optional.empty(), Optional.empty(),
                false, List.of(), Map.of());
        Post p = new Post(new PageId("/posts/x/"), fm, "", "/posts/x/", sec,
                LocalDate.of(2026, 1, 1), List.of());
        IndexPage home = new IndexPage(new PageId("/"), fm, "", "/", sec, List.of());

        SiteConfig cfg = new SiteConfig(URI.create("https://blogs.linhvu.dev"),
                "t", "d", ZoneId.of("UTC"), Locale.ENGLISH);
        Site site = new Site(cfg, List.of(sec), List.of(),
                Map.of("/", home, "/posts/x/", p));

        emitter.emit(site, dist);

        String xml = Files.readString(dist.resolve("sitemap.xml"));
        assertThat(xml).contains("<urlset");
        assertThat(xml).contains("https://blogs.linhvu.dev/");
        assertThat(xml).contains("https://blogs.linhvu.dev/posts/x/");
    }
}
```

- [ ] **Step 11: Run test to verify it fails, then implement**

Run: `./gradlew test --tests SitemapEmitterTest`
Expected: FAIL.

`src/main/java/dev/linhvu/my_blog/output/internal/SitemapEmitter.java`:

```java
package dev.linhvu.my_blog.output.internal;

import dev.linhvu.my_blog.content.model.Site;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;

@Component
public class SitemapEmitter {

    public void emit(Site site, Path distRoot) {
        StringBuilder b = new StringBuilder();
        b.append("<?xml version=\"1.0\" encoding=\"utf-8\"?>\n");
        b.append("<urlset xmlns=\"http://www.sitemaps.org/schemas/sitemap/0.9\">\n");
        site.byPath().keySet().forEach(url ->
                b.append("<url><loc>").append(site.config().baseUrl()).append(url).append("</loc></url>\n"));
        b.append("</urlset>\n");

        try {
            Files.createDirectories(distRoot);
            Files.writeString(distRoot.resolve("sitemap.xml"), b.toString());
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
}
```

Run: `./gradlew test --tests SitemapEmitterTest`
Expected: PASS.

- [ ] **Step 12: Implement RobotsEmitter (no test — single-line file)**

`src/main/java/dev/linhvu/my_blog/output/internal/RobotsEmitter.java`:

```java
package dev.linhvu.my_blog.output.internal;

import dev.linhvu.my_blog.content.model.Site;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;

@Component
public class RobotsEmitter {
    public void emit(Site site, Path distRoot) {
        String body = """
                User-agent: *
                Allow: /
                Sitemap: %ssitemap.xml
                """.formatted(site.config().baseUrl());
        try {
            Files.createDirectories(distRoot);
            Files.writeString(distRoot.resolve("robots.txt"), body);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
}
```

- [ ] **Step 13: Implement DefaultOutputApi**

`src/main/java/dev/linhvu/my_blog/output/internal/DefaultOutputApi.java`:

```java
package dev.linhvu.my_blog.output.internal;

import dev.linhvu.my_blog.content.model.Site;
import dev.linhvu.my_blog.output.OutputApi;
import dev.linhvu.my_blog.render.model.RenderedPage;
import org.springframework.stereotype.Component;

import java.nio.file.Path;
import java.util.Map;

@Component
public class DefaultOutputApi implements OutputApi {

    private final HtmlEmitter html;
    private final RssEmitter rss;
    private final SitemapEmitter sitemap;
    private final RobotsEmitter robots;

    public DefaultOutputApi(HtmlEmitter html, RssEmitter rss, SitemapEmitter sitemap, RobotsEmitter robots) {
        this.html = html; this.rss = rss; this.sitemap = sitemap; this.robots = robots;
    }

    @Override
    public void emitAll(Site site, Map<String, RenderedPage> rendered, Path distRoot) {
        html.emit(rendered, distRoot);
        rss.emit(site, distRoot);
        sitemap.emit(site, distRoot);
        robots.emit(site, distRoot);
    }
}
```

- [ ] **Step 14: Run all output tests**

Run: `./gradlew test --tests "dev.linhvu.my_blog.output.*"`
Expected: PASS.

- [ ] **Step 15: Run module structure test**

Run: `./gradlew test --tests ModuleStructureTests`
Expected: PASS.

- [ ] **Step 16: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/output src/test/java/dev/linhvu/my_blog/output
git commit -m "feat(output): HTML, RSS, sitemap, robots emitters"
```

---

## Task A7: Assets module — copy static files

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/assets/AssetsApi.java`
- Create: `src/main/java/dev/linhvu/my_blog/assets/internal/DefaultAssetsApi.java`
- Test: `src/test/java/dev/linhvu/my_blog/assets/DefaultAssetsApiTest.java`

- [ ] **Step 1: Write failing test**

`src/test/java/dev/linhvu/my_blog/assets/DefaultAssetsApiTest.java`:

```java
package dev.linhvu.my_blog.assets;

import dev.linhvu.my_blog.assets.internal.DefaultAssetsApi;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultAssetsApiTest {

    private final AssetsApi assets = new DefaultAssetsApi();

    @Test
    void copiesStaticFilesPreservingTree(@TempDir Path tmp) throws Exception {
        Path src = tmp.resolve("static");
        Path dist = tmp.resolve("dist");
        Files.createDirectories(src.resolve("nested"));
        Files.writeString(src.resolve("a.css"), "body{color:red}");
        Files.writeString(src.resolve("nested/b.js"), "console.log(1)");

        assets.copyStatic(src, dist);

        assertThat(Files.readString(dist.resolve("a.css"))).contains("color:red");
        assertThat(Files.readString(dist.resolve("nested/b.js"))).contains("console.log");
    }

    @Test
    void missingStaticDirIsNoOp(@TempDir Path tmp) {
        Path src = tmp.resolve("does-not-exist");
        Path dist = tmp.resolve("dist");
        assets.copyStatic(src, dist);  // should not throw
        assertThat(Files.exists(dist)).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests DefaultAssetsApiTest`
Expected: FAIL.

- [ ] **Step 3: Implement AssetsApi + DefaultAssetsApi**

`src/main/java/dev/linhvu/my_blog/assets/AssetsApi.java`:

```java
package dev.linhvu.my_blog.assets;

import java.nio.file.Path;

public interface AssetsApi {
    void copyStatic(Path staticRoot, Path distRoot);
}
```

`src/main/java/dev/linhvu/my_blog/assets/internal/DefaultAssetsApi.java`:

```java
package dev.linhvu.my_blog.assets.internal;

import dev.linhvu.my_blog.assets.AssetsApi;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.*;
import java.util.stream.Stream;

@Component
public class DefaultAssetsApi implements AssetsApi {

    @Override
    public void copyStatic(Path staticRoot, Path distRoot) {
        if (!Files.exists(staticRoot)) return;
        try (Stream<Path> stream = Files.walk(staticRoot)) {
            stream.forEach(src -> copy(src, staticRoot, distRoot));
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private static void copy(Path src, Path staticRoot, Path distRoot) {
        try {
            Path dst = distRoot.resolve(staticRoot.relativize(src));
            if (Files.isDirectory(src)) {
                Files.createDirectories(dst);
            } else {
                Files.createDirectories(dst.getParent());
                Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
            }
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests DefaultAssetsApiTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/linhvu/my_blog/assets src/test/java/dev/linhvu/my_blog/assets
git commit -m "feat(assets): copy static directory to dist"
```

---

## Task A8: Engine — SiteBuilder, BuildSiteCommand, `buildSite` Gradle task, sample content

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/engine/SiteBuilder.java`
- Create: `src/main/java/dev/linhvu/my_blog/engine/BuildResult.java`
- Create: `src/main/java/dev/linhvu/my_blog/engine/cli/BuildSiteCommand.java`
- Create: `content/_index.md`
- Create: `content/posts/2026-05-07-hello-world.md`
- Create: `content/pages/about.md`
- Create: `static/prism.css`
- Create: `static/prism.js`
- Modify: `build.gradle`
- Test: `src/test/java/dev/linhvu/my_blog/engine/SiteBuilderTest.java`
- Test: `src/test/java/dev/linhvu/my_blog/engine/BuildSiteSmokeTest.java`

- [ ] **Step 1: Create real content (the first article!)**

`content/_index.md`:

```markdown
---
title: "Home"
description: "Notes on Java by Linh Vu"
---

Welcome to my blog about Java, the JVM, and modern backend development.
```

`content/posts/2026-05-07-hello-world.md`:

```markdown
---
title: "Hello, world"
date: 2026-05-07
tags: [meta]
description: "First post on this brand-new Spring Boot 4 / Java 25 blog."
draft: false
---

# Hello, world

This is the first article on my new blog, built on **Spring Boot 4** and **Java 25**.

```java
sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double s) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Square(double s) -> s * s;
    };
}
```

More to come.
```

`content/pages/about.md`:

```markdown
---
title: "About"
description: "About Linh Vu"
---

I write about Java backend topics. This site is built on Spring Boot 4 + Java 25.
```

- [ ] **Step 2: Add Prism.js assets (downloaded from CDN copies, kept local for offline dev)**

`static/prism.css` — paste the contents of [prism.css 1.30 default theme](https://cdn.jsdelivr.net/npm/prismjs@1.30.0/themes/prism.min.css). To save: `curl -sLo static/prism.css https://cdn.jsdelivr.net/npm/prismjs@1.30.0/themes/prism.min.css`

`static/prism.js` — minimal autoloader:

```javascript
// prism.min.js + autoloader
// Download: curl -sLo static/prism.js https://cdn.jsdelivr.net/npm/prismjs@1.30.0/prism.min.js
// Then: curl -sL https://cdn.jsdelivr.net/npm/prismjs@1.30.0/components/prism-java.min.js >> static/prism.js
```

Run those two `curl` commands now to actually download the files. Verify both exist:

```bash
ls -la static/prism.css static/prism.js
```

- [ ] **Step 3: Create BuildResult record**

`src/main/java/dev/linhvu/my_blog/engine/BuildResult.java`:

```java
package dev.linhvu.my_blog.engine;

import java.time.Instant;

public record BuildResult(int pageCount, Instant builtAt) {}
```

- [ ] **Step 4: Create SiteBuilder orchestrator**

`src/main/java/dev/linhvu/my_blog/engine/SiteBuilder.java`:

```java
package dev.linhvu.my_blog.engine;

import dev.linhvu.my_blog.assets.AssetsApi;
import dev.linhvu.my_blog.content.ContentApi;
import dev.linhvu.my_blog.content.model.SiteConfig;
import dev.linhvu.my_blog.output.OutputApi;
import dev.linhvu.my_blog.render.RenderApi;
import org.springframework.stereotype.Component;

import java.nio.file.Path;
import java.time.Instant;

@Component
public class SiteBuilder {

    private final ContentApi content;
    private final RenderApi render;
    private final OutputApi output;
    private final AssetsApi assets;

    public SiteBuilder(ContentApi content, RenderApi render, OutputApi output, AssetsApi assets) {
        this.content = content;
        this.render = render;
        this.output = output;
        this.assets = assets;
    }

    public BuildResult build(SiteConfig config, Path contentRoot, Path staticRoot, Path distRoot) {
        var site = content.loadSite(contentRoot, config);
        var rendered = render.renderAll(site);
        output.emitAll(site, rendered, distRoot);
        assets.copyStatic(staticRoot, distRoot);
        return new BuildResult(site.byPath().size(), Instant.now());
    }
}
```

- [ ] **Step 5: Write failing test for BuildSiteSmokeTest (E2E)**

`src/test/java/dev/linhvu/my_blog/engine/BuildSiteSmokeTest.java`:

```java
package dev.linhvu.my_blog.engine;

import dev.linhvu.my_blog.content.model.SiteConfig;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.net.URI;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.ZoneId;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class BuildSiteSmokeTest {

    @Autowired SiteBuilder builder;

    @Test
    void buildsRealContentToTempDist(@TempDir Path tmp) throws Exception {
        Path dist = tmp.resolve("dist");
        SiteConfig cfg = new SiteConfig(URI.create("http://localhost:8080"),
                "linhvu", "test build", ZoneId.of("UTC"), Locale.ENGLISH);

        BuildResult result = builder.build(cfg, Path.of("content"), Path.of("static"), dist);

        assertThat(result.pageCount()).isGreaterThan(0);
        assertThat(Files.exists(dist.resolve("index.html"))).isTrue();
        assertThat(Files.readString(dist.resolve("index.html"))).contains("<!DOCTYPE html>");
        assertThat(Files.exists(dist.resolve("posts/hello-world/index.html"))).isTrue();
        assertThat(Files.readString(dist.resolve("posts/hello-world/index.html")))
                .contains("Hello, world");
        assertThat(Files.exists(dist.resolve("about/index.html"))).isTrue();
        assertThat(Files.exists(dist.resolve("index.xml"))).isTrue();
        assertThat(Files.exists(dist.resolve("sitemap.xml"))).isTrue();
        assertThat(Files.exists(dist.resolve("robots.txt"))).isTrue();
    }
}
```

- [ ] **Step 6: Run test to verify it passes (everything is wired)**

Run: `./gradlew test --tests BuildSiteSmokeTest`
Expected: PASS — full pipeline works end-to-end with real `content/` and `static/`.

- [ ] **Step 7: Implement BuildSiteCommand (CLI entry point)**

`src/main/java/dev/linhvu/my_blog/engine/cli/BuildSiteCommand.java`:

```java
package dev.linhvu.my_blog.engine.cli;

import dev.linhvu.my_blog.content.model.SiteConfig;
import dev.linhvu.my_blog.engine.BuildResult;
import dev.linhvu.my_blog.engine.SiteBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.nio.file.Path;
import java.time.ZoneId;
import java.util.Locale;

@Component
@Profile("build")
public class BuildSiteCommand implements CommandLineRunner {

    private final SiteBuilder builder;

    @Value("${myblog.content-root}") private String contentRoot;
    @Value("${myblog.static-root}")  private String staticRoot;
    @Value("${myblog.dist-root}")    private String distRoot;
    @Value("${myblog.base-url}")     private String baseUrl;
    @Value("${myblog.site-title}")   private String title;

    public BuildSiteCommand(SiteBuilder builder) {
        this.builder = builder;
    }

    @Override
    public void run(String... args) {
        SiteConfig cfg = new SiteConfig(URI.create(baseUrl), title, "",
                ZoneId.of("UTC"), Locale.ENGLISH);
        BuildResult r = builder.build(cfg, Path.of(contentRoot), Path.of(staticRoot), Path.of(distRoot));
        System.out.printf("[buildSite] %d pages → %s%n", r.pageCount(), distRoot);
    }
}
```

- [ ] **Step 8: Add the `buildSite` Gradle task**

Append to `build.gradle`:

```groovy
tasks.register('buildSite', JavaExec) {
    group = 'build'
    description = 'Build the static site to dist/'
    dependsOn classes
    classpath = sourceSets.main.runtimeClasspath
    mainClass = 'dev.linhvu.my_blog.MyBlogApplication'
    args = ['--spring.profiles.active=build', '--spring.main.web-application-type=none']
}
```

- [ ] **Step 9: Run `./gradlew buildSite` and verify output**

Run: `./gradlew buildSite`
Expected: build succeeds, output ends with `[buildSite] N pages → ./dist`.

Verify:

```bash
ls dist/
ls dist/posts/hello-world/
cat dist/index.html | head -20
```

`dist/index.html`, `dist/posts/hello-world/index.html`, `dist/about/index.html`, `dist/index.xml`, `dist/sitemap.xml`, `dist/robots.txt`, `dist/prism.css`, `dist/prism.js` should all exist.

- [ ] **Step 10: Run the full test suite to confirm nothing regressed**

Run: `./gradlew test`
Expected: PASS. All module tests + smoke + boundary tests green.

- [ ] **Step 11: Commit**

```bash
git add content static src/main/java/dev/linhvu/my_blog/engine src/test/java/dev/linhvu/my_blog/engine build.gradle
git commit -m "feat(engine): SiteBuilder + buildSite Gradle task; first article live (locally)"
```

---

### 🎯 Phase A milestone

Run `./gradlew buildSite`, then `cd dist && python3 -m http.server 8000`, then open `http://localhost:8000/`. Your blog should render with the homepage list, the hello-world post (with Java syntax highlighting via Prism), and the About page.

---

# Phase B — Dev server & hot reload

## Task B1: DevServer with file watcher + SSE live reload

**Files:**
- Create: `src/main/java/dev/linhvu/my_blog/engine/DevServer.java`
- Create: `src/main/java/dev/linhvu/my_blog/engine/live/LiveReloadEndpoint.java`
- Create: `src/main/resources/static/livereload.js`
- Modify: `src/main/jte/base.jte`
- Test: manual (file watcher requires real filesystem events)

- [ ] **Step 1: Implement DevServer with WatchService**

`src/main/java/dev/linhvu/my_blog/engine/DevServer.java`:

```java
package dev.linhvu.my_blog.engine;

import dev.linhvu.my_blog.content.model.SiteConfig;
import dev.linhvu.my_blog.engine.live.LiveReloadEndpoint;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.net.URI;
import java.nio.file.*;
import java.time.ZoneId;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;
import java.util.concurrent.Executors;

@Component
@Profile("!build")
public class DevServer {

    private final SiteBuilder builder;
    private final LiveReloadEndpoint live;

    @Value("${myblog.content-root}") private String contentRoot;
    @Value("${myblog.static-root}")  private String staticRoot;
    @Value("${myblog.dist-root}")    private String distRoot;
    @Value("${myblog.base-url}")     private String baseUrl;
    @Value("${myblog.site-title}")   private String title;

    public DevServer(SiteBuilder builder, LiveReloadEndpoint live) {
        this.builder = builder;
        this.live = live;
    }

    @EventListener(ContextRefreshedEvent.class)
    public void start() {
        rebuild();
        Executors.newVirtualThreadPerTaskExecutor().submit(this::watch);
    }

    private void rebuild() {
        var cfg = new SiteConfig(URI.create(baseUrl), title, "",
                ZoneId.of("UTC"), Locale.ENGLISH);
        builder.build(cfg, Path.of(contentRoot), Path.of(staticRoot), Path.of(distRoot));
        live.broadcastReload();
    }

    private void watch() {
        try (WatchService ws = FileSystems.getDefault().newWatchService()) {
            registerRecursive(ws, Path.of(contentRoot));
            registerRecursive(ws, Path.of(staticRoot));
            registerRecursive(ws, Path.of("src/main/jte"));
            while (true) {
                WatchKey key = ws.take();
                for (WatchEvent<?> ignored : key.pollEvents()) { /* drain */ }
                Thread.sleep(100); // debounce
                rebuild();
                key.reset();
            }
        } catch (IOException | InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private static void registerRecursive(WatchService ws, Path root) throws IOException {
        if (!Files.exists(root)) return;
        Files.walkFileTree(root, new SimpleFileVisitor<>() {
            @Override public FileVisitResult preVisitDirectory(Path dir, java.nio.file.attribute.BasicFileAttributes a) throws IOException {
                dir.register(ws, StandardWatchEventKinds.ENTRY_CREATE,
                        StandardWatchEventKinds.ENTRY_MODIFY, StandardWatchEventKinds.ENTRY_DELETE);
                return FileVisitResult.CONTINUE;
            }
        });
    }
}
```

- [ ] **Step 2: Implement LiveReloadEndpoint (SSE)**

`src/main/java/dev/linhvu/my_blog/engine/live/LiveReloadEndpoint.java`:

```java
package dev.linhvu.my_blog.engine.live;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@Component
@RestController
public class LiveReloadEndpoint {

    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();

    @GetMapping(value = "/__live", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe() {
        SseEmitter e = new SseEmitter(0L);
        emitters.add(e);
        e.onCompletion(() -> emitters.remove(e));
        e.onTimeout(() -> emitters.remove(e));
        return e;
    }

    public void broadcastReload() {
        for (SseEmitter e : List.copyOf(emitters)) {
            try {
                e.send(SseEmitter.event().name("reload").data("ok"));
            } catch (IOException ex) {
                emitters.remove(e);
            }
        }
    }
}
```

- [ ] **Step 3: Create the SSE client script**

`src/main/resources/static/livereload.js`:

```javascript
(function () {
  const es = new EventSource('/__live');
  es.addEventListener('reload', () => location.reload());
  es.onerror = () => setTimeout(() => location.reload(), 2000);
})();
```

- [ ] **Step 4: Inject livereload.js into base.jte (dev only)**

Modify `src/main/jte/base.jte` — add inside `<head>` before `</head>`:

```jte
@if(!"prod".equals(System.getProperty("spring.profiles.active")))
<script src="/livereload.js" defer></script>
@endif
```

- [ ] **Step 5: Configure Spring MVC to serve dist/ as static root in dev**

Create `src/main/java/dev/linhvu/my_blog/engine/DevWebConfig.java`:

```java
package dev.linhvu.my_blog.engine;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
@Profile("!build")
public class DevWebConfig implements WebMvcConfigurer {

    @Value("${myblog.dist-root}") private String distRoot;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry r) {
        r.addResourceHandler("/**")
                .addResourceLocations("file:" + distRoot + "/")
                .resourceChain(false);  // no caching in dev
    }
}
```

- [ ] **Step 6: Run `./gradlew bootRun`**

Run: `./gradlew bootRun`
Expected: app starts on port 8080, logs `[buildSite]`-style output, then keeps running.

- [ ] **Step 7: Verify in browser**

Open `http://localhost:8080/` — homepage shows.
Open another tab: `http://localhost:8080/posts/hello-world/` — post shows with syntax highlighting.

- [ ] **Step 8: Verify hot reload manually**

While `bootRun` is running, edit `content/posts/2026-05-07-hello-world.md` (change a word in the body) and save. Browser should reload within ~500ms and show the change.

- [ ] **Step 9: Stop bootRun and commit**

Ctrl-C in the bootRun terminal.

```bash
git add src/main/java/dev/linhvu/my_blog/engine/DevServer.java \
        src/main/java/dev/linhvu/my_blog/engine/DevWebConfig.java \
        src/main/java/dev/linhvu/my_blog/engine/live \
        src/main/resources/static/livereload.js \
        src/main/jte/base.jte
git commit -m "feat(engine): DevServer with file-watch + SSE live reload"
```

---

## Task B2: Document the dev workflow

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write a short README documenting the workflow**

`README.md`:

```markdown
# my-blog

Build-time SSG on Spring Boot 4 + Java 25. See `docs/superpowers/specs/2026-05-07-blog-design.md`.

## Local dev

```bash
./gradlew bootRun       # http://localhost:8080, hot reload on .md / .jte / static/
./gradlew buildSite     # one-shot build to ./dist
./gradlew test          # unit + module-boundary + snapshot + smoke tests
```

## Authoring

Drop new posts in `content/posts/YYYY-MM-DD-<slug>.md` with frontmatter:

```yaml
---
title: "Post title"
date: 2026-05-07
tags: [java25]
description: "One-line summary"
draft: false
---
```

## Deploy

Push to `main`; GitHub Actions builds and deploys to `https://blogs.linhvu.dev` (see `.github/workflows/deploy.yml`).
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: README with dev workflow"
```

---

### 🎯 Phase B milestone

`./gradlew bootRun` runs a dev server on http://localhost:8080. Editing a `.md`, `.jte`, or `static/*` file triggers a rebuild and the browser auto-reloads in under 500 ms.

---

# Phase C — Visual design via DESIGN.md flow

## Task C1: Draft DESIGN.md inspired by awesome-design-md

**Files:**
- Create: `DESIGN.md`

- [ ] **Step 1: Pick a reference design from awesome-design-md**

Browse https://github.com/VoltAgent/awesome-design-md/tree/main/design-md and skim 2-3 candidates that fit a personal Java technical blog. Recommended starting points:

- `linear/DESIGN.md` — clean, minimal, technical
- `vercel/DESIGN.md` — dark-friendly, developer aesthetic
- `stripe/DESIGN.md` — editorial, polished

Pick one. (Recommendation: **Vercel** — closest match to "Java technical blog" + dark mode out of the box.)

- [ ] **Step 2: Adapt the chosen DESIGN.md to this project**

Copy the chosen file's contents to `DESIGN.md` at repo root. Then edit:

- Replace product/brand mentions with "linhvu — notes on Java"
- Adjust the colour palette to match your taste (light/dark or auto)
- Trim any sections irrelevant to a blog (e.g., e-commerce buttons, marketing components)
- Keep: typography scale, colour tokens, spacing scale, border radius, button styles, link styles, code block styling

The final `DESIGN.md` should describe:
- Brand voice and personality (1-2 paragraphs)
- Colour tokens (CSS custom properties)
- Typography (families, scales, line heights)
- Spacing scale
- Layout primitives (max-width, gutters)
- Component patterns: header, nav, article, post-list item, code block, footer, link, button

- [ ] **Step 3: Commit the DESIGN.md draft**

```bash
git add DESIGN.md
git commit -m "design: draft DESIGN.md (adapted from awesome-design-md)"
```

---

## Task C2: Generate CSS + refine JTE templates from DESIGN.md

**Files:**
- Create: `static/site.css`
- Modify: `src/main/jte/base.jte`
- Modify: `src/main/jte/post.jte`
- Modify: `src/main/jte/page.jte`
- Modify: `src/main/jte/list.jte`

This task is **manual + Claude-assisted** — the agentic worker invokes the `frontend-design` skill with the DESIGN.md as input.

- [ ] **Step 1: Invoke the frontend-design skill**

Open a fresh Claude session. Provide:

> Use the `frontend-design` skill. Generate a single `static/site.css` and updated JTE templates (`base.jte`, `post.jte`, `page.jte`, `list.jte`) for this blog. Apply the design tokens, typography scale, and component patterns from `DESIGN.md`. Constraints:
> - JTE syntax only (no React/Vue/Tailwind classes — vanilla CSS).
> - Light + dark via `prefers-color-scheme` with manual toggle button in the header.
> - No JS framework. Toggle is ~10 lines of vanilla JS.
> - Keep the existing `<link rel="stylesheet" href="/prism.css">` and Prism `<script>` tags.
> - Code blocks must inherit from DESIGN.md but stay compatible with Prism.js class names (`language-java`, etc.).

Save the generated files. Review and edit if needed.

- [ ] **Step 2: Update base.jte to include site.css**

Inside `<head>` of `base.jte`, after the prism.css link, add:

```jte
<link rel="stylesheet" href="/site.css">
```

- [ ] **Step 3: Run the dev server and review**

Run: `./gradlew bootRun`

Open http://localhost:8080/ — homepage should reflect the new design.
Open http://localhost:8080/posts/hello-world/ — article view styled.
Open http://localhost:8080/about/ — page styled.

Iterate on `site.css` and templates until satisfied. Each save triggers hot reload.

- [ ] **Step 4: Run all tests to confirm nothing regressed**

Run: `./gradlew test`
Expected: PASS. (Snapshot tests may need updating — accept the new HTML if the change is intentional.)

- [ ] **Step 5: Commit**

```bash
git add static/site.css src/main/jte
git commit -m "design: implement DESIGN.md tokens in CSS + JTE templates"
```

---

### 🎯 Phase C milestone

Site looks intentional — colours, type, spacing all derived from DESIGN.md. Light/dark toggle works.

---

# Phase D — AWS infrastructure (CDK Java)

## Task D1: Add `infra/` subproject with AWS CDK Java setup

**Files:**
- Create: `infra/build.gradle`
- Create: `infra/cdk.json`
- Create: `infra/src/main/java/dev/linhvu/my_blog/infra/InfraApp.java`
- Modify: `settings.gradle`

- [ ] **Step 1: Update root settings.gradle to include infra**

Replace `settings.gradle` with:

```groovy
rootProject.name = 'my-blog'
include 'infra'
```

- [ ] **Step 2: Create infra/build.gradle**

`infra/build.gradle`:

```groovy
plugins {
    id 'java'
    id 'application'
}

repositories {
    mavenCentral()
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(25) }
}

dependencies {
    implementation 'software.amazon.awscdk:aws-cdk-lib:2.158.0'
    implementation 'software.constructs:constructs:10.4.2'
}

application {
    mainClass = 'dev.linhvu.my_blog.infra.InfraApp'
}
```

- [ ] **Step 3: Create cdk.json**

`infra/cdk.json`:

```json
{
  "app": "../gradlew :infra:run -q",
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": true,
    "@aws-cdk/aws-cloudfront:defaultSecurityPolicyTLSv1.2_2021": true
  }
}
```

- [ ] **Step 4: Create InfraApp entry point (skeleton — stack added in next task)**

`infra/src/main/java/dev/linhvu/my_blog/infra/InfraApp.java`:

```java
package dev.linhvu.my_blog.infra;

import software.amazon.awscdk.App;
import software.amazon.awscdk.Environment;
import software.amazon.awscdk.StackProps;

public class InfraApp {
    public static void main(String[] args) {
        App app = new App();
        new BlogStack(app, "BlogStack", StackProps.builder()
                .env(Environment.builder().region("us-east-1").build())
                .build());
        app.synth();
    }
}
```

- [ ] **Step 5: Verify the subproject compiles**

Run: `./gradlew :infra:compileJava`
Expected: build fails because `BlogStack` doesn't exist yet — that's the next task.

- [ ] **Step 6: Commit the scaffold**

```bash
git add infra settings.gradle
git commit -m "build(infra): CDK Java subproject scaffold"
```

---

## Task D2: BlogStack — S3 bucket (private) + CloudFront with OAC + ACM cert + Route 53

**Files:**
- Create: `infra/src/main/java/dev/linhvu/my_blog/infra/BlogStack.java`

- [ ] **Step 1: Implement BlogStack**

`infra/src/main/java/dev/linhvu/my_blog/infra/BlogStack.java`:

```java
package dev.linhvu.my_blog.infra;

import software.amazon.awscdk.*;
import software.amazon.awscdk.services.certificatemanager.Certificate;
import software.amazon.awscdk.services.certificatemanager.CertificateValidation;
import software.amazon.awscdk.services.cloudfront.*;
import software.amazon.awscdk.services.cloudfront.origins.S3BucketOrigin;
import software.amazon.awscdk.services.route53.*;
import software.amazon.awscdk.services.route53.targets.CloudFrontTarget;
import software.amazon.awscdk.services.s3.BlockPublicAccess;
import software.amazon.awscdk.services.s3.Bucket;
import software.amazon.awscdk.services.s3.BucketEncryption;
import software.constructs.Construct;

import java.util.List;

public class BlogStack extends Stack {

    public static final String DOMAIN = "blogs.linhvu.dev";
    public static final String PARENT_ZONE = "linhvu.dev";

    public BlogStack(final Construct scope, final String id, final StackProps props) {
        super(scope, id, props);

        Bucket bucket = Bucket.Builder.create(this, "SiteBucket")
                .bucketName("blogs-linhvu-dev")
                .blockPublicAccess(BlockPublicAccess.BLOCK_ALL)
                .encryption(BucketEncryption.S3_MANAGED)
                .versioned(true)
                .removalPolicy(RemovalPolicy.RETAIN)
                .build();

        IHostedZone zone = HostedZone.fromLookup(this, "ParentZone",
                HostedZoneProviderProps.builder().domainName(PARENT_ZONE).build());

        Certificate cert = Certificate.Builder.create(this, "SiteCert")
                .domainName(DOMAIN)
                .validation(CertificateValidation.fromDns(zone))
                .build();

        Function rewriteFn = Function.Builder.create(this, "TrailingSlashFn")
                .runtime(FunctionRuntime.JS_2_0)
                .code(FunctionCode.fromInline("""
                        function handler(event) {
                          var req = event.request;
                          var uri = req.uri;
                          if (uri.endsWith('/')) { req.uri = uri + 'index.html'; }
                          else if (!uri.includes('.')) { req.uri = uri + '/index.html'; }
                          return req;
                        }
                        """))
                .build();

        Distribution distribution = Distribution.Builder.create(this, "SiteDistribution")
                .domainNames(List.of(DOMAIN))
                .certificate(cert)
                .defaultRootObject("index.html")
                .defaultBehavior(BehaviorOptions.builder()
                        .origin(S3BucketOrigin.withOriginAccessControl(bucket))
                        .viewerProtocolPolicy(ViewerProtocolPolicy.REDIRECT_TO_HTTPS)
                        .functionAssociations(List.of(
                                FunctionAssociation.builder()
                                        .function(rewriteFn)
                                        .eventType(FunctionEventType.VIEWER_REQUEST)
                                        .build()))
                        .build())
                .errorResponses(List.of(
                        ErrorResponse.builder()
                                .httpStatus(404)
                                .responseHttpStatus(404)
                                .responsePagePath("/404.html")
                                .build()))
                .build();

        ARecord.Builder.create(this, "SiteAliasRecord")
                .zone(zone)
                .recordName("blogs")
                .target(RecordTarget.fromAlias(new CloudFrontTarget(distribution)))
                .build();

        AaaaRecord.Builder.create(this, "SiteAliasRecordAAAA")
                .zone(zone)
                .recordName("blogs")
                .target(RecordTarget.fromAlias(new CloudFrontTarget(distribution)))
                .build();

        CfnOutput.Builder.create(this, "BucketName").value(bucket.getBucketName()).build();
        CfnOutput.Builder.create(this, "DistributionId").value(distribution.getDistributionId()).build();
        CfnOutput.Builder.create(this, "SiteUrl").value("https://" + DOMAIN).build();
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `./gradlew :infra:compileJava`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 3: Commit**

```bash
git add infra/src
git commit -m "feat(infra): CDK BlogStack — S3 (private) + CF (OAC) + ACM + Route 53"
```

---

## Task D3: Bootstrap CDK and synth

**Files:** none (operational task)

- [ ] **Step 1: Verify AWS CLI is configured**

Run: `aws sts get-caller-identity`
Expected: prints account ID and user ARN. If it fails, configure credentials before proceeding.

- [ ] **Step 2: Verify CDK CLI is installed**

Run: `cdk --version`
Expected: prints `2.x.x`. If not installed: `npm install -g aws-cdk`.

- [ ] **Step 3: Bootstrap the AWS account/region**

Run from `infra/`:
```bash
cd infra
cdk bootstrap aws://$(aws sts get-caller-identity --query Account --output text)/us-east-1
```
Expected: `Environment ... bootstrapped` (creates the `CDKToolkit` stack the first time only).

- [ ] **Step 4: Synthesize the stack (no deploy yet)**

Run: `cdk synth`
Expected: prints CloudFormation template. No errors.

- [ ] **Step 5: Commit a note about the bootstrap (no code changes)**

No commit — bootstrap is a one-time AWS action, not a code change.

---

## Task D4: Deploy the stack

**Files:** none (operational task)

- [ ] **Step 1: Deploy from infra/**

Run: `cd infra && cdk deploy --require-approval never`

This will:
- Create S3 bucket `blogs-linhvu-dev` (private)
- Issue ACM cert for `blogs.linhvu.dev` (DNS validation via Route 53 — automatic since the parent zone is in this account)
- Create CloudFront distribution with OAC bound to the bucket
- Create A and AAAA records in the linhvu.dev hosted zone

Expected: completes in 5–15 minutes (CloudFront propagation is the slow part). Outputs three values: `BucketName`, `DistributionId`, `SiteUrl`.

- [ ] **Step 2: Save the outputs to a local config file**

Append to `.gitignore`:

```
### CDK outputs (contain account-specific IDs) ###
infra/.deploy-outputs
```

Then write outputs to that file:

```bash
cdk deploy --outputs-file .deploy-outputs.json --require-approval never
```

(They're also visible in the AWS console under CloudFormation → BlogStack → Outputs.)

- [ ] **Step 3: Verify the cert is issued**

In AWS console: CloudFront → distribution → check it's `Deployed` status.
Or: `aws cloudfront get-distribution --id <DistributionId> --query 'Distribution.Status'`
Expected: `"Deployed"` (after ~10 min).

- [ ] **Step 4: Verify DNS resolves**

Run: `dig +short blogs.linhvu.dev`
Expected: prints CloudFront IPs.

- [ ] **Step 5: Verify HTTPS responds (with 403 because bucket is empty)**

Run: `curl -I https://blogs.linhvu.dev/`
Expected: `HTTP/2 403` (bucket has nothing yet — that's fine, infra is up).

---

## Task D5: Bucket policy verification

**Files:** none (verification task)

- [ ] **Step 1: Verify bucket is private**

Run: `aws s3 ls s3://blogs-linhvu-dev/` (should succeed for you, the account owner)
Run: `curl -I https://blogs-linhvu-dev.s3.amazonaws.com/index.html` (should be 403 Forbidden — public access blocked)
Expected: AWS-side ls works; public URL blocked.

- [ ] **Step 2: Verify CloudFront has access via OAC**

The bucket policy was auto-generated by CDK. Check it:
```bash
aws s3api get-bucket-policy --bucket blogs-linhvu-dev --query Policy --output text | jq
```
Expected: a policy granting `s3:GetObject` to `cloudfront.amazonaws.com` for the specific distribution ARN.

---

### 🎯 Phase D milestone

CloudFront distribution is live at `blogs.linhvu.dev` (returns 403 because S3 is empty). DNS resolves. Cert is valid. Phase E will deploy content into the bucket.

---

# Phase E — CI/CD & first deploy

## Task E1: Create AWS IAM role for GitHub OIDC

**Files:** none initially, then optionally add to CDK in a follow-up

This task creates an IAM role that GitHub Actions assumes via OIDC. We do this in the AWS console for v1 (simple one-time action). It can be CDK-ified later.

- [ ] **Step 1: Create the OIDC identity provider in AWS IAM**

If not already present, create the GitHub OIDC IdP:
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

Or via CLI:
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

(If the provider already exists, this returns `EntityAlreadyExists` — fine.)

- [ ] **Step 2: Create the IAM role with trust policy + permissions**

Save as `infra/oidc-trust.json` (delete after creating role — don't commit):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": { "token.actions.githubusercontent.com:sub": "repo:GITHUB_ORG/my-blog:ref:refs/heads/main" }
    }
  }]
}
```

Replace `ACCOUNT_ID` with `aws sts get-caller-identity --query Account --output text` and `GITHUB_ORG` with the GitHub org/user that will own this repo.

Save as `infra/oidc-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::blogs-linhvu-dev"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::blogs-linhvu-dev/*"
    },
    {
      "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation"],
      "Resource": "*"
    }
  ]
}
```

Create the role:

```bash
aws iam create-role --role-name my-blog-deployer \
  --assume-role-policy-document file://infra/oidc-trust.json
aws iam put-role-policy --role-name my-blog-deployer \
  --policy-name DeployBlog \
  --policy-document file://infra/oidc-policy.json
aws iam get-role --role-name my-blog-deployer --query 'Role.Arn' --output text
```

Save the printed ARN — you'll set it as a GitHub repo secret next.

Delete the trust/policy JSON files so they're not committed:
```bash
rm infra/oidc-trust.json infra/oidc-policy.json
```

- [ ] **Step 3: Set GitHub repo secrets**

In the GitHub repo settings → Secrets and variables → Actions:
- `DEPLOY_ROLE_ARN` = the role ARN from previous step
- `CF_DIST_ID` = the CloudFront DistributionId from Phase D outputs

---

## Task E2: GitHub Actions deploy workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Write the workflow**

`.github/workflows/deploy.yml`:

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 25

      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}

      - name: Test
        run: ./gradlew test

      - name: Build site
        run: ./gradlew buildSite

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Sync to S3
        run: |
          aws s3 sync dist/ s3://blogs-linhvu-dev --delete \
            --cache-control "public, max-age=60" \
            --exclude "*.css" --exclude "*.js" --exclude "*.png" \
            --exclude "*.jpg" --exclude "*.svg" --exclude "*.woff*"
          aws s3 sync dist/ s3://blogs-linhvu-dev \
            --cache-control "public, max-age=31536000, immutable" \
            --exclude "*" --include "*.css" --include "*.js" --include "*.png" \
            --include "*.jpg" --include "*.svg" --include "*.woff*"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DIST_ID }} \
            --paths "/*"
```

- [ ] **Step 2: Commit**

```bash
git add .github
git commit -m "ci: GitHub Actions deploy workflow (OIDC, sync, invalidate)"
```

---

## Task E3: First deploy & verify v1 Definition of Done

**Files:** none (operational)

- [ ] **Step 1: Push to GitHub**

(If GitHub repo doesn't exist yet, create one called `my-blog` under your account, then:)

```bash
git remote add origin git@github.com:<YOUR_GITHUB_USER>/my-blog.git
git push -u origin main
```

- [ ] **Step 2: Watch the workflow run**

Open https://github.com/<YOUR_GITHUB_USER>/my-blog/actions
Expected: `Deploy to AWS` workflow runs and turns green within ~3 minutes.

- [ ] **Step 3: Verify the site is live**

Run: `curl -I https://blogs.linhvu.dev/`
Expected: `HTTP/2 200` with `cache-control: public, max-age=60`.

Open `https://blogs.linhvu.dev/` in a browser.
Expected: homepage renders with your design + the hello-world post listed.

Open `https://blogs.linhvu.dev/posts/hello-world/`
Expected: post renders with Java code highlighted by Prism.

- [ ] **Step 4: Verify auxiliary files**

```bash
curl -I https://blogs.linhvu.dev/index.xml
curl -I https://blogs.linhvu.dev/sitemap.xml
curl -I https://blogs.linhvu.dev/robots.txt
```
Each should return `HTTP/2 200`.

- [ ] **Step 5: Verify push-to-deploy works**

Edit `content/posts/2026-05-07-hello-world.md` — add a line to the body. Commit and push:

```bash
git add content/posts/2026-05-07-hello-world.md
git commit -m "post: add a line to hello-world"
git push
```

Wait ~3 minutes. Reload `https://blogs.linhvu.dev/posts/hello-world/`.
Expected: new line is visible.

---

### 🎯 v1 Definition of Done met

`https://blogs.linhvu.dev` shows the first article. `git push origin main` deploys.

---

## Self-review

**1. Spec coverage:** Walking through the spec sections vs tasks:

| Spec section | Tasks |
|---|---|
| §3.2 Module structure (5 modules + Modulith) | A2 |
| §3.3 Package layout | A2, A3, A4, A5, A6, A7, A8 (each task creates the correct paths) |
| §4 Content model (records + sealed) | A3 |
| §5 Build pipeline phases | A4 (LOAD), A5 (RENDER), A6 (EMIT), Phase E (PUBLISH) |
| §6 Tech stack | A1 (deps), A5 (JTE), A4 (commonmark) |
| §7 Dev experience (Gradle tasks) | A8 (`buildSite`), B1 (`bootRun`) |
| §8 AWS deployment | D1–D5 |
| §8 CI/CD (`deploy.yml`) | E2 |
| §9 Visual design (DESIGN.md) | C1, C2 |
| §10 Testing strategy | A2 (boundary), A3-A7 (unit), A8 (E2E smoke) |
| §11 Phase 1 DoD | E3 |

All Phase 1 spec items have a task. ✓

**2. Placeholder scan:** No "TBD" / "TODO" / "implement appropriate X" / "similar to Task N" — every step contains its full code.

**3. Type consistency:**
- `ContentApi.loadSite(Path, SiteConfig)` consistent across A4, A8, B1, BuildSiteSmokeTest.
- `RenderApi.renderAll(Site) → Map<String, RenderedPage>` consistent across A5, A6, A8.
- `OutputApi.emitAll(Site, Map<String, RenderedPage>, Path)` consistent across A6, A8.
- `AssetsApi.copyStatic(Path staticRoot, Path distRoot)` consistent across A7, A8, B1, DevServer.
- `SiteBuilder.build(SiteConfig, Path contentRoot, Path staticRoot, Path distRoot) → BuildResult` — defined A8, used A8 (smoke test), B1 (DevServer).
- Sealed `Page permits Post, StaticPage, IndexPage` — same across A3, A5, A6.
- `LayoutResolver.layoutFor(Page) → "post.jte" | "page.jte" | "list.jte"` — A5; consumed by `JteRenderer` A5.

All consistent. ✓

**4. Scope check:** Phase 1 is one cohesive deliverable (v1 deployed). Each phase ends at a testable milestone. The plan is large but the size is essential — Phase 1 covers SSG + dev server + design + infra + CI/CD, all of which are required for the DoD.
