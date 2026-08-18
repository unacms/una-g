# UNA-G rewrite plan

Evidence from unacms/una-g master (SHA eeae5b0, same tree as unacms/UNA) as of 18 Aug 2026 (AEST).

**North star:** UNA stays the community OS (Studio, modules, perms, payments, storage, API) and the API/admin brain for Neo (Next.js + React Native). Keep profiles, ACL, Studio, modules, payments, and the API. PHP templates are not the future frontend. Incremental PRs only. No big-bang rewrite. No new framework in the first five PRs.

---

## 1. Goals

- Keep UNA as the installable community OS and the backend for Neo.
- Ship security + testability first, then shrink the attack surface and the Dolphin-era naming tax.
- Make api.php + module services a stable contract Neo can call. Studio remains the operator UI.
- Replace world-writable install guidance and turn existing PHPUnit on.

## 2. Non-goals

- Throwing away profiles, ACL, Studio, modules/, payments, or the service API.
- Replacing PHP templates with a new frontend framework inside this repo as the next UI.
- A one-shot rename of every BX_DOL / BxDol* / boonex path (breaks module install).
- Rewriting search, cron, or GDPR as a greenfield stack before the first five PRs.

---

## 3. Current-state findings

Only facts verified by opening files/API. Sampled where noted.

### Repo identity

- api.github.com/repos/unacms/una-g: public fork of unacms/UNA. default_branch = master. Description: "UNA Community Management System". Created 2026-08-18. License MIT.
- inc/version.inc.php: BX_DOL_VERSION = 15.0.0-RC1. Guard: defined(BX_DOL) or defined(BX_DOL_INSTALL).

### Security

- SECURITY.md is 329 bytes. Supported: 15.x, 14.x. Report path is email only: team@unacms.com. No advisory process, no private GHSA mention.
- **CVE-2025-32101 / CVE-2025-66571 is not present in this fork.** Advisory (KIS-2025-01) showed unserialize($mixedProfileId) on POST profile_id in BxBaseMenuSetAclLevel::getCode(). Current template/scripts/BxBaseMenuSetAclLevel.php uses json_decode($mixedProfileId, true) after urldecode. File fetch was truncated at the constructor; the getCode() AJAX branch was fully readable and contains no unserialize.
- **CVE path confirmed patched** (full file): template/scripts/BxBaseMenuSetAclLevel.php getCode() AJAX branch does urldecode then json_decode($mixedProfileId, true). No unserialize.
- **create_function: 0 hits** in unacms/UNA (same tree).
- **First-party unserialize is still the live risk.** `inc/classes/` alone has 20 hits (parent-repo search; una-g is the same SHA). Highest leverage:
  - `BxDolService::callSerialized()` — `@unserialize($s)` then `call(module, method, params)`. Cron (`periodic/cron.php`) and alerts (`BxDolAlerts.php`) feed it stored `service_call` blobs.
  - `BxDolSession.php` — unserialize session `data` from DB.
  - `BxDolKey.php` — unserialize keyed payload.
  - Queues: `BxDolQueueEmail.php` params, `BxDolQueuePush.php` message.
  - Config blobs (usually admin-written, still no `allowed_classes`): FormQuery, Grid(+Query), Chart, Storage(+Query), Transcoder(+Query), ContentFilter, VoteReactions, Metatags, Recommendation, SearchExtendedQuery, ConnectionRelation.
  - Studio: `BxDolStudioWidgets.php` unserializes widget notices then service-calls.
  Vendor/upgrade copies (AWS SDK, Neuron, Elastic serializers, Facebook SDK) are noise for PR1.

### Install / 777

- inc/params.inc.php hard-codes BX_DOL_DIR_RIGHTS = 0777 and BX_DOL_FILE_RIGHTS = 0666.
- Official install wiki (unacms.com/wiki/Installation and github.com/unacms/UNA/wiki/Installation) still tells operators chmod -R 777 on inc, cache, cache_public, tmp, logs, storage, plus "writable or 777" in the GitHub wiki. Parent wiki, not a file in this fork.

### CI and tests

- .github/workflows/ci.yml: actions/checkout@v3, actions/cache@v3, composer validate, composer install, Phing package. **PHPUnit step is commented out** (# - name: Run test suite / composer run-script test). Root composer.json has no test script.
- tests/ exists: README.md, bootstrap.php, phpunit.xml, phpcpd.sh, tests/composer.json (phpunit/phpunit: 11.*). Units sampled: tests/units/inc/UtilTest.php, tests/units/inc/classes/BxDolDbTest.php, tests/units/modules/boonex/antispam/ (dir only; contents not listed).

### Architecture (sampled)

- **Forest of root PHP entry points.** Root listing (API /contents/) has 43 *.php files: agents.php, api.php, cart.php, chart.php, cmts.php, conn.php, em.php, embed.php, favorite.php, feature.php, form.php, get_rss_feed.php, grid.php, gzip_loader.php, healthcheck.php, image_transcoder.php, index.php, invoices.php, label.php, live_updates.php, logout.php, manifest.json.php, member.php, menu.php, oembed.php, orders.php, page.php, privacy.php, r.php, recommendation.php, report.php, score.php, searchExtended.php, searchKeyword.php, searchKeywordContent.php, splash.php, storage.php, storage_uploader.php, subscriptions.php, sw.js.php, view.php, vote.php, webhook.php.
- .htaccess already routes some traffic: ^m/ to modules/index.php, ^page/ to page.php, ^s/ to storage.php, fallback r.php.
- index.php goes to splash or page.php with i=home. page.php (fetch truncated at top) ends in BxDolPage::getObjectInstanceByURI / displayPage().
- api.php (fetch truncated at top): GET r = module/method/class, BxDolRequest::serviceExists, is_safe_service / is_public_service unless sys_api_access_unsafe_services, params via json_decode, JSON {status, module, method, params, data, hash}.
- Dolphin naming is live: BX_DOL / BxDol* throughout inc/params.inc.php, inc/classes/BxDol.php, install classes (install/classes/BxDolInstall*.php). inc/classes/ listing is a long BxDol* catalog (Account, Acl, AI, Cache*, Cron*, Form, Storage, …). First page of the API listing was truncated; not counted.
- Modules: modules/base/ (connect, files, general, groups, notifications, payment, profile, template, text), modules/system/, **modules/boonex/ = 92 app dirs** (accounts, acl, ads, api, elasticsearch, oauth2, payment, persons, posts, …). Vendor name is still boonex.
- Studio stays a first-class tree: studio/ with dashboard, store, builders, options, polyglot, storages, studio/api.php, etc. inc/params.inc.php defines BX_DOL_STUDIO_FOLDER = studio.
- Background jobs: periodic/cron.php (DB jobs + serialized BxDolService::callSerialized + AI schedulers). Also inc/classes/BxDolBackgroundJobs.php, BxDolCron*.php.
- Search stack already in Composer: elasticsearch/elasticsearch ^9.3, opensearch-project/opensearch-php ^2.5, typesense/typesense-php. Module: modules/boonex/elasticsearch. Root search entry points: searchExtended.php, searchKeyword.php, searchKeywordContent.php.
- GDPR: no dedicated module in the boonex listing. README claims "compliance preparation service". Account delete exists via modules/boonex/accounts (wiki, not re-read here). Personal-data map lives on the parent wiki, not in-repo.

### Dependencies (verified files)
- composer.json: PHP >=8.1; achingbrain/php5-akismet 0.5; tpyo/amazon-s3-php-class (dev-master) plus unacms hmac-v2 fork; intervention/image ^2.6; erusev/parsedown @dev; Chargebee, Stripe ^19, PayPal, PHPMailer ^7, Neuron AI ^3, AWS SDK ^3.371.
- bower.json still present (una-js-libs: combodate, jqueryui-touch-punch, history).
- JS manifest files are in the repo root.
- JS root manifests exist (name una-js-libs). Versions: jquery 3.7.1, migrate 3.6.0, UI 1.14.2, quill 1.3.7. Bower is still a devDependency.

---

## 4. First 5 PRs

Order is intentional: security, then CI that would have caught it, then stop recommending world-writable dirs. No new framework.

### PR1 — Keep the ACL unserialize fix locked; audit remaining unserialize

- **Problem:** The CVE path is already json_decode, but nothing in CI proves it stays that way, and BxDolService::callSerialized() still unserializes blobs. SECURITY.md is email-only.
- **Files/areas:** template/scripts/BxBaseMenuSetAclLevel.php; inc/classes/BxDolService.php; SECURITY.md; new tests/units/ for ACL getCode() input (numeric / JSON array / reject serialized PHP objects). Full unserialize/eval inventory (code search needs auth).
- **Done-when:** Regression test fails if unserialize() returns on POST profile_id. callSerialized documented as trusted-store-only (or allowed_classes / JSON migration started). SECURITY.md has a GHSA/private-report path, not only team@unacms.com.

### PR2 — Turn PHPUnit on in CI

- **Problem:** Tests exist (tests/, PHPUnit 11) but .github/workflows/ci.yml comments them out. Actions are @v3.
- **Files/areas:** .github/workflows/ci.yml; root composer.json scripts; tests/phpunit.xml; tests/bootstrap.php. Bump checkout/cache to current major. Add a PHP 8.1 (or the version you actually run) matrix step that runs tests/vendor/bin/phpunit.
- **Done-when:** A PR that breaks UtilTest or the new ACL test fails CI. Packaging (Phing) still runs. No leftover comment that tests are optional.

### PR3 — Delete 777 guidance and default world-writable rights

- **Problem:** BX_DOL_DIR_RIGHTS / BX_DOL_FILE_RIGHTS are 0777/0666. Install wiki tells operators to make inc, caches, tmp, logs, storage world-writable.
- **Files/areas:** inc/params.inc.php; install permission checks under install/ (re-read BxDolInstall* before editing); README / any in-repo install notes; parent wiki + unacms.com wiki (docs PR, can land after code).
- **Done-when:** Defaults are owner/group writable (0770/0660 or 0775/0664), never world-writable. Installer still works as www-data. Wiki no longer recommends 777. cache_public remains web-readable files, not 777 trees.

### PR4 — Dependency hygiene (no framework change)
- Problem: Dead/old stack in lockfiles: Bower, php5-akismet 0.5, old S3 class, Intervention Image 2.x, Parsedown at-dev, Quill 1.3.7, jQuery plus migrate plus UI.
- Files/areas: composer manifests, bower.json, JS root manifests, patches/.
- Done-when: Each abandoned package has an owner (replace, pin, or delete). Bower removed or sunset-dated. Install still builds. No new PHP framework.

### PR5 — API contract tests for Neo (keep api.php)
- Problem: Neo will live on api.php plus module services. Safety is a runtime is_safe_service / is_public_service check. No CI coverage of the JSON contract. 43 root PHP files remain public surface.
- Files/areas: api.php; modules/boonex/api; BxDolRequest / BxDolService (sampled); new tests for r=module/method 404/403/200 shapes; inventory of root PHP files (which stay vs fold into page.php/r.php later).
- Done-when: CI asserts missing r is 404 JSON; unsafe service is 403 unless the flag is on; a known public service returns status 200 with module, method, data. Root-entry list checked in. Still no front-controller rewrite in this PR.

---

## 5. What comes after

Brief, after the first five land.
- Front controller: Collapse the 43 root PHP files behind .htaccess / r.php / page.php incrementally. Keep api.php, storage.php, periodic/cron.php, studio/ as named surfaces. Do not start this before PR5 inventory.
- Dolphin / BX_DOL / BxDol* / modules/boonex rename: compatibility layer only. Module ZIP install and Studio store paths must keep working. Alias old class names; do not break the boonex vendor dir until a dual-read installer exists.
- API for Neo: grow api.php + modules/boonex/api + OAuth2 (modules/boonex/oauth2) as the Neo contract. PHP templates stay for legacy web. Do not make PHP HTML the future UI.
- Studio stays: operator brain. Later, Studio-over-API so Neo admin can reuse the same services. Do not replace Studio in year one.
- Search stack: already Composer-wired (ES / OpenSearch / Typesense) plus modules/boonex/elasticsearch. Pick one default for Neo, document the others, stop adding engines.
- Background jobs: periodic/cron.php + BxDolBackgroundJobs* + email/push queues. Next: a real worker (same PHP). Keep the job table.
- GDPR: build on modules/boonex/accounts delete-with-content + the parent wiki personal-data table. Add export/erasure APIs Neo can call. README compliance-service line is not a product feature.

---

## Fetch failures

- GitHub code search on una-g itself is unindexed (fork). Inventory above is from unacms/UNA at the same SHA eeae5b0.
- Several raw-file fetches were truncated at the start (api.php, page.php, periodic/cron.php, vote.php, privacy.php, BxDolService.php, BxBaseMenuSetAclLevel.php constructor). Cited lines are from the readable remainder. Re-open those files locally before coding.
- tests/phpunit.xml fetch returned stripped/empty markup. Existence confirmed via contents/tests.
- inc/classes/ listing truncated; class count not verified.
- tests/units/modules/boonex/antispam listed as a dir only; files inside not opened.
