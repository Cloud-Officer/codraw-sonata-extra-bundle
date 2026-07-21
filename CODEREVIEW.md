# Code Review — codraw/sonata-extra-bundle

Reviewed: all PHP sources, `composer.json`, DI configuration, service definitions and Twig resources of `codraw-sonata-extra-bundle` (namespace `Draw\Bundle\SonataExtraBundle`). Tests were skimmed for coverage assessment.

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
Low-risk fixes were applied for the findings below. `composer validate --no-check-publish` passes and all modified files pass `php -l`.

**Validation pass (2026-07-20):** `composer install` resolves cleanly with the updated `composer.json` (no constraint adjustments needed). Full PHPUnit run against MySQL: 171 tests, 219 assertions, all passing with the fixes applied — no test-expectation updates were required. The 17 PHPUnit notices ("mock without expectations") are pre-existing test-style issues in untouched test files. PHPStan (`phpstan.dist.neon`) reports 32 errors, all verified pre-existing (identical error list with the fixes stashed): missing optional `symfony/workflow`/`phpdocumentor` classes, config-builder type inference in `Configuration.php`, and a few test-file nits; no stale baseline entries surfaced. `markdownlint-cli2` reports 0 errors. No additional code changes were made in the validation pass.

**composer.json (H3, L5):**

- Moved `sonata-project/admin-bundle` (`^4.8`) and `sonata-project/doctrine-orm-admin-bundle` (`^4.2`) from `require-dev` to `require` (H3 — shipped code hard-depends on both, including unconditional compiler passes).
- Added missing runtime dependencies used unconditionally by shipped code: `doctrine/orm` (`^3.6`), `doctrine/persistence` (`^2.2 || ^3.0`), `doctrine/common` (`^3.1`, `ClassUtils` in `Twig/EntityTwigExtension`), `doctrine/inflector` (`^2.0`, `Extension/BatchActionExtension`), `psr/log` (`^3`), `twig/twig` (`^3.3`) (H3).
- Added a `suggest` section for optional, feature-gated integrations: `phpdocumentor/reflection-docblock` (auto_help feature, H3), `symfony/workflow` (workflow feature), `symfony/notifier` (notifier feature/execution notifications, also kept in `require-dev`), `sonata-project/exporter` (optional export support in `ControllerTrait`).
- Filled in the empty package `description` (L5).
- Open item (H3, not changed): `symfony/browser-kit` remains in `require` even though it is only used by `Test/AdminTestHelperTrait.php` — removing/demoting it could break consumers relying on it transitively.

**Code fixes:**

- M2: `ActionableAdmin/AdminAction.php` — `getController()` return type changed to `?string`, removing the guaranteed `TypeError` for actions registered without a controller.
- M3 (partial): `ActionableAdmin/ObjectActionExecutioner.php` — removed the dead `$this->admin->getModelManager();` statement, and `execute()` now skips `null` objects (`skip('not-found')`) instead of yielding them into the execution callback where they caused a `TypeError` (and a second `TypeError` in the catch-block logging). The hardcoded `o`/`id` alias assumptions and N+1 fetching remain open.
- M5: `ActionableAdmin/ExecutionNotifier.php` — `%error%` is now passed through `escapeHtml()` like `%object%`.
- M7: `Doctrine/Filter/RelativeDateTimeFilter.php` — `strtotime()` result is checked for `false`; unparseable input now skips the filter instead of filtering on 1970-01-01.
- M8: `DependencyInjection/Compiler/AutoConfigureSubClassesCompilerClass.php` — added `ksort()` so the `position` tag attribute is honored.
- M9: `Controller/BatchAdminController.php` — decoded batch `data` is validated with `is_array()`; scalar JSON now yields a `BadRequestHttpException` (4xx) instead of a `TypeError` (500).
- L1: `DependencyInjection/Configuration.php` — `} if` replaced with `} elseif` (behavior-identical today, removes the trap).
- L2: `Controller/KeepAliveController.php` — deprecated `Routing\Annotation\Route` import replaced with `Routing\Attribute\Route`.
- L3: `Workflow/Extension/WorkflowExtension.php` — exception message now references the real `admin_action_class` option key.
- L4: `EventListener/SessionTimeoutRequestListener.php` — script injected once before the first `<title>` via `substr_replace()` instead of `str_replace()` on every occurrence.
- L5 (partial): `ActionableAdmin/GenericFormHandler.php` — missing `execution` key now throws a clear `InvalidArgumentException` instead of a warning + late failure; `ActionableAdmin/ArgumentResolver/ObjectActionExecutionerValueResolver.php` — `['action']` access uses the `?? null` idiom, removing the warning when the attribute is absent. The `skip()` unconditional decrement remains open.

**Deliberately not fixed (need design/security decisions):** H1 (voter strategy), H2 (workflow CSRF/GET), M1 (compiler-pass guards — moot now that the Sonata packages are hard requirements), M4 (CSRF option-slot redesign), M6 (CDN asset/SRI), and the remaining parts of M3/L5 noted above.

## Overall assessment

This is a feature-rich, generally well-engineered Sonata Admin companion bundle: modern PHP 8 style, a clean event-driven "ActionableAdmin" execution pipeline, a thoughtful PreventDelete subsystem (attributes + config + Doctrine-metadata inference with cache), and disciplined per-feature DI wiring where disabled features have their service definitions removed. Queries are consistently parameterized — no SQL injection was found. However, there are notable issues concentrated in the security-sensitive features: the PreventDelete voter is silently defeated by the bundle's own DefaultCanVoter under Symfony's default (affirmative) access-decision strategy; workflow transitions are applied by plain GET links with no CSRF protection; and the package's `composer.json` does not declare its real runtime dependencies (all Sonata/Doctrine packages are `require-dev` only, and one runtime dependency is not declared at all). Test coverage of runtime behavior is thin — the DI layer and one listener are well tested, but the execution pipeline, controllers, extensions and voters are untested in this package.

---

## Findings

### High

#### H1. PreventDeleteVoter is neutralized by DefaultCanVoter under Symfony's default access-decision strategy

- `PreventDelete/Security/Voter/PreventDeleteVoter.php:28-32` returns `ACCESS_DENIED` when a protected relation exists.
- `Security/Voter/DefaultCanVoter.php:10-22` returns `ACCESS_GRANTED` for **any** `SONATA_CAN_*` attribute.

Both voters are enabled together by the documented configuration (`can_security_handler.enabled: true` with default `grant_by_default: true` plus `prevent_delete_voter.enabled: true`, see `DependencyInjection/DrawSonataExtraExtension.php:88-130`). Symfony's default access-decision strategy is *affirmative*: one `GRANTED` wins over any number of `DENIED`. So for `SONATA_CAN_DELETE` on an object with blocking relations, `DefaultCanVoter` grants, `PreventDeleteVoter` denies — and access is **granted**. The delete-prevention feature silently does nothing unless the application happens to configure a `unanimous`/`priority` strategy, which is neither enforced nor documented anywhere in this bundle (checked `README.md` and `docs/`). Either the deny should be expressed so it wins (e.g. `DefaultCanVoter` abstaining when a prevent-delete relation matches, a single combined voter, or documented `unanimous` requirement enforced at container-build time).

#### H2. Workflow transitions are state-changing GET requests with no CSRF protection

- `Workflow/Action/WorkflowTransitionAction.php:21-57` applies a workflow transition and calls `$admin->update($object)` on any request method, keyed only by the `transition` request parameter.
- `Workflow/Extension/WorkflowExtension.php:165-171` generates the transition links as plain menu URIs (GET).

Unlike `DeleteAction` (`ActionableAdmin/Action/DeleteAction.php:24-28`), which wires `CsrfTokenValidatorListener` and only mutates on POST/DELETE, the workflow action sets no CSRF intention and accepts GET. A logged-in admin can be CSRF'd into applying any enabled transition (e.g. via an `<img src=".../workflow-apply-transition?transition=publish">`), and GET side effects are also vulnerable to link prefetchers. State transitions should require POST + CSRF token like delete does.

#### **[FIXED]** H3. composer.json does not declare the bundle's real runtime dependencies

- `composer.json:10-26`: the only production requirements are `codraw/dependency-injection`, `symfony/browser-kit`, `symfony/framework-bundle`, `symfony/expression-language`, `symfony/string`.

Virtually every class in the bundle type-hints against `sonata-project/admin-bundle` (and many against `sonata-project/doctrine-orm-admin-bundle`, e.g. `ObjectActionExecutioner` uses `Sonata\DoctrineORMAdminBundle\Datagrid\ProxyQuery`, the Doctrine filters extend `Sonata\DoctrineORMAdminBundle\Filter\Filter`), yet those packages are `require-dev` only — Composer will happily install this bundle into a project with an incompatible or missing Sonata version. Worse, `EventListener/AutoHelpListener.php:6` uses `phpDocumentor\Reflection\DocBlockFactory`, which appears in **no** dependency section at all: enabling `auto_help` in a project that does not coincidentally ship `phpdocumentor/reflection-docblock` produces a fatal "class not found" at runtime. At minimum the Sonata packages belong in `require` (or `conflict` bounds + `suggest`), and the phpDocumentor package must be declared. Additionally `symfony/browser-kit` is a production requirement but is only used by the test helper `Test/AdminTestHelperTrait.php` — it belongs in `require-dev`/`suggest`.

### Medium

#### M1. Compiler passes assume Sonata services exist and use fragile positional access

- `DependencyInjection/Compiler/DecoratesCompilerPass.php:16-34` unconditionally decorates `sonata.admin.builder.orm_form` and `sonata.admin.field_description_factory.orm`. If `sonata-doctrine-orm-admin-bundle` is not installed, container compilation fails with a `ServiceNotFoundException` from the decorator pass — there is no `$container->has()` guard.
- `DependencyInjection/Compiler/AutoConfigureSubClassesCompilerClass.php:12-13` calls `getDefinition('sonata.admin.pool')` (throws if admin-bundle absent) and reads/writes constructor argument index `3` of Sonata's pool, which will break silently or loudly if Sonata reorders its constructor.

#### **[FIXED]** M2. `AdminAction::getController(): string` on a nullable property causes TypeError

- `ActionableAdmin/AdminAction.php:18` declares `private ?string $controller = null` but `getController()` (line 131-134) has return type `string`.
- `ActionableAdmin/Extension/ActionableAdminExtension.php:36` calls `$action->getController()` for every action while configuring routes.

Any `AdminAction` registered without `setController()` being called produces `TypeError: Return value must be of type string, null returned` during route configuration. The guard `if ($action->getController())` at line 36 shows the call site expects `?string`; the return type should be `?string`.

#### **[PARTIALLY FIXED]** M3. ObjectActionExecutioner hardcodes root alias `o` and identifier field `id`, and can yield null objects

- `ActionableAdmin/ObjectActionExecutioner.php:71` (`$this->target->select('o.id as id')`) and `:111-115` (`select('COUNT(o.id)')`) assume the ProxyQuery root alias is `o` and the entity's identifier field is literally named `id`. Entities with a differently named identifier (`uuid`, `code`) or composite identifiers break with a DQL error.
- Line 73: `$this->admin->getModelManager();` is a dead statement (result unused).
- Line 75-77: `$this->admin->getObject($id['id'])` can return `null` (row deleted between the id-query and the fetch in a long batch); `null` is then yielded to the execution callback, producing a TypeError inside `execute()` (e.g. `DeleteAction`'s closure takes `$object` and `$this->admin->id($object)` in the catch block would also fail). A `null` check with `skip('not-found')` would be safer. Batch execution is also N+1 (one `getObject()` per row) — acceptable for admin batches but worth noting.

#### M4. Silent CSRF bypass footgun in CsrfTokenValidatorListener

- `ActionableAdmin/EventListener/CsrfTokenValidatorListener.php:33-35`: during `PrepareExecutionEvent`, if `options[TOKEN]` is not set, the listener *generates* a fresh token for the intention and stores it; `onPreExecutionEvent` (lines 49-55) then validates that same token — which is always valid.

The protection only works if the controller explicitly copies the token from the request into `options[TOKEN]` (as `DeleteAction.php:26-28` does for POST/DELETE). Any custom action that sets `INTENTION` but forgets to set `TOKEN` from the request gets a CSRF check that silently always passes. The generate-on-prepare behavior (intended to feed the confirmation template) and the validate-on-pre-execute behavior should not share the same option slot. Also, an invalid token throws a plain `\RuntimeException` (line 55) → HTTP 500 instead of a 4xx (compare `ControllerTrait::validateCsrfToken` which correctly throws `HttpException(400)`).

#### **[FIXED]** M5. Unescaped exception message in flash notification (XSS vector)

- `ActionableAdmin/ExecutionNotifier.php:103-104`: `%object%` is passed through `escapeHtml()` (implying the flash target renders raw HTML) but `%error%` interpolates `$throwable->getMessage()` unescaped into the same message.

If any execution error message echoes attacker-influenced data (validation errors, DB driver messages containing input, etc.), it lands unescaped in a `sonata_flash_error` message. Either both should be escaped or neither; as written the code is internally inconsistent and the error path is the dangerous one. Leaking raw internal exception messages to the UI is also an information-disclosure concern.

#### M6. Third-party CDN JavaScript injected into every admin page without SRI

- `DependencyInjection/DrawSonataExtraExtension.php:262-267`: with the default `install_assets: true`, `https://cdn.jsdelivr.net/npm/jquery.json-viewer@1.2.0/...` JS and CSS are prepended to `sonata_admin` assets for the whole admin backend, with no Subresource Integrity hash and no self-host option. A CDN compromise executes arbitrary JS in the admin session. Vendor the asset or add SRI.

#### **[FIXED]** M7. RelativeDateTimeFilter silently filters on 1970-01-01 for invalid input

- `Doctrine/Filter/RelativeDateTimeFilter.php:43`: `date('Y-m-d H:i:s', strtotime($value))` — `strtotime()` returns `false` for unparseable user input, and `date(..., false)` coerces to timestamp `0`, so an invalid filter value silently becomes `<= 1970-01-01 00:00:00`, showing confusingly wrong results instead of an empty/no-op filter. (Passing `false` where `?int` is expected is also deprecated on recent PHP.) Check `strtotime()` for `false` and skip the filter.

#### **[FIXED]** M8. Sub-class tag `position` attribute is not actually ordered

- `DependencyInjection/Compiler/AutoConfigureSubClassesCompilerClass.php:15-21`: tags are grouped by `position` into `$subClasses[position][label]`, then flattened with `array_merge(...$subClasses)` **without** `ksort()`. PHP preserves insertion order of the position keys, so tags declared as position 2 before position 1 keep declaration order — the `position` attribute is ignored. Add `ksort($subClasses)` before the merge.

#### **[FIXED]** M9. Batch `data` JSON accepted without array validation

- `Controller/BatchAdminController.php:106`: `$forwardedRequest->request->add(json_decode($encodedData, true, ...))` — a request with `data="5"` or `data="\"x\""` decodes to a scalar and `InputBag::add()` throws a TypeError → HTTP 500 from user-controllable input. Validate `is_array()` and throw `BadRequestHttpException` otherwise (the code already handles the malformed-JSON case properly).

### Low

#### **[FIXED]** L1. `} if` instead of `elseif` in configuration normalizer

- `DependencyInjection/Configuration.php:170`: `} if (!isset($configuration['field_name'])) {` — two independent `if`s on one line where an `elseif` was intended. It happens to be harmless today (the second branch re-assigns the same value for scalars) but it is a trap for future edits.

#### **[FIXED]** L2. Deprecated `Routing\Annotation\Route` import

- `Controller/KeepAliveController.php:7` imports `Symfony\Component\Routing\Annotation\Route` (deprecated in 6.4, removed in Symfony 7 in favor of `Symfony\Component\Routing\Attribute\Route`). Blocks a future Symfony 7 bump.

#### **[FIXED]** L3. Wrong option key referenced in exception message

- `Workflow/Extension/WorkflowExtension.php:198` uses `$this->options['action_class']` in the `LogicException` message; the option is named `admin_action_class` — the error path itself triggers an "undefined array key" warning.

#### **[FIXED]** L4. Session-timeout script injection replaces every `<title>` occurrence

- `EventListener/SessionTimeoutRequestListener.php:99-113`: `str_replace('<title>', ...)` replaces **all** occurrences; pages containing inline SVG `<title>` elements get the script duplicated. Use a position-limited replace (`preg_replace` with limit 1) on `<head>`/first `<title>`.

#### **[PARTIALLY FIXED]** L5. Minor robustness gaps

- `ActionableAdmin/GenericFormHandler.php:32` reads `$executions['execution']` without checking the key exists (ObjectActionExecutioner validates it only later).
- `ActionableAdmin/ArgumentResolver/ObjectActionExecutionerValueResolver.php:40`: `$request->attributes->get('_actionableAdmin')['action']` emits a warning when the attribute is absent (the `?? null` idiom used at `ActionableAdminListener.php:33` is correct).
- `ActionableAdmin/ObjectActionExecutioner.php:81-86`: `skip()` decrements `processedCount` unconditionally; calling it from a `PreExecutionEvent` listener drives the count negative.
- `composer.json:3`: empty `description` for a published package.

---

## Strengths

- **Modern, consistent codebase**: constructor property promotion, typed properties, attributes (`#[AsEventListener]`, `#[AutoconfigureTag]`, `#[Exclude]`), first-class enum-free config — pleasant to read and uniform across ~90 files.
- **ActionableAdmin pipeline design**: the Prepare → Pre → Execution(+Error) → Post event lifecycle in `ObjectActionExecutioner` with pluggable listeners (access check, CSRF, notification, redirect fallback) is a genuinely good abstraction that unifies single-object and batch actions, and `DeleteAction` shows the intended secure usage.
- **PreventDelete subsystem**: relations inferred from Doctrine association mappings (honoring `onDelete` SET NULL/CASCADE), class/property/trait attributes, plus config overrides, with a `ConfigCache`-backed cache and an XML dumper for inspection — a lot of care went into this.
- **Disciplined DI**: every feature is individually toggleable and disabled features have their definitions *removed* (`DrawSonataExtraExtension::load`), avoiding dead services; `container.excluded`-tagged leftovers are swept at the end.
- **No SQL injection**: all user-influenced query values go through bound parameters (`InFilter`, `RelativeDateTimeFilter`, `PreventDelete::createQueryBuilder`).
- **Batch controller** preserves Sonata's CSRF validation, confirmation flow and extension hooks while adding sub-request-based per-action controllers.
- Clean use of decoration (`CanSecurityHandler`, `EventDispatcherFormContractor`, `SubClassFieldDescriptionFactory`) to extend Sonata without forking behavior.

---

## Test Coverage

Roughly 1,050 lines of tests versus ~5,450 lines of source. Coverage is concentrated on wiring, not behavior:

- **Well covered**: `DependencyInjection` — `Configuration` tree and `DrawSonataExtraExtension` service add/remove logic have systematic per-feature tests (8 extension test files); `SessionTimeoutRequestListener` has a thorough 400-line unit test; `DecoratesCompilerPass` is tested.
- **Shallow**: `PreventDeleteExtension` has a single small test (access-restriction path only).
- **Untested in this package**: the entire ActionableAdmin runtime (`ObjectActionExecutioner`, `GenericFormHandler`, `AdminActionLoader`, all its event listeners including the CSRF validator), `BatchAdminController`, `PreventDeleteRelationLoader`/`PreventDelete` query building, both Doctrine filters, `GridExtension`, `AutoActionExtension`, `ListFieldPriorityExtension`, `DoctrineInheritanceExtension`, `WorkflowExtension`/`WorkflowTransitionAction`, both security voters and `CanSecurityHandler`, the notifier channel, and the Twig extension. Several of the bugs found above (H1, M2, M3) would likely have been caught by unit tests of those classes. Integration coverage may exist elsewhere in the monorepo, but within this package the security- and correctness-critical paths run on faith.

**Overall grade: C** — good architecture and code quality undermined by real defects in the security-sensitive features and by incomplete dependency declaration.
