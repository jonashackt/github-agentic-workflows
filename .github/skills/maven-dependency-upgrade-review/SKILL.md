---
name: maven-dependency-upgrade-review
description: Review and repair Renovate dependency-update pull requests for a Spring Boot / Maven (Java 21) project.
---

# Maven dependency upgrade review

Use this skill for pull requests that change `pom.xml`, `renovate.json` or GitHub Actions versions in `.github/workflows/`.

## Checks

- Identify what changed: Spring Boot parent, a managed dependency, a version property (e.g. `rest-assured.version`), or an action version.
- Patch and minor bumps of managed dependencies are normally safe. Major bumps need a look at the upstream release notes for breaking changes.
- A Spring Boot minor bump may require changes in `src/main/resources/application.properties` (renamed or removed properties) and in `WebSecurityConfiguration`.
- Look for version properties that were bumped in only one place.
- For action bumps, check that the `with:` inputs used in `build-publish.yml` still exist.

## Fixing a broken build

- Reproduce with `mvn package --batch-mode --no-transfer-progress` (the same command CI uses).
- Make the smallest change that fixes the failure (renamed API, moved package, changed property). Never delete or weaken tests to get a green build.
- If the cause is unclear after reading the build log and release notes, do not guess: report it.

## Scope

- You never merge. Renovate merges green patch and minor updates itself; major bumps are merged by the maintainer.
- Fix the build for major bumps too, but mention the breaking-change risk in a review comment.
