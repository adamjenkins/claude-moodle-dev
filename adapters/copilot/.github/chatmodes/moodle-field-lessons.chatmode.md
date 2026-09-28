---
description: Use when writing, reviewing or testing Moodle plugin code, to avoid non-obvious pitfalls learned from shipping real plugins: version.php supported ranges and cross-branch API guards, services.php/tasks.php registration, authorisation and state-collision bugs, output escaping (html_writer, format_string, imported content), DB/XMLDB API small print, backup/restore/reset and DST-safe date shifting, privacy provider drift, lang string placeholders, forms and admin settings, PHPUnit/Behat blind spots, AMD/modal/TinyMCE gotchas, navigation and block placement, and CI false greens. Skip when you need a full checklist for one topic (use the sibling skill named in each section).
tools: ['codebase', 'search', 'editFiles', 'runCommands']
---
# Moodle Field Lessons

## Overview

Rules distilled from defects that shipped (or nearly shipped) in real Moodle
plugins despite passing phpcs, PHPUnit and CI. Each rule is short and
imperative, with the reason it matters. Topic checklists live in sibling
skills; this skill holds the small print those checklists do not spell out.
Longer rules for testing, forms, lang strings, JS and UI are in
[reference.md](reference.md).

## When to Use

- Writing or changing `version.php`, `db/*.php`, `install.xml`, external functions, restore steps or settings
- Reviewing a diff for things linters and unit tests cannot see
- Designing tests, especially security regression tests and date/timezone assertions
- Supporting several Moodle branches from one code line
- **Skip when:** you need the broad checklist for one topic: `moodle-security-audit`
  (security), `moodle-release-preflight` (before a release or external review),
  `moodle-definition-of-done` (before calling a task finished),
  `moodle-phpunit-testing`, `moodle-behat-testing`, `moodle-ci-matrix`,
  `moodle-5-3-changes`

## The most expensive mistakes

1. **Trusting the green suite.** PHPUnit cannot see: services/tasks never
   registered, free functions the bootstrap preloads, routed POST bodies,
   hook dispatch on module pages, backup element collisions, real form POSTs.
   Every such change needs one real HTTP request.
2. **Calling a newest-branch API across a supported range.** Your local site
   is one row of the CI matrix. Guard it or use the name every branch has.
3. **Changing `db/services.php` or `db/tasks.php` without a version bump.**
   Upgrade prints success; nothing is registered.
4. **Authorising against a client-supplied id**, or checking capability only
   at request time and not at the async sink.
5. **Two actors writing the same state value** to one column, so the
   lower-privileged one can undo the higher-privileged one's decision.
6. **Unescaped `html_writer::tag()` contents** and unclean restore input.
7. **A security test never seen failing.** It passes against vulnerable code
   when the fixture misses the column the renderer reads.
8. **DDL or payloads from a standalone CLI script** - `config.php` is the live DB.
9. **Fixed-seconds date shifts across DST** in restore/reset for wall-clock data.
10. **Hand-computed expected timestamps** in tests, then "fixing" correct code.

## Versioning and cross-branch support

- **`$plugin->supported` is an inclusive `[low, high]` pair**, even for one
  branch: `[502, 502]`, never `[502]`. A malformed value such as `[502]` makes
  core throw `coding_exception('Incorrect syntax in plugin supported
  declaration')` when it loads plugin info; third-party tooling may silently
  fall back to `requires` instead. Keep `requires` at or above the lowest branch.

  ```php
  $plugin->requires = 2025041400; // 5.0.
  $plugin->supported = [500, 502];
  ```

- **Guard every core API that changed inside your range** with
  `method_exists()`/`class_exists()`. Grep each stable branch for the symbol and
  for deprecation attributes; 5.0 keeps code under `lib/`, 5.1+ under
  `public/lib/`, so a single-path grep reports false absences.

  ```php
  if (method_exists(\core_courseformat\local\cmactions::class, 'delete')) {
      (new \core_courseformat\local\cmactions($course))->delete($cmid);
  } else {
      course_delete_module($cmid);
  }
  ```

  Old APIs emit deprecation notices that `--fail-on-warning` CI rejects on the
  newest branch; new ones fatal ("Call to undefined method") on the oldest.
- **When core namespaces a global class** (for example `navigation_node`
  gaining a namespaced name with a `class_alias`), keep using the global name
  until the lowest supported branch has the new one. A `use` of the new name
  in a hook listener breaks every page on older branches.
- **Raise `$plugin->dependencies`** from `ANY_VERSION` to the dependency's
  `$plugin->version` in the same commit that starts calling a newer symbol
  from it. `ANY_VERSION` only proves the plugin is installed.
- **Keep `version` and `release` separate.** Schema, services, tasks,
  capabilities, caches or messages changes bump the numeric `version`; bump
  the semantic `release` only as a deliberate release, with notes and a tag.
- **5.1+ `/public` layout:** core admin CLI stays at `<root>/admin/cli/`;
  plugin CLI (including `admin/tool/phpunit` and `admin/tool/behat`) lives
  under `<root>/public/`. Do not add `public/` to core CLI paths by analogy.

## Registration that only happens on upgrade

- **`db/services.php`, `db/tasks.php`** (and other `db/` definitions) are
  re-read only when the plugin's numeric version changed. Without a bump,
  `admin/cli/upgrade.php` says "No upgrade needed", AJAX calls fail
  (`invalidrecordunknown`) or the task never exists, and unit tests calling
  the class directly stay green. Verify the effect:

  ```php
  $info = \core_external\external_api::external_function_info(
      'local_myplugin_do_thing', IGNORE_MISSING);
  ```

  ```bash
  php admin/cli/scheduled_task.php --list | grep local_myplugin
  ```

- **PHPUnit init is a no-op** while the site-wide versions hash is unchanged:
  a table added to `install.xml` without a version bump never reaches the
  test DB. Rebuild with `util.php --drop` then `init.php`. "Initialised for
  different version" means some component changed on disk, not that yours
  is broken.
- **`$CFG->routerconfigured = true`** must be set in `config.php` above the
  `require_once` of `lib/setup.php`; setup normalises it early.

## Web services and routed controllers

- **`'loginrequired' => false` functions must not call
  `self::validate_context()`** - it ends in `require_login()`. Set the page
  context with `$PAGE->set_context()` instead.
- **Match the gate of the page the function backs.** Validating course
  context enforces enrolment; if the page requires ownership, unenrolled
  owners silently get `requireloginerror`. Test with a non-enrolled owner.
- **Routing Engine controllers without a `requestbody` schema** get
  `$request->getParsedBody() === []` in production (the request validator
  replaces it). Declare the schema, or read POST data with `optional_param()`
  plus `require_sesskey()`. `route_testcase::process_request()` does not run
  that middleware, so verify with a real HTTP POST.

## Authorisation and state

- **Never authorise against a client-supplied id.** Derive course, context and
  instance from the URL/context; re-validate each submitted id against the set
  you offered; re-check capability at async sinks (tasks, grade pushes);
  enforce size/duration limits server-side; do not ship answer or grading
  metadata to the browser.

  ```php
  $accountid = required_param('accountid', PARAM_INT);
  if (!array_key_exists($accountid, $offeredaccounts)) {
      throw new \moodle_exception('invalidaccount', 'mod_myplugin');
  }
  ```

- **One permission helper guarding several actions** is a smell: check each
  action wants that gate. Test actor -> control -> resulting state, and
  assert a lower-privileged subject cannot reverse a state a higher one set.
- **Grep every writer of a status column**, not only its readers, before
  reusing a value. Never let two actors of different privilege write the same
  value; give each a distinct state or record the actor on the row.
- **Widening who sees a stored field is a new output sink.** A column filled
  from `$e->getMessage()` is untrusted text (it disclosed a server path once
  shown to non-admins); show only plugin-composed messages.
- **Enumerate all entry points mechanically before an access audit** (every
  top-level and `admin/**` PHP file, every external function):
  `grep -rlE 'require_login|require_course_login' --include='*.php' .`
  Sibling controllers gated only by `require_login()` are where exports leak.
- **Serve uploads with force-download** unless the MIME type is validated
  audio/video; allowlist type/extension on upload; escape at output.
- **`db/access.php`: `clonepermissionsfrom` overrides `archetypes`** and copies
  the source capability's real grants. Fixing access.php later does not
  re-derive grants on existing sites; check `{role_capabilities}`.
- **Account-linking and token callbacks need a session-bound, single-use
  `state`** issued before the redirect, or they allow login CSRF.
- **Test system-context gates with a separate account or "Log in as"**, never
  "Switch role to" (course contexts only).
- **Inspect uploaded plugin ZIPs with core** (`\core\update\validator`,
  `\core\update\code_manager::unzip_plugin_file()`); never `include` an
  untrusted `version.php` - tokenise it with `token_get_all()` for the fields
  core's parser skips (`supported`, `dependencies`).
- **Probe suspected traversal/XXE live before filing it.** Core's zip extractor
  strips `../`; flag-less `simplexml_load_file()` is XXE-safe on PHP 8 with
  libxml >= 2.9 unless `LIBXML_NOENT`/`LIBXML_DTDLOAD` is passed.

## Output escaping

- **`html_writer::tag()`, `link()`, `select()` escape attributes, not contents.**
  Wrap user text in `s()` at the sink; do not `s()` an already
  `format_string()`-ed value (double-encodes `&`).

  ```php
  echo html_writer::tag('option', s($label), ['value' => $id]);
  ```

- **`format_string()` strips tags; `s()` escapes them.** Its `<`/`>` handling
  depends on the `formatstringstriptags` site setting (on by default), so
  "uses `format_string()`" is not an XSS argument on its own. Tests written
  against `s()` output break after migrating.
- **Content you write into core tables is rendered by core**, often `noclean`.
  `clean_text()` skips text without `<`, `>` or `&`, so a Markdown
  `[x](javascript:...)` imported as FORMAT_MARKDOWN survives. Clean every
  imported field and convert or clamp the format.
- **`fullname()`, `get_string()` and `html_writer` content escape at no
  layer**, so wrap the composed value in `s()`. `format_string(..., ['escape'
  => false])` cannot undo pre-encoded entities; use
  `html_entity_decode(format_string(...))` for plain-text sinks.
- **Before calling `{{{ }}}` a sink, read `export_for_template()`**; escaping
  often happens there. Prove findings with a payload through the real page.
- **In block classes use `$this->page`, never `global $PAGE`** (moodle-cs
  `moodle.PHP.ForbiddenGlobalUse.BadGlobal` fails CI).
- **Filters emitting tall inline content** (ruby annotations, math) inside
  `div.no-overflow` need top padding in `styles.css`; clipping shows in
  Firefox and is nearly invisible in Chromium:
  `.no-overflow:has(.filter_myplugin-tall) { padding-top: .75em; }`

## DB and XMLDB small print

- **`update_record()` writes every property.** Use `set_field()` or a scoped
  UPDATE near concurrently incremented counters.
- **`get_in_or_equal()` defaults to `?` params**; pass `SQL_PARAMS_NAMED` when
  the query uses named params ("Mixed types of sql query parameters!!").
- **`get_records()` has six parameters** and is keyed by the first field in
  `$fields` (`id` for `*`). A seventh "key by" argument is silently ignored;
  put the key column first in `$fields` or use `get_records_menu()`.
- **XMLDB foreign keys are indexes, not enforced FKs.** To change NOT NULL on a
  keyed column: `drop_key()` -> `change_field_notnull()` -> `add_key()`. Never
  raw `ALTER TABLE` with a hardcoded prefix.
- **Never declare `CHAR NOTNULL="true" DEFAULT=""`.** After any install.xml
  change, fresh-install and grep the output for `debugging()`; pair every
  install.xml fix with an upgrade.php step.
- **A new format/units column beside a value** widens the value's contract for
  every reader (other plugins, WS clients). Extend them or normalise on write.
- **Never run DDL or mutating probes from a standalone CLI script.** Use an
  `advanced_testcase` with `resetAfterTest()`; if a script must touch the DB,
  print `$CFG->dbname`/`$CFG->prefix` first.

## Backup, restore, reset and uninstall

- **Plan all of them for any data-storing plugin**: backup/moodle2, the reset
  triad (`*_reset_course_form_definition()`, `*_reset_course_form_defaults()`,
  `*_reset_userdata()`) plus gradebook reset, and `db/uninstall.php` for data
  outside your tables. Only `*_reset_userdata()` means no checkbox, so the
  deletion branch is unreachable.
- **Restore input is attacker input**: apply the same `clean_param()` /
  allowlist in every `process_*()` step as the forms and settings do.
- **Restore overwrites course settings, including `visible`.** Re-assert the end
  state after bulk writes (`course_change_visibility($id, false)`), and test it.
- **Wall-clock dates across DST:** restore/reset shifts add fixed seconds
  (09:00 becomes 08:00). Round the offset to days; shift in the right timezone.

  ```php
  $days = (int) round(($this->apply_date_offset(1) - 1) / DAYSECS);
  $new = (new \DateTimeImmutable('@' . $ts))->setTimezone($tz)
      ->modify("+{$days} days")->getTimestamp();
  ```

- **List every path that writes or shifts each timestamp** you compare
  (insert, restore offset, reset timeshift); restore shifts date fields but
  not snapshot times.
- **Build test `.mbz` files with core's packer**
  (`get_file_packer('application/vnd.moodle.backup')`), not `tar -czf`: a
  missing `.ARCHIVE_INDEX` fails with a misleading "missing moodle_backup.xml",
  so negative tests pass for the wrong reason. Do not name nested backup
  elements after columns of the same record.
- **Privacy:** every table in `get_metadata()` must be found by both
  discovery methods, exported by `export_user_data()`, and deleted by all
  three delete methods, with a test on generator rows; grep
  the provider after any table rename.

## Activity module lifecycle

- **Inside `<modname>_add_instance()` use `$data->coursemodule`**, never a
  lookup by instance: the `course_modules` row still has `instance = 0`.
- **Module calendar events with a `groupid`** are visible only to group
  members; a groupless teacher with `accessallgroups` sees none. Offer another view.
- **Declaring `FEATURE_COMPLETION_HAS_RULES` requires
  `classes/completion/custom_completion.php`**; test repeat and degenerate
  attempts, not only the happy path.
- **Question engine outside mod_quiz:** pass `slots` to
  `process_all_actions()` and filter post data to `get_field_prefix($slot)`,
  or every slot named in client data is graded.

## More lessons in reference.md

[PHPUnit traps](reference.md#phpunit-traps), [security regression tests](reference.md#security-regression-tests), [Behat and browser automation](reference.md#behat-and-browser-automation), [forms and admin settings](reference.md#forms-and-admin-settings),
[lang strings](reference.md#lang-strings), [JavaScript, AMD, modals, TinyMCE](reference.md#javascript-amd-modals-tinymce), [UI, CSS and themes](reference.md#ui-css-and-themes),
[navigation, blocks and pages](reference.md#navigation-blocks-and-pages), [core API small print](reference.md#core-api-small-print), [CI false greens](reference.md#ci-false-greens).

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `$plugin->supported = [502];` | Core throws `coding_exception`; some tooling falls back to `requires` | `[502, 502]` |
| services.php edit, no version bump | Function never registered | Bump `version`, check `external_function_info()` |
| `validate_context()` in a public WS | Anonymous callers refused | `$PAGE->set_context()` |
| `html_writer::tag('td', $name)` | Stored XSS | `s($name)` |
| `get_coursemodule_from_instance()` in add_instance | "Can't find data record", creation rolled back | `$data->coursemodule` |
| Shared "hidden" state for author and moderator | Moderation bypass | Distinct states or record the actor |
| `.finally()` on a `core/ajax` promise | Works once, then control stays disabled | `Promise.resolve(...)` wrapper |
| `PARAM_INT` on a submit button | Action silently does nothing | `PARAM_RAW`, test `!== ''` |
| `tar -czf` for a tampered .mbz | Negative test passes for the wrong reason | Core's backup packer |
| Expected timestamps computed by hand | Correct code "fixed" to match a wrong test | Derive with an independent tool |

## References

Moodle developer docs: [version.php](https://moodledev.io/docs/apis/commonfiles/version.php), [web services](https://moodledev.io/docs/apis/subsystems/external), [routing](https://moodledev.io/docs/apis/subsystems/routing),
[output and escaping](https://moodledev.io/docs/apis/subsystems/output), [DML](https://moodledev.io/docs/apis/core/dml), [backup](https://moodledev.io/docs/apis/subsystems/backup),
[privacy](https://moodledev.io/docs/apis/subsystems/privacy), [access](https://moodledev.io/docs/apis/subsystems/access).
