# Moodle Field Lessons - reference

Overflow rules for [SKILL.md](SKILL.md). Same format: imperative rule, then why.

## PHPUnit traps

See also `moodle-phpunit-testing`.

- **Know PHPUnit's blind spots and add one real web request** for: free
  functions from `course/lib.php`, `lib/filelib.php`,
  `question/engine/lib.php` (the PHPUnit bootstrap preloads them, the browser
  does not - add explicit `require_once`), routed POST handlers, hook and
  navigation dispatch claims, draft file areas, backup/restore, and external
  HTTP integrations.

  ```php
  require_once($CFG->dirroot . '/course/lib.php');
  require_once($CFG->libdir . '/filelib.php');
  ```

- **`assign_capability()` defaults to `$overwrite = false`** and does nothing if
  a row exists (for example the archetype's CAP_ALLOW). Pass `true`, clear
  caches, and assert the gate actually flipped before testing it.

  ```php
  assign_capability('mod/myplugin:view', CAP_PROHIBIT, $roleid, $context->id, true);
  accesslib_clear_all_caches_for_unit_testing();
  $this->assertFalse(has_capability('mod/myplugin:view', $context, $user));
  ```

- **`block_<name>` classes are not autoloaded.** Load them first;
  `block_load_class()` requires both `block_base` and the block file.

  ```php
  require_once($CFG->libdir . '/blocklib.php');
  block_load_class('myplugin');
  $block = new \block_myplugin();
  $block->init();
  ```

- **Assorted traps:** `$cm->id` is a numeric string (use `assertEquals` or
  cast); `$PAGE->cm` is a magic property; `CLI_SCRIPT` is true under PHPUnit,
  so a CLI-only guard short-circuits positive tests; your own observers (for
  example on `course_deleted`) may delete fixtures the test built.
- **Some module generators need a real user**: call `$this->setAdminUser()`
  before `create_module('workshop', ...)` ("No guests here!"). mod_workshop
  creates two grade items per instance - a ready multi-grade-item fixture.
- **Generators append modules to the end of a section.** To test another
  order, set `course_sections.sequence` and rebuild the cache.

  ```php
  $DB->set_field('course_sections', 'sequence', "$cm2,$cm1", ['id' => $sectionid]);
  rebuild_course_cache($course->id, true);
  ```

- **`route_testcase` group paths:** pass `''` as `$grouppath` to
  `add_class_routes_to_route_loader()`; `ROUTE_GROUP_PAGE` builds a `//path`
  pattern that never matches (every expected-404 test then passes for the
  wrong reason), and `null` makes the test case guess from the namespace. Always include a 200 case.
- **`\core\task\grade_cron_task` locks nothing** in courses whose course grade
  item has `needsupdate = 1`. Call `grade_regrade_final_grades($courseid)`
  before running it in synthetic setups.
- **Derive expected timestamps independently** whenever DST, a timezone or a
  window boundary is involved, and paste the command into a comment. When a
  new test fails, first ask whether the test or the code is wrong.

  ```php
  // python3 -c "from datetime import datetime; from zoneinfo import ZoneInfo;
  //   print(int(datetime(2026, 11, 3, 9, tzinfo=ZoneInfo('Europe/London')).timestamp()))"
  ```

- **Users made outside the generator** (for example `create_user_record()`)
  need firstname, lastname and required profile fields, or `require_login()`
  redirects every request to `/user/edit.php` and scripted POSTs return an
  innocent-looking 303.
- **Multilang in tests:** set `$SESSION->forcelang` after `setUser()` (which
  clears it); `force_current_language()` no-ops without an installed pack.
- **Test helpers need full docblocks** (description, blank line, `@param`,
  `@return`); run phpcs and moodlecheck on test files too.

## Security regression tests

See also `moodle-security-audit` and `moodle-release-preflight`.

- **A security test is evidence only after you saw it fail with the fix
  reverted, for the right reason.** Build the fixture from the columns the
  renderer actually reads, and assert the exact sink position with its
  surrounding markup, not a page-wide substring that an escaped attribute next
  door can satisfy.

  ```php
  $this->assertStringContainsString('>' . s($payload) . '</option>', $html);
  ```

- **Negative tests assert the specific error**, not just "rejected": a forged
  backup rejected as malformed proves nothing about the check under test.

## Behat and browser automation

See also `moodle-behat-testing`.

- **Gradebook navigation:** the action-bar selector is a
  `li[role=option][data-value]` listbox with no anchors. Use core's step with
  `@javascript`: `And I navigate to "Setup > Gradebook setup" in the course gradebook`.
- **`I navigate to "A > B" in site administration` does not fail on an
  ambiguous path**; two plugins' "General settings" pages collide. Navigate to
  your plugin's category, then follow the page link.
- **Generator-created rich-text activities** need an explicit HTML format
  (for example `contentformat` 1) for TinyMCE to attach.
- **Run local Behat with `--scss-deprecations`** as CI does; Bootstrap 4 class
  names fail there even on core steps.
- **Checkboxes outside a moodleform** (Mustache bulk-select tables) need the
  HTML5 `form="<formid>"` attribute on each input; in non-JS Behat, check the
  field rather than clicking it.
- **Browser automation (Playwright and similar):** wait for network idle
  before filling a password field - core's `togglesensitive` module replaces
  the input via `outerHTML`, discarding early input - and assert the value
  before submitting. Select buttons by role and name (submit buttons are never
  unique); read `getAttribute('action')`, not `form.action` (shadowed by a
  hidden input named `action`); quiz answer radios have no `label[for]`.

## Forms and admin settings

- **Named submit buttons: `PARAM_RAW` and presence**, never `PARAM_INT` (the
  translated label cleans to 0). Put numeric flags in hidden inputs.

  ```php
  if (optional_param('confirm', '', PARAM_RAW) !== '' && confirm_sesskey()) {
      // Act.
  }
  ```

- **In `moodleform::validation()` never call `get_new_filename()`,
  `get_file_content()` or `save_file()`** - they recurse into validation to an
  out-of-memory fatal. Read `$data` (a filepicker value is its draft itemid)
  or `$files`.
- **A plain `<input type="file">` does not fill a draft area**; only the JS
  filepicker does. Outside moodleform, store the upload into the user's draft
  area yourself before `file_save_draft_area_files()`, which otherwise saves
  nothing and reports success.
- **`admin_setting_configmulticheckbox` is unticked until first saved**; the
  default only feeds the "Default:" hint. Fall back only on a strict `false`
  (an admin who unticks everything stores `''`). Keep choice labels plain text.

  ```php
  $value = get_config('local_myplugin', 'options');
  if ($value === false) {
      $value = $defaults;
  }
  ```

- **Tool plugin `settings.php` receives only `$ADMIN`**; create your own page
  inside `if ($hassiteconfig)` or `$settings->add()` on null crashes upgrade.

  ```php
  if ($hassiteconfig) {
      $settings = new admin_settingpage('tool_myplugin_settings',
          get_string('pluginname', 'tool_myplugin'));
      $ADMIN->add('tools', $settings);
      if ($ADMIN->fulltree) {
          // Add settings.
      }
  }
  ```

- **An `admin_externalpage` guarded by a custom capability** goes outside the
  `if ($hassiteconfig)` block (core precedent: report_log), or non-admin
  holders of the capability get accessdenied.

## Lang strings

- **Match `{$a}` vs `{$a->prop}` to the argument shape** and test the rendered
  message; a scalar argument leaves `{$a->title}` literally in the text.

  ```php
  $sink = $this->redirectMessages();
  // Trigger the notification.
  $this->assertStringNotContainsString('{$a', $sink->get_messages()[0]->subject);
  ```

- **Prove every referenced key exists** (including dynamically built ones)
  and assert rendered pages contain no `[[`. Keep keys sorted and check the
  order programmatically for the whole file.
- **Values in International English** (Licence, organisation); identifiers
  follow core's US spelling (`license`).
- **"Is X required?"** - check both core's code path and the moodledev.io
  plugin-type docs and contribution checklist; filters need `pluginname` and
  `filtername` even though core filters omit one.
- **Multilang:** `format_string()` filters only when the filter applies to
  "Content and headings"; `format_text()` always does. Wrap only inline
  content in `<span lang>`, one pair inside each block element (`<div lang>`
  is ignored; spans around blocks are emptied by HTMLPurifier).

## JavaScript, AMD, modals, TinyMCE

See also `moodle-amd-javascript`.

- **Never chain `.finally()` onto a `core/ajax` promise** (jQuery Deferred);
  it throws while the chain is built, so the request works once and the
  button stays disabled.

  ```javascript
  Promise.resolve(Ajax.call([{methodname: 'local_myplugin_do', args: {}}])[0])
      .then(render)
      .catch(Notification.exception)
      .finally(() => {
          button.disabled = false;
      });
  ```

- **AMD names resolve only to `amd/build/<name>.min.js`** (no index fallback).
  A dynamic `import()` in `amd/src` becomes a failing RequireJS call; vendor ES
  modules as `.js`, not `.mjs`.
- **5.2+ ESM bundler path** (`js/esm/src/*.ts|tsx`, built by `grunt react`)
  picks up only `.ts`/`.tsx` and cannot serve `.wasm` (not re-verified).
- **Modal cleanup:** `destroy()` removes the root before triggering
  `ModalEvents.destroyed`, so a handler bound on `getRoot()` never fires. Bind
  elsewhere, and do not tear down only in the save handler (Cancel, Escape and
  backdrop bypass it).
- **TinyMCE buttons appear only via `configuration.js` `configure()`**;
  `addButton()` alone only makes them referenceable. Empty stubs need a
  comment body for eslint.
- **Wall-clock to timestamp in JS:** re-measure the zone offset at the
  candidate instant; a single offset is off by the DST amount near transitions.

## UI, CSS and themes

See also `moodle-theme-development`.

- **Bootstrap 5 names on 4.4+/5.x** (`visually-hidden`, `me-2`, `fw-bold`),
  including JS-built markup. `{{#pix}}` is icon-sized only. Never put tag
  syntax inside `{{! }}` comments - they close at the first `}}`.
- **Check every CSS custom property exists** on the target theme
  (`getComputedStyle(el).getPropertyValue('--x')`): Boost on 5.2 exposes
  `--bs-*`; the Moodle app uses `--ion-text-color-step-N` and `:root.dark`.
  Use `animation-fill-mode: backwards`.
- **Plugin `styles.css` is merged site-wide** (plugins, parent themes, theme),
  so scope selectors tightly; new files appear only after a cache purge.
- **H5P content** takes CSS only from the `core_h5p | h5pcustomcss` setting.

## Navigation, blocks and pages

- **Module-page secondary navigation from a non-activity plugin:** use
  `<component>_extend_settings_navigation()` and add under
  `$nav->find('modulesettings', settings_navigation::TYPE_SETTING)` in
  CONTEXT_MODULE. `core\hook\navigation\secondary_extend` fires only for course
  navigation (verified on 5.2).
- **Content on course-page activity cards** from a non-activity plugin: gate on
  `$PAGE->url`, not pagetype (other pages set `course-view-<format>`); filter
  `uservisible` and `is_visible_on_course_page()`; pick the anchor from core's
  template; re-apply after reactive reloads. Course page order differs from
  modinfo order: subsection content comes last. Render on
  `before_footer_html_generation` (5.0-5.2, no dedicated hook);
  `sort_cm_array()` exists only from 5.0.3.
- **A block cannot choose its default region**; the page and the "Add a block"
  trigger decide. Document a manual move instead of patching the DB.
- **`$USER` is cached in the session** - DB changes to the user (for example
  `lang`) do not reach already-logged-in sessions.
- **Moodle app site plugins:** `updateContent()` is a cached read - pass
  `getFromCache: false, saveToCache: false, emergencyCache: false` for "next
  item" flows; `otherdata` starting with `{`/`[` arrives parsed; call
  `detectChanges()` after awaits; recreate `core-question` with `*ngIf` to show
  new HTML. See `moodle-mobile-app`.

## Core API small print

- **Core string `datechanged` is deprecated from 5.2** (lang/en/deprecated.txt);
  reset status items use the module's own string.
- **`user_create_user($user, false)` stores the password verbatim**; pass `true`.
- **`core_external\util::generate_token()` always mints a new row** - track
  every token you mint for revocation.
- **`assign::save_grade()` outside the grading form** needs
  `$data->attemptnumber = 0; $data->applytoall = 0;`.
- **A throwing `finally` replaces the original exception**; keep cleanup
  non-throwing.
- **Another user's timezone:** if `empty($tz) || $tz == 99`, use
  `core_date::get_server_timezone()`; `get_user_timezone(99)` resolves via the
  viewer's `$USER->timezone`.
- **Resolve a class's FQN from its `namespace` line**, not its path;
  `class_exists()` autoloads as a side effect, so probe one name per process.
- **Do not use PCRE `\p{Han}` to mean ideographs** - it also matches `、` and
  `。`. Use explicit ranges and print sample matches before trusting counts.
- **Keep `defined('MOODLE_INTERNAL') || die();` only in files that run code
  at require time**; let phpcs decide, and run it on code copied from others.
- **No scaffolding placeholders in headers**; one canonical `@copyright` line.
- **Without `moodle/backup:userinfo`** the backup users setting is a hidden
  input, not an unticked checkbox.

## CI false greens

See also `moodle-ci-matrix` and `moodle-definition-of-done`.

- **Derive every matrix cell** (Moodle x PHP x DB) from each branch's
  `admin/environment.xml`; never cross current PHP/DB with old branches. Check
  service images, `include` entries missing axes, and artifact-name characters.
- **Vacuous passes:** an empty `tests/` passes phpunit/behat; phpmd exits 0
  with violations; `mustache` needs an `Example context (json):` block per
  template; `.mustachelintignore` silences only HTML validation;
  `grunt --force` hides eslint failures and stale builds.
- **No `error_log()` in best-effort catch blocks** (a forbidden function).
  Treat "Unexpected debugging() call" or deprecation notices in a green run as
  findings; `--fail-on-warning` will fail them later.
