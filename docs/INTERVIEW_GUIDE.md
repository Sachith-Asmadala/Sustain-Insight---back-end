# Sustain Insight: code learning and interview guide

Reviewed backend snapshot: `e2e99800365b620681ab8d464e6fc27efc63b0f6`; companion frontend snapshot: `321f940f16a73eead99691785f2a6aa6e745ad37`. This is a source walkthrough, not a deployment or test-success certificate.

## Problem and scope

Sustain Insight collects sustainability-related news and exposes article browsing, bookmarks, and user interactions. The backend contains Java 21, Spring Boot 3.5.5, Spring Security, JPA, PostgreSQL/Flyway, news-source fetchers, and AI synthesis services. The [React/TypeScript frontend](https://github.com/Sachith-Asmadala/Sustain-Insight---front-end) is a separate repository. Understanding one complete interaction is more useful than claiming to have built every subsystem.

## Startup and communication

[BackendApplication](https://github.com/Sachith-Asmadala/Sustain-Insight---back-end/blob/e2e99800365b620681ab8d464e6fc27efc63b0f6/src/main/java/com/news_aggregator/backend/BackendApplication.java) calls `SpringApplication.run()` and enables scheduling. Spring creates controllers, services, repositories, security filters, and configured database connections. Flyway configuration/migrations provide the database setup path; successful startup depends on the required environment and services.

The frontend's API helpers send HTTP requests. `SecurityConfig` permits designated public/authentication endpoints and requires authentication for the others, including insights and bookmarks. `JwtAuthenticationFilter` participates in establishing the user principal. Controllers receive requests, services apply rules, and repositories access PostgreSQL. Background fetch/filter/synthesis services form a separate news-ingestion path; the insight walkthrough below does not invoke the AI provider.

## Important code to read

All backend paths below are under `src/main/java/com/news_aggregator/backend/`.

| File/class | Why it matters |
| --- | --- |
| [controller/InsightController.java](https://github.com/Sachith-Asmadala/Sustain-Insight---back-end/blob/e2e99800365b620681ab8d464e6fc27efc63b0f6/src/main/java/com/news_aggregator/backend/controller/InsightController.java) | Maps add/remove/count/current-user HTTP operations |
| [service/InsightService.java](https://github.com/Sachith-Asmadala/Sustain-Insight---back-end/blob/e2e99800365b620681ab8d464e6fc27efc63b0f6/src/main/java/com/news_aggregator/backend/service/InsightService.java) | Handles duplicate checks and cached count updates |
| `service/ArticleService.java` | Looks up articles and maps entities into response DTOs |
| [repository/ArticleRepository.java](https://github.com/Sachith-Asmadala/Sustain-Insight---back-end/blob/e2e99800365b620681ab8d464e6fc27efc63b0f6/src/main/java/com/news_aggregator/backend/repository/ArticleRepository.java) | Implements SQL increment/decrement of insight counts |
| `repository/InsightRepository.java`, `model/Insight.java`, `model/InsightId.java` | Persist the user/article relationship with a composite identity |
| `controller/GlobalExceptionHandler.java` | Maps validation, conflicts, authentication, and fallback errors |
| `service/BookmarkService.java` | Another readable controller/service/repository feature with pagination |

## Walkthrough: add an insight to an article

1. In the frontend, [ArticleCard.toggleInsight](https://github.com/Sachith-Asmadala/Sustain-Insight---front-end/blob/321f940f16a73eead99691785f2a6aa6e745ad37/src/components/articles/ArticleCard.tsx) opens a loading modal and calls `addInsight(articleData.id)` when the current state is false.
2. [src/api/insights.ts](https://github.com/Sachith-Asmadala/Sustain-Insight---front-end/blob/321f940f16a73eead99691785f2a6aa6e745ad37/src/api/insights.ts) calls `apiFetch('/api/insights/{articleId}', {method: 'POST'})`. The shared helper handles credentials/access-token behavior.
3. The backend security path authenticates the request. `InsightController.addInsight()` obtains the authenticated `User` and calls `ArticleService.getArticleEntity(articleId)`.
4. `InsightService.addInsight()` returns for null inputs or a duplicate user/article pair. Otherwise it builds an `Insight` with a timestamp and saves it.
5. Within the service transaction, `adjustInsightCount()` calls a SQL increment using `COALESCE(insight_count, 0) + 1` and updates the in-memory article value.
6. The controller returns `ArticleService.mapToDto(article, user)`. The current frontend waits for success, then adjusts its own boolean/count locally; it does not use the returned DTO to replace those values.
7. Removal calls the DELETE route, checks existence, removes the relationship, and decrements with a zero floor. For sequential calls, adding the same insight again is intended to leave the count unchanged. Concurrent calls need database-backed verification.

Example: with count 7 and no relationship for this user, a successful add is intended to produce one relationship and count 8. A later remove is intended to restore 7. These are expected code paths, not results from a new integration run.

## Data structures and engineering decisions

`InsightId` models a composite key: one identity per user/article pair. JPA entities represent persistent relationships; DTOs control the API shape. Article lists use `List<ArticleDto>` and Spring `Page`/`Pageable` so the API can return a bounded page rather than every row. Bookmarks convert one-based frontend page numbers into Spring's zero-based indexes.

The article's cached `insightCount` avoids counting all interaction rows for every read. That improves the read path at the cost of keeping the cached count consistent. `@Transactional` groups database operations, but it does not by itself prove all concurrent duplicate-add/remove behavior is correct. SQL increments avoid a simple read-then-write increment race; entity state and uniqueness conflicts still need tests.

## Errors, limitations, and improvements

The frontend displays loading/success/error feedback and updates local counts after successful requests. Backend advice returns 400 for validation, 409 for data conflicts, 401 for bad credentials, and a generic 500 fallback. A missing article currently throws a generic `RuntimeException`, so this path reaches a server error rather than a specific 404 handler. Some JWT errors can arise before controller advice; do not claim every authentication failure has a consistent response.

The current test tree contains an application-context test, not focused insight/authentication tests. Good next cases are duplicate add, remove absent relationship, null cached count, count never below zero, concurrent add/remove, unauthorized user, and missing article. Also verify password-reset session invalidation and frontend refresh behavior separately. The client currently refreshes on some 403 responses as well as 401, which can confuse permission errors with expired authentication.

## Contribution evidence

GitHub attributes [commit 4c1d4ad](https://github.com/Sachith-Asmadala/Sustain-Insight---back-end/commit/4c1d4ad337466ceb513980fc3edb28459bf4f2af) adding `InsightService.java` to `Sachith-Asmadala`. Treat this as specific account-attributed evidence. Confirm your implementation role and assistance before saying “I implemented the insight service.” Ownership of the entire backend, frontend, ingestion, and AI features is not established by that commit.

## Five interview questions and model answers

1. **Why is insight counting stored on the article?** It is a cached total for faster reads. Each add/remove must keep that total synchronized with relationship rows, so consistency is the main tradeoff.
2. **Does checking existence guarantee no duplicate under concurrency?** No. Two requests can both pass the check. The composite database identity, transaction behavior, and conflict handling must be tested together.
3. **Why use a transaction here?** The relationship change and SQL count update should succeed or roll back together. A transaction does not automatically resolve every isolation or stale-entity problem.
4. **Where does the user ID come from?** The controller receives an authenticated principal, rather than accepting a user ID from the insight request body. Security filters establish that principal.
5. **What would you improve first in this feature?** Add focused database integration tests for duplicate/concurrent operations, return a clear 404 for a missing article, and reconcile client state with the returned DTO. These are proposals, not implemented fixes.

## Practice

Draw the POST path across both repositories. Explain the difference between a relationship row, a cached total, and React state. Predict which HTTP response a missing article currently produces before proposing a change.
