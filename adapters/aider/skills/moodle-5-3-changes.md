# moodle-5-3-changes

> Use when upgrading a plugin or theme to Moodle 5.3, targeting 5.3 (branch 503, MOODLE_503_STABLE), setting $plugin->requires for 5.3, or fixing 5.3 deprecations and breakages — user/lib.php functions moved to \core\user, legacy external_* classes and functions, FEATURE_GROUPMEMBERSONLY, Classic theme removal, Boost dark colour mode, React navigation and block_timeline, theme_boost/bootstrap/* imports, report builder sortable/set_main_table changes, duration element units, quiz report group selector, assign overrides and marker allocation, adhoc task ids, login/token.php hardening. Skip when working on 5.2 or earlier only, or for general plugin authoring (use moodle-plugin-development).

# Moodle 5.3 Changes

## Overview

A developer digest of what changed in Moodle 5.3 and what plugin and theme
authors must do about it. Breaking changes and deprecations come first, then
new capabilities, then an upgrade checklist.

**Status:** based on the Moodle **5.3beta (Build: 20260916)** `UPGRADING.md`
and the MDL tracker issues it cites, checked against the 5.3beta source. It
will be re-verified against 5.3.0 final. Where the notes and the beta code
disagree, this skill follows the code and says so.

See [reference.md](reference.md) for the full per-component catalogue — every
5.3beta upgrade note, with actions, snippets and MDL links.

## When to Use

- Porting a plugin or theme to 5.3, or adding MOODLE_503_STABLE to CI
- A 5.3 site shows debugging notices, `coding_exception`s or install failures
  from plugin code
- Choosing replacements for APIs deprecated in 5.3
- Deciding whether a new 5.3 API (React components, colour modes, PATs, OAuth2
  scopes) is usable given your supported versions
- **Skip when:** the plugin only supports 5.2 or earlier, or the question is
  general Moodle plugin structure (use `moodle-plugin-development`,
  `moodle-upgrade-migration`)

## Platform and requirements

| Item | 5.3 value |
|---|---|
| `$branch` | `503` |
| Beta version number | `2026091600.00` (`5.3beta (Build: 20260916)`) — re-check at 5.3.0 |
| PHP | 8.3.0 minimum |
| Databases | MariaDB 11.4, MySQL 8.4, PostgreSQL 17, SQL Server 15.0 (MariaDB was 10.11 and PostgreSQL 16 in 5.2) |
| Upgrade from | Moodle 4.4 or later |
| Composer | `composer install` was already required in 5.2; 5.3 adds `league/oauth2-server` (MDL-88457), and OAuth2 server admin pages check it via the new `\core\composer` status API (MDL-88576), so re-run `composer install` after upgrading |

The database minimums come from `admin/environment.xml`, not UPGRADING.md.
At the time of the beta the moodledev.io 5.3 release page listed older values
(PostgreSQL 16, MariaDB 10.11, upgrade from 4.5); treat `environment.xml` as
authoritative and re-check at release.

For CI: drop MariaDB < 11.4 and PostgreSQL < 17 from MOODLE_503 jobs.

## Breaking changes and removals

Code that will fail, throw, or visibly break on 5.3:

| Area | Change | Action |
|---|---|---|
| core (MDL-83231) | `plugin_supports(..., FEATURE_GROUPMEMBERSONLY)` always throws `coding_exception`; installing/upgrading a module whose `<mod>_supports('groupmembersonly')` returns true throws `plugin_defective_exception` | Delete the `case FEATURE_GROUPMEMBERSONLY:` |
| core_external (MDL-81225, MDL-76583) | `external_generate_token()`, `external_create_service_token()`, `external_delete_descriptions()`, `external_validate_format()`, `external_format_string()`, `external_format_text()`, `external_generate_token_for_current_user()`, `external_log_token_request()` are final-deprecated stubs and throw | Use `\core_external\util::` methods |
| core_external | Global `\external_api`, `\external_value`, `\external_single_structure` etc. are no longer `class_alias`es in `lib/externallib.php`; they resolve via the renamed-classes map | `use core_external\external_api;` etc.; drop `require_once($CFG->libdir . '/externallib.php')` |
| login/token.php (MDL-87010) | Credentials in the query string throw; `appsitecheck` parameter removed; service is validated before authentication | POST credentials; don't probe with `appsitecheck=1` |
| core_form (MDL-89434) | `duration` element with `units` throws if the effective `defaultunit` (default `MINSECS`) is not in `units` | Set `defaultunit` to one of your `units` |
| core_reportbuilder (MDL-87404) | Columns are sortable by default | Add `->set_is_sortable(false)` where sorting must stay off |
| core_reportbuilder (MDL-88397) | `set_main_table($table, $alias)` — alias has no default | Always pass an alias |
| core_ai (MDL-89123) | `prompttokens`/`completiontoken(s)` moved from `ai_action_*` child tables to `ai_action_register` | Update direct SQL |
| mod_quiz (MDL-81096) | Group selector no longer printed for custom quiz reports | Call `$this->print_action_bar(...)` (see below) |
| mod_assign (MDL-87709) | `marker_updated` event no longer triggered by core | Observe `marker_added` / `marker_removed` |
| core_courseformat (MDL-88410) | Collapse/expand-all toggle moved from `content\section` output + template to `content` | Move overrides to `local/content.mustache` |
| core_courseformat (MDL-88949) | Course index subsection ARIA moved to the activity `treeitem` | Update both `courseindex/cm` and `courseindex/section` overrides |
| block_timeline (MDL-88287) | Rewritten in ESM + React; `output\main`, `output\renderer`, AMD modules and templates removed without stubs | Drop overrides of the old renderer/templates |
| theme (MDL-88351) | Classic theme removed from core and uninstalled on upgrade; customised settings are migrated to Boost only if Classic was the site default theme | Re-parent Classic child themes or install Classic separately *before* upgrading |
| theme_boost (MDL-89050) | `drawer` template block `{{$drawerheadercontent}}` replaced by `{{$drawercontrols}}` | Rename the block in overriding templates |
| Navigation (MDL-87830, MDL-89294) | Primary and secondary nav rendered by React `core/nav/Nav` / `PrimaryNav`: `.mds-nav-pill` instead of `.nav-link`, no `.moremenu`; Boost navbar includes `core/primarymoremenu` | Retarget CSS/JS/Behat selectors; child navbars switch to `core/primarymoremenu` |
| Modals (MDL-75699) | `core/modal` title is `<h2 class="modal-title fs-5">`, was `<h5>` | Body headings start at `<h3>`; fix selectors on `h5.modal-title` |
| core_grades (MDL-89497, MDL-88407) | Courses with penalty-deducted grades are frozen on upgrade until a grade manager keeps or applies the fix | Warn admins; penalty-applying plugins re-test grading |
| tool_task (MDL-86422) | `queue_adhoc_task($task, true)` returns the existing task's id instead of `false` for a duplicate | Don't use truthiness to mean "newly queued" |
| Exporters (MDL-79755, also 5.2.2+) | Formatted exporter strings use numeric entities (`&#38;` not `&amp;`) | Update test assertions and JS comparisons |
| Block uninstall (MDL-89289) | Instances deleted later by an ad-hoc task | Put per-instance cleanup in `instance_delete()`; don't assume contexts are gone |
| Linear navigation (MDL-89406) | Formats returning true from `uses_linear_navigation()` get the prev/next footer **on by default** unless they add a `format_<name>/enablelinearnav` admin setting | Add the setting if you need an off switch |

Quiz report minimal fix:

```php
// In your quiz report's display() — mod_quiz\local\reports\report_base subclass.
$this->print_action_bar('myreport', null, $cm, $reporturl);
```

`duration` element fix:

```php
$mform->addElement('duration', 'timelimit', get_string('timelimit', 'quiz'),
    ['units' => [HOURSECS, DAYSECS], 'defaultunit' => HOURSECS]);
```

## Deprecations (with replacements)

Still working in 5.3 but emitting notices (or scheduled for removal):

| Deprecated | Replacement | MDL |
|---|---|---|
| 25 `user/lib.php` functions (`user_create_user()`, `user_update_user()`, `user_delete_user()`, `user_get_user_details()`, `user_can_view_profile()` …) | `\core\user::create_user()` etc. (name without `user_` prefix); `user_update_device_public_key()` → `\core_user\devicekey::update_device_public_key()` | MDL-82650 |
| `theme_boost/bootstrap/*` AMD imports | `import {Tooltip} from 'bootstrap';` (5.3+) or `from 'theme_boost/index'` (5.2-compatible, until 7.0) | MDL-88766 |
| `core_courseformat\base::get_return_section()` | `get_page_section()` | MDL-86284 |
| `get_view_url()` option `'sr'` | `'pagesectionid' => $section->id` (docblock-only, no notice) | MDL-86284 |
| report builder `get_main_table()` | `get_main_table_sql()` (includes `{braces}`) | MDL-88397 |
| `extend_user_menu::add_navitem()` / `get_navitems()` | `add_menu_item(\core_user\output\user_action_menu\link ...)` / `get_menu_items()` | MDL-88938 |
| `grade_item::update_deducted_mark()` | none — `\core_grades\penalty_manager` applies penalties | MDL-88407 |
| `\core\hub\registration::get_dataroot_size()` | `get_filepool_usage()` | MDL-88805 |
| `core/external_content_banner` template | `core_admin/notification_ctas` | MDL-89290 |
| `core_admin_renderer::admin_notifications_page()` | `notifications_page()` (three banner args removed) | MDL-89290 |
| `core_admin_renderer` `campaign_content()`, `services_and_support_content()`, `userfeedback_encouragement()`, `marketplace_integration_notice()` | customise `core_admin/notification_ctas` | MDL-89290 |
| `core_admin_renderer::upgradekey_form_page()` | `upgradekey_form_page_with_validation($url, false)` | MDL-87896 |
| `$CFG->showcampaigncontent` | no effect; hide cards with `$CFG->disablenotificationctas` | MDL-89290 |
| `assign::delete_override()`, `assign::delete_all_overrides()`; global `move_group_override()`, `reorder_group_overrides()` (removal in 6.0, MDL-87324) | `\mod_assign\override_manager` (`delete_overrides_by_id()`, …) | MDL-86513 |
| `assign::get_allocated_markers()` / `update_allocated_markers()` | `get_marker_allocations()` / `update_marker_allocations()` (different 2nd arg) | MDL-87709 |
| `ASSIGN_MULTIMARKING_MAX_MARKERS` | `ASSIGN_MULTIMARKING_DEFAULT_MAX_MARKERS` | MDL-87709 |
| `enrol_manual_plugin::enrol_cohort()` | none (use enrol_cohort) | MDL-89439 |
| `\core\task\manager::task_is_scheduled()` | `get_queued_adhoc_task_record($task, false)` | MDL-86422 |
| Old grade action bar templates | `core/navigation_action_bar`, `core/action_bar` | MDL-81096 |
| Behat `I set portfolio instance "X" to "Y"` | `I set the portfolio instance "X" to "Y"` | MDL-89069 |
| `NO_MOODLE_COOKIES` checks | `\core\session\manager::supports_cookies()` | MDL-87174 |

Renamed without notices (old names keep working): ~110 adminlib classes moved
to `\core_admin\setting\...` (MDL-81935), e.g. `admin_setting_configtext` →
`\core_admin\setting\setting\configtext`. Keep the old names while you
support 5.2.

A version-spanning replacement:

```php
// Moodle 5.3+ only.
$userid = \core\user::create_user((object) $data);
// Supporting 5.2 too: keep user_create_user() (with require_once of
// user/lib.php) until your minimum is 5.3; it still works, with a notice.
```

## New capabilities by area

| Area | What's new | MDL |
|---|---|---|
| Theming | Boost dark colour mode (experimental `theme_boost/enablecolourmodes`), `data-bs-theme` on `<html>`, `theme_boost\colour_mode`; TinyMCE follows the mode | MDL-68037 |
| Theming | Self-hosted Noto Sans default font (+ cyrillic, Noto Sans JP for `:lang(ja)`); inline navbar search | MDL-88412, MDL-89024 |
| React | `html_writer::react_component()`, `react_component_renderable`, AMD `core/import` and `core/component`, `scripts/swizzle.mjs` | MDL-89296, MDL-88505, MDL-88509 |
| Output | `$PAGE->set_show_navigation_footer()`, `set_has_sticky_footer()`, `set_supplementary_content()`; `core\output\submenu`; `subpanel` URL; `notification_base` `headinglevel` | MDL-87575, MDL-87302, MDL-88601, MDL-88312, MDL-88458 |
| Hooks | `\core\hook\email\before_email_to_user` — edit, add headers, or block outgoing mail | MDL-69724 |
| DI | `#[\DI\Attribute\Inject]` properties; `\core\authentication`, `\core\authentication\password`, `\core_auth\validate_user`, `\core\composer` services | MDL-89528, MDL-88580, MDL-88576 |
| REST API | Personal access tokens (`\core\api\token_manager`, `moodle/api:createtoken`), OAuth2 server clients, route scopes `#[scopeset]` / `#[unscoped_resource]` | MDL-87706, MDL-89181, MDL-89089 |
| Web services | `'allowcorsrequests' => true` for no-login AJAX functions; `mod_assign_*_overrides`, `mod_forum_set_read_state`, `mod_quiz_get_users_in_report` | MDL-87150, MDL-86513, MDL-87887, MDL-81096 |
| Exceptions | `new moodle_exception(..., previous: $e)` | MDL-88579 |
| Tasks | `adhoc_task::set_soft_retry_delay()` (also 5.2.2+); `set_scheduled_task_nextruntime()` returns bool | MDL-79763, MDL-89200 |
| Course formats | `uses_linear_navigation()`, `inline_help` option key, disabled activity chooser items | MDL-87302, MDL-88669, MDL-87373 |
| Report builder | `week`/`month`/`year` aggregations, `set_main_table_sql()`, auto `prepend_joins`, `add_fields(array)` | MDL-84635, MDL-88397, MDL-87405, MDL-89004 |
| Accessibility | `core/imagedetails/modal` `getImageDetails(file)`; Behat `I set the focus on`, `--colourmode=dark` runs | MDL-89214, MDL-84065, MDL-68037 |
| Testing | PHPUnit `util.php --snapshot`, `--restore=NAME`, `--upgrade` | MDL-88495 |
| Grades | Scale-less outcomes linked to course modules | MDL-88881 |

Dark-mode-safe plugin CSS:

```css
.local_myplugin-status {
    background-color: var(--bs-tertiary-bg);
    color: var(--bs-body-color);
    border: 1px solid var(--bs-border-color);
}
[data-bs-theme="dark"] .local_myplugin-status {
    /* only genuine per-mode differences here */
}
```

Email hook listener (`db/hooks.php`):

```php
$callbacks = [
    [
        'hook' => \core\hook\email\before_email_to_user::class,
        'callback' => [\local_myplugin\hook_listener::class, 'before_email'],
    ],
];
```

```php
public static function before_email(\core\hook\email\before_email_to_user $hook): void {
    if (str_ends_with($hook->email->user->email, '@example.invalid')) {
        $hook->email->add_block_reason('Recipient domain is blocked');
    }
}
```

## Upgrade checklist

1. Grep for `FEATURE_GROUPMEMBERSONLY` — remove it.
2. Grep for `external_generate_token`, `external_format_`, `external_validate_format`, `external_delete_descriptions`, `external_log_token_request` — replace with `\core_external\util::`.
3. Grep for bare `external_api`, `external_value`, `external_single_structure`, `external_multiple_structure`, `external_function_parameters` — switch to `use core_external\...`.
4. Grep for the 25 `user/lib.php` functions — migrate or plan to when your minimum is 5.3.
5. Grep `'duration'` form elements with `'units'` — check `defaultunit`.
6. Report builder: pass an alias to `set_main_table()`; audit sortability; replace `get_main_table()`.
7. Quiz report subplugins: call `print_action_bar()`; implement `has_permission()`.
8. Grep AMD `src/` for `theme_boost/bootstrap/`; rebuild with grunt.
9. Themes: remove Classic parentage; check `drawerheadercontent`, `.nav-link`/`.moremenu`, `h5.modal-title`, `core/loginform`, courseindex and `local/content` overrides.
10. Styles: replace `bg-white`, `text-dark`, literal colours with `--bs-*` variables; run Behat once with `--colourmode=dark`.
11. Event observers on `mod_assign\event\marker_updated` → `marker_added` / `marker_removed`.
12. Callers of `queue_adhoc_task($task, true)` that test the return value.
13. Course formats using linear navigation: add `format_<name>/enablelinearnav` if needed.
14. CI: add MOODLE_503_STABLE with PHP 8.3+, PostgreSQL 17 / MariaDB 11.4.
15. qbank plugins with bulk actions: override `get_action_icon()` (base returns `i/empty`) — MDL-73051.
16. Run with `DEBUG_DEVELOPER` and fix every deprecation notice from your component.

## Common mistakes

| Mistake | Fix |
|---|---|
| Replacing `user_create_user($arr)` with `\core\user::create_user($arr)` | New methods are typed: cast with `(object) $arr` |
| Setting `$plugin->requires` to 5.3 just to use `\core\user::` or `bootstrap` imports while claiming 5.2 support | Keep old calls (or `theme_boost/index`) until the minimum really is 5.3 |
| Treating `set_columnheadersattributes()`, `add_header_attributes()`, soft retry delay as 5.3-only | They are also in 5.2.x: soft retry delay from 5.2.2, `set_columnheadersattributes()`/`add_header_attributes()` from 5.2.3 |
| Using `\core\router\attributes\route` from the notes | The attribute class is `\core\router\route` (`cookies:` option) |
| Calling `\core\authentication::get_plugin()` statically | Instance methods: `\core\di::get(\core\authentication::class)->get_plugin($auth)` |
| `json_encode()`-ing props for `html_writer::react_component()` | Pass the array/object; the method encodes it |
| Assuming `ai_action_register.courseid` is always a course | `0` = not yet backfilled, `-1` = context not in a course |
| Assuming routes without `#[scopeset]` are rejected | Runtime applies no scope restriction; declare scopes explicitly |
| Adding `'allowcorsrequests' => true` broadly | Only for `loginrequired => false` functions safe cross-origin |
| Looking for `override_manager::delete_override()` | The method is `delete_overrides_by_id()` |
| Hard-coding `fill` in SVGs shown via `<img>` | Use the pix/icon API so icons inherit `currentColor` in dark mode |

## References

- https://moodledev.io/docs/5.3/devupdate
- https://moodledev.io/general/releases/5.3
- Root `UPGRADING.md` in the Moodle 5.3 codebase (section `## 5.3beta`)
- https://tracker.moodle.org/browse/MDL-82650 (user/lib.php → `\core\user`)
- https://tracker.moodle.org/browse/MDL-81225 (legacy external classes)
- https://tracker.moodle.org/browse/MDL-68037 (colour modes)
- https://tracker.moodle.org/browse/MDL-88351 (Classic removal)
- https://tracker.moodle.org/browse/MDL-88766 (Bootstrap imports)
- https://tracker.moodle.org/browse/MDL-86887 (5.3 environment requirements)
- [reference.md](reference.md) — full per-component catalogue
