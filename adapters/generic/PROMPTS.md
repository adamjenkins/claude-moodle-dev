# Moodle Development Prompts (agent-agnostic)

Drop these into any LLM coding assistant. Each section is self-contained.

## Skills (load on relevant tasks)

### moodle-5-3-changes

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


---

### moodle-accessibility

> Use when ensuring Moodle plugin UI meets WCAG 2.1 AA — semantic HTML in Mustache, ARIA via core helpers, keyboard navigation, color contrast in SCSS, focus management in modals, screen reader testing, and Pa11y/axe automation.

# Moodle Accessibility (WCAG 2.1 AA)

## Overview

Moodle targets WCAG 2.1 AA. UI in plugins must follow same standard for inclusion. Boost theme + Bootstrap 5 + Moodle's component library do most of the heavy lifting — your job is to use them correctly.

## When to Use

- Building any UI (Mustache template, modal, custom form element)
- Reviewing PR for accessibility regression
- Submission to moodle.org plugin directory (a11y is reviewed)
- Responding to a user-reported a11y bug

## Top rules

1. **Use semantic HTML**: `<button>` for actions, `<a>` for navigation, `<h1>`–`<h6>` in order
2. **Every interactive element keyboard-reachable** with visible `:focus`
3. **Every image** has `alt`; decorative images: `alt=""`
4. **Color contrast** ≥ 4.5:1 (text), 3:1 (UI components)
5. **Forms**: `<label for>` linked to every input
6. **Don't use color alone** to convey meaning
7. **Modals trap focus**, restore on close

## Semantic Mustache

```mustache
{{! BAD }}
<div class="btn" onclick="...">Save</div>

{{! GOOD }}
<button type="submit" class="btn btn-primary">{{#str}}save, core{{/str}}</button>
```

Headings:

```mustache
<h2 id="ann-heading">{{#str}}announcements, local_example{{/str}}</h2>
<section aria-labelledby="ann-heading">...</section>
```

Skip levels = WCAG fail. `<h2>` then `<h4>` is wrong.

## Forms — formslib

`MoodleQuickForm` outputs `<label for>` automatically. Don't bypass:

```php
$mform->addElement('text', 'name', get_string('name', 'local_example'));
$mform->setType('name', PARAM_TEXT);
$mform->addRule('name', null, 'required', null, 'client');

// Help button — adds aria-described relationship
$mform->addHelpButton('name', 'name', 'local_example');
```

For required fields, `addRule('required')` adds `aria-required="true"`.

## Buttons / links

| Element | Use for |
|---------|---------|
| `<button type="submit">` | Form submit |
| `<button type="button">` | JS action |
| `<a href="...">` | Navigation to a URL |

Never `<a href="#" onclick>` — use `<button>`. Never `<div role="button">` unless you also handle keyboard (Enter + Space) and focus.

## Icons

```mustache
{{#pix}}t/edit, core, {{#str}}edit, core{{/str}}{{/pix}}
```

Pix renderer outputs `<img alt="...">` with the title as alt. For decorative icons accompanying visible text:

```mustache
<button>
  {{#pix}}t/edit, core{{/pix}}<span class="visually-hidden"></span>
  {{#str}}edit, core{{/str}}
</button>
```

Bootstrap 5: `.visually-hidden` (was `.sr-only` in BS4).

## Color contrast

Boost defines `$primary`, `$secondary`, etc. Don't override to low-contrast values.

```scss
// theme/yourtheme/scss/post.scss
$primary: #0f6fc5;     // contrast vs white = 5.13:1 ✓
$danger:  #d9534f;     // contrast vs white = 3.34:1 ✗ — fails AA for text
```

Test: https://webaim.org/resources/contrastchecker/

Don't rely on color only:

```mustache
{{! BAD — color only }}
<span class="text-danger">{{name}}</span>

{{! GOOD — icon + color }}
<span class="text-danger">
  {{#pix}}t/error, core, {{#str}}invalid, local_example{{/str}}{{/pix}}
  {{name}}
</span>
```

## Tables

```mustache
<table class="table">
  <caption>{{#str}}attendance, local_example{{/str}}</caption>
  <thead>
    <tr>
      <th scope="col">{{#str}}user, core{{/str}}</th>
      <th scope="col">{{#str}}status, core{{/str}}</th>
    </tr>
  </thead>
  <tbody>
    {{#rows}}
    <tr>
      <th scope="row">{{name}}</th>
      <td>{{status}}</td>
    </tr>
    {{/rows}}
  </tbody>
</table>
```

`html_table` from `lib/outputcomponents.php` renders accessibly:

```php
$table = new \html_table();
$table->head = [get_string('name'), get_string('status')];
$table->headspan = [1, 1];
$table->caption = get_string('attendance', 'local_example');
echo \html_writer::table($table);
```

## ARIA — minimal use

Native HTML > ARIA. Only add ARIA when no native equivalent.

```mustache
{{! tab pattern — needs ARIA }}
<div role="tablist" aria-label="{{#str}}sections, local_example{{/str}}">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">One</button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2">Two</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">...</div>
```

Bootstrap 5 tab JS handles arrow keys. Don't reimplement.

## Modals

Use `core/modal` — handles focus trap, ESC key, focus restore on close.

```javascript
import Modal from 'core/modal';

const modal = await Modal.create({
    title: await getString('confirm', 'core'),
    body: await Templates.render('local_example/confirm', {}),
    show: true,
    removeOnClose: true,
});
// focus auto-traps inside; on close, focus returns to trigger element
```

Custom focus management:

```javascript
import {trapFocus} from 'core/local/aria/focusmanager';
const release = trapFocus(modalEl);
// ... when closing:
release();
triggerEl.focus();
```

## Live regions (toasts, async updates)

```javascript
import {add as addToast} from 'core/toast';
addToast('Saved', {type: 'success'});
```

`core/toast` uses `aria-live="polite"`. For urgent alerts: `aria-live="assertive"` (sparingly — interrupts screen readers).

## Keyboard

Every interactive control:
- Focusable in source order (don't `tabindex="0"` everything)
- Activated with Enter / Space
- Visible focus ring (don't `outline: 0` without replacement)
- Custom widgets follow [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/patterns/)

## Screen reader testing

| OS | SR | Browser |
|----|----|---------|
| Windows | NVDA (free) | Firefox |
| macOS | VoiceOver | Safari |
| Linux | Orca | Firefox |
| iOS | VoiceOver | Safari |
| Android | TalkBack | Chrome |

Quick checks:
- Tab through entire UI — every interactive element reachable
- Use SR-only — does the experience make sense?
- Resize text 200% — layout still works?
- Disable CSS — content order still meaningful?

## Automation

```bash
# axe-core via Puppeteer
npm install -g @axe-core/cli
axe http://localhost:8000/local/example/

# Pa11y
npm install -g pa11y
pa11y --standard WCAG2AA http://localhost:8000/local/example/
```

CI integration: run on PRs against staging.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `<div onclick>` | `<button>` |
| Missing `alt` on `<img>` | Add — `alt=""` if decorative |
| `placeholder` as label | Add real `<label for>` |
| `outline: 0` no replacement | Restore visible focus indicator |
| Heading level skip | Sequential `<h1>`/`<h2>`/`<h3>` |
| Link text "click here" / "more" | Descriptive (`Edit user "Alice"`) |
| `role="button"` without keyboard | Use `<button>` instead |
| Color-only error indication | Add icon + text |
| Color contrast < 4.5:1 | Adjust SCSS palette |
| Modal without focus trap | Use `core/modal` |
| `aria-label` on a `<div>` with no role | Either add role or use text instead |
| `aria-hidden="true"` on focusable element | Either unhide or remove from tab order |

## Plugin directory review

moodle.org reviewers run a11y checks. Common rejection reasons:
- New custom modal without focus trap
- Hard-coded colors failing AA contrast
- Custom widgets without ARIA / keyboard
- Missing `<label>` on form fields

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- Modal title is now `<h2 class="modal-title fs-5">`; start modal body headings at `<h3>` ([MDL-75699](https://tracker.moodle.org/browse/MDL-75699)).
- Experimental dark mode: no literal colours, `bg-white` or `text-dark`; use `--bs-*`/`--mds-*` tokens or `bg-body*`/`text-body*` utilities, and icons via the pix API so they inherit `currentColor` ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- `core/notification_base` accepts `headinglevel` (1-6) ([MDL-88458](https://tracker.moodle.org/browse/MDL-88458)); override `{{$searchrole}}{{/searchrole}}` in `core/search_input_auto` to drop a nested search landmark ([MDL-88833](https://tracker.moodle.org/browse/MDL-88833)).
- Course-index subsection ARIA (`aria-owns`/`aria-expanded`) moved to the delegating activity's treeitem; overrides of `courseindex/cm` and `courseindex/section` must change together ([MDL-88949](https://tracker.moodle.org/browse/MDL-88949)).
- Behat: `I set the focus on the "<element>" "<selector>"` (@javascript only) ([MDL-84065](https://tracker.moodle.org/browse/MDL-84065)); `--colourmode=dark` runs a suite in dark mode ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- `core/imagedetails/modal` `getImageDetails(file)` collects alt text / decorative flag for your own image-upload UI ([MDL-89214](https://tracker.moodle.org/browse/MDL-89214)); `flexible_table::set_columnheadersattributes()` (also 5.2.3+) ([MDL-89384](https://tracker.moodle.org/browse/MDL-89384)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Accessibility: https://moodledev.io/general/development/policies/accessibility
- WCAG 2.1: https://www.w3.org/TR/WCAG21/
- WAI-ARIA APG: https://www.w3.org/WAI/ARIA/apg/
- Boost a11y: https://docs.moodle.org/dev/Boost_-_Accessibility
- pix renderer: https://moodledev.io/docs/apis/subsystems/output#pix-icons
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-amd-javascript

> Use when writing JavaScript for Moodle — AMD modules, ES6 source, grunt build, RequireJS loading, core/ajax + core/str + core/templates + core/modal + core/notification, Mustache template hydration, and AMD unit tests.

# Moodle AMD JavaScript

## Overview

Moodle uses AMD (RequireJS) for browser JS. Source goes in `<plugin>/amd/src/*.js` (ES6+), built into minified AMD in `amd/build/*.min.js` via `grunt`. Never edit `amd/build/` directly. Modules import Moodle core helpers via string IDs (`core/ajax`, `core/str`, `core/templates`).

## When to Use

- Adding interactive JS to a plugin
- Calling a web service from the browser (`core/ajax`)
- Rendering a Mustache template client-side
- Showing a modal, notification, or toast
- Building a custom AMD module + grunt config
- Debugging "module not found" / "build out of date" warnings

**Skip when:** writing pure backend PHP — use `moodle-plugin-development`.

## File layout

```
<plugin>/
  amd/
    src/
      foo.js              # ES6 source — edit this
      bar.js
    build/                # generated; commit but never hand-edit
      foo.min.js
      foo.min.js.map
```

Module ID at runtime = `<component>/<filename>` (no extension):
- `local/example/amd/src/marker.js` → `local_example/marker`
- `mod/quiz/amd/src/preflight.js` → `mod_quiz/preflight`

## Module skeleton

```javascript
// amd/src/marker.js
import Ajax from 'core/ajax';
import Notification from 'core/notification';
import {get_string as getString} from 'core/str';
import Templates from 'core/templates';

const SELECTORS = {
    BUTTON: '[data-action="mark-present"]',
    ROW: '[data-region="attendance-row"]',
};

/**
 * Initialise: bind click handlers.
 *
 * @param {number} sessionid
 */
export const init = (sessionid) => {
    document.addEventListener('click', async (e) => {
        const btn = e.target.closest(SELECTORS.BUTTON);
        if (!btn) {
            return;
        }
        e.preventDefault();
        const userid = parseInt(btn.dataset.userid, 10);
        try {
            const result = await markPresent(sessionid, userid);
            const html = await Templates.render('local_example/row', result);
            const row = btn.closest(SELECTORS.ROW);
            Templates.replaceNode(row, html, '');
        } catch (err) {
            Notification.exception(err);
        }
    });
};

const markPresent = (sessionid, userid) => {
    const request = Ajax.call([{
        methodname: 'local_example_mark_present',
        args: {sessionid, userid},
    }]);
    return request[0];
};
```

## Loading from PHP

```php
$PAGE->requires->js_call_amd('local_example/marker', 'init', [$sessionid]);
```

The third argument array is JSON-encoded and passed to your `init` function.

## grunt build

Moodle ships a root `Gruntfile.js`. From Moodle root:

```bash
nvm use                # uses .nvmrc — match Moodle's required Node
npm ci
npx grunt amd          # build all AMD
npx grunt amd --root=local/example   # only your plugin
npx grunt watch        # rebuild on change
```

Build also runs ESLint + minification. `amd/build/*.min.js` and `*.min.js.map` are commit-required (Moodle plugin policy).

## Common core modules

| Module | Use |
|--------|-----|
| `core/ajax` | Call web services |
| `core/str` | Load lang strings (`getString('foo', 'local_example')`) |
| `core/templates` | Render Mustache (`render`, `renderForPromise`, `replaceNode`, `appendNodeContents`) |
| `core/notification` | Toasts, alerts, exception handling (`Notification.exception(err)`) |
| `core/modal` (Moodle 4.3+) | Replaces `core/modal_factory` |
| `core/modal_factory` | (deprecated 4.3+) — use `core/modal` |
| `core/toast` | Bootstrap-style ephemeral toasts (`add(msg, {type})`) |
| `core/pending` | Mark async work pending — required for Behat to wait |
| `core/event` | Custom DOM events for cross-module pub/sub |
| `core/log` | Console logging (silenced in production) |
| `core/fragment` | Server-rendered HTML fragments via `mod_x_output_fragment_*` |
| `core/url` | Build Moodle URLs |
| `core/config` | Access `M.cfg` (wwwroot, sesskey) safely |

## Mustache template + JS pair

```php
// PHP
$data = ['items' => $items, 'sesskey' => sesskey()];
echo $OUTPUT->render_from_template('local_example/list', $data);
$PAGE->requires->js_call_amd('local_example/list', 'init');
```

```mustache
{{! templates/list.mustache }}
<ul data-region="item-list">
  {{#items}}
  <li data-id="{{id}}">{{name}}</li>
  {{/items}}
</ul>
```

```javascript
// amd/src/list.js
import {get_strings as getStrings} from 'core/str';

export const init = async () => {
    const [confirmTitle, confirmBody] = await getStrings([
        {key: 'confirm', component: 'core'},
        {key: 'confirmdelete', component: 'local_example'},
    ]);
    // ...
};
```

## Pending — make Behat wait

Whenever JS does async work, mark it pending so Behat's `I wait until the page is ready` actually waits:

```javascript
import Pending from 'core/pending';

const pending = new Pending('local_example/marker:save');
try {
    await markPresent(...);
} finally {
    pending.resolve();
}
```

Without `Pending`, Behat scenarios are flaky.

## Modal (Moodle 4.3+)

```javascript
import Modal from 'core/modal';
import ModalEvents from 'core/modal_events';

const modal = await Modal.create({
    title: 'Confirm',
    body: Templates.render('local_example/confirm', data),
    show: true,
    removeOnClose: true,
});
modal.getRoot().on(ModalEvents.save, () => { /* handler */ });
```

## ESLint / coding style

Moodle ships `.eslintrc` in root. Run:

```bash
npx grunt eslint --root=local/example
```

Rules:
- ES2018+ source, transpile via grunt
- Prefer `const`/`let`, never `var`
- Arrow functions for callbacks
- JSDoc on every exported function (`@param`, `@return`)
- 4-space indent
- Single quotes for strings
- Semicolons mandatory
- No `console.log` (use `core/log`)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Editing `amd/build/*.min.js` | Edit `amd/src/*.js`, run `npx grunt amd` |
| Forgetting to commit `amd/build/` | Plugin policy — commit both src + build |
| Wrong module ID (`local_example/amd/marker`) | Use `local_example/marker` (no `amd/`) |
| `require(['jquery'], ...)` syntax | Use ES6 imports — grunt transpiles to AMD |
| No `Pending` around async | Behat scenarios flake — wrap with `core/pending` |
| Hard-coded strings | Load via `core/str` so they translate |
| `core/modal_factory` in 4.3+ | Switch to `core/modal` |
| Not passing args to `js_call_amd` | Args become `init(arg1, arg2)` parameters |
| Building with wrong Node version | `nvm use` from Moodle root respects `.nvmrc` |
| `M.cfg.wwwroot` directly | Import `core/config` for safe access |

## Calling web services

```javascript
import Ajax from 'core/ajax';

const requests = Ajax.call([
    {methodname: 'local_example_get_items', args: {courseid: 5}},
    {methodname: 'local_example_get_users', args: {courseid: 5}},
]);
const [items, users] = await Promise.all(requests);
```

Methods must be declared `'ajax' => true` in `db/services.php`. The web service token is auto-attached from session (no manual sesskey for AJAX).

## Testing AMD

Moodle's AMD has limited unit-test infrastructure. Options:
- **Manual** — load page, interact, watch console (`Notification.exception` surfaces errors)
- **Behat with `@javascript`** — full browser testing
- **Jest** (Moodle 4.4+) — `npx grunt jest` runs `tests/jest/*.test.js` if present

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- Do not import `theme_boost/bootstrap/*`: use `import {Tooltip} from 'bootstrap'` (5.3+) or `'theme_boost/index'` for multi-version plugins, then rebuild; exception: `util`/`dom` helpers still load directly (`bootstrap/dom/event-handler` on 5.3+, `theme_boost/bootstrap/dom/event-handler` on ≤5.2) ([MDL-88766](https://tracker.moodle.org/browse/MDL-88766)).
- Modal title is `<h2 class="modal-title fs-5">` (selectors on `h5.modal-title` break) ([MDL-75699](https://tracker.moodle.org/browse/MDL-75699)); nav items are React `.mds-nav-pill`, not `.nav-link`/`.moremenu` ([MDL-87830](https://tracker.moodle.org/browse/MDL-87830), [MDL-89294](https://tracker.moodle.org/browse/MDL-89294)).
- React mounting: `core/import` (native dynamic `import()` for `@moodle/lms/...` specifiers) and `core/component` `appendToDom()`/`prependToDom()` ([MDL-88505](https://tracker.moodle.org/browse/MDL-88505)); PHP side `\core\output\html_writer::react_component($module, $props)`, do not pre-`json_encode` props ([MDL-89296](https://tracker.moodle.org/browse/MDL-89296)).
- block_timeline is now ESM/React; its AMD modules and templates are gone ([MDL-88287](https://tracker.moodle.org/browse/MDL-88287)).
- `core/imagedetails/modal` `getImageDetails(file)` resolves `{alt, presentation, width, height}` or `null` ([MDL-89214](https://tracker.moodle.org/browse/MDL-89214)).
- TinyMCE picks its skin from `data-bs-theme` at setup; plugin dialogs/content CSS should use theme colour tokens ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- JavaScript modules: https://moodledev.io/docs/apis/subsystems/javascript-modules
- AMD: https://moodledev.io/docs/apis/subsystems/javascript-modules#amd-modules
- Templates JS: https://moodledev.io/docs/apis/subsystems/output/templates#using-templates-from-javascript
- core/ajax: https://moodledev.io/docs/apis/subsystems/external/writing-a-service#calling-from-javascript
- Modal: https://moodledev.io/docs/apis/subsystems/output/modal
- Coding style: https://moodledev.io/general/development/policies/codingstyle/javascript
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-behat-testing

> Use when writing or running Behat acceptance tests for Moodle plugins. Covers feature files, custom step definitions, tags, data generators in Background, JavaScript scenarios, and Selenium/Chromedriver setup.

# Moodle Behat Testing

## Overview

Behat drives a real browser against a dedicated Moodle test site. Features live in `<plugin>/tests/behat/*.feature`. Custom steps go in `tests/behat/behat_<component>.php` extending `behat_base`. Moodle provides hundreds of built-in steps (login, course creation, navigation).

## When to Use

- Writing end-to-end / acceptance tests
- Testing JavaScript-driven UI (drag-drop, modals, AJAX)
- Reproducing bugs that need full request cycle + cookies + session
- Adding scenarios to a plugin's regression suite

**Skip when:** writing unit tests for business logic — use `moodle-phpunit-testing`.

## First-time setup

```bash
# config.php additions:
$CFG->behat_wwwroot   = 'http://localhost:8000';
$CFG->behat_dataroot  = '/var/moodledata_behat';
$CFG->behat_prefix    = 'beh_';

# Initialize test site:
php admin/tool/behat/cli/init.php

# Install + start a Selenium-compatible driver (one of):
docker run -d -p 4444:4444 selenium/standalone-chrome:latest
# or chromedriver, geckodriver

# Run:
vendor/bin/behat --config /var/moodledata_behat/behat/behat.yml \
  --tags @local_example
```

## Feature file skeleton

```gherkin
@local_example @javascript
Feature: Mark attendance
  In order to track presence
  As a teacher
  I need to mark students present

  Background:
    Given the following "courses" exist:
      | fullname | shortname |
      | Maths    | M101      |
    And the following "users" exist:
      | username | firstname | lastname |
      | teacher1 | Tina      | Teach    |
      | student1 | Sam       | Student  |
    And the following "course enrolments" exist:
      | user     | course | role           |
      | teacher1 | M101   | editingteacher |
      | student1 | M101   | student        |

  Scenario: Teacher marks a student present
    Given I log in as "teacher1"
    And I am on "Maths" course homepage
    When I follow "Attendance"
    And I click on "Mark present" "button" in the "Sam Student" "table_row"
    Then I should see "1 present" in the "Today" "fieldset"
```

Tags:
- `@<component>` — required for `--tags` filtering
- `@javascript` — uses real browser; without it, runs headless (Goutte) — fast but no JS
- `@_file_upload`, `@_switch_window`, `@_alert` — capability tags so runner can skip on incompatible drivers

## Background data generators

Moodle exposes generators as Behat steps via `behat_data_generators`:

```gherkin
Given the following "local_example > items" exist:
  | course | name        | userid    |
  | M101   | First item  | student1  |
```

To enable, declare in `tests/generator/lib.php` AND register in `tests/behat/behat_local_example.php`:

```php
public function get_creatable_entities(): array {
    return [
        'items' => [
            'singular' => 'item',
            'datagenerator' => 'item',
            'required' => ['name'],
            'switchids' => ['course' => 'courseid', 'user' => 'userid'],
        ],
    ];
}
```

Generator method:

```php
// in local_example_generator
public function create_item(array $record): \stdClass {
    // same as PHPUnit generator
}
```

## Custom step definitions

`tests/behat/behat_local_example.php`:

```php
<?php
require_once(__DIR__ . '/../../../../lib/behat/behat_base.php');

use Behat\Mink\Exception\ExpectationException;

class behat_local_example extends behat_base {

    /**
     * @Given /^there are (\d+) attendance items$/
     */
    public function there_are_n_items(int $count): void {
        $gen = \testing_util::get_data_generator()
            ->get_plugin_generator('local_example');
        for ($i = 0; $i < $count; $i++) {
            $gen->create_item();
        }
    }

    /**
     * @Then /^the attendance count should be "(\d+)"$/
     */
    public function attendance_count_should_be(int $expected): void {
        global $DB;
        $actual = $DB->count_records('local_example_items');
        if ($actual !== $expected) {
            throw new ExpectationException(
                "Expected $expected, got $actual",
                $this->getSession()
            );
        }
    }
}
```

Class name **must** match `behat_<component>`. After adding/changing steps:

```bash
php admin/tool/behat/cli/init.php    # re-scan
```

## Running

```bash
# All Behat for plugin
vendor/bin/behat --config $CFG->behat_dataroot/behat/behat.yml --tags @local_example

# Single feature
vendor/bin/behat --config ... tests/behat/mark_attendance.feature

# Single scenario (line number)
vendor/bin/behat --config ... tests/behat/mark_attendance.feature:23

# Parallel (4 runners)
php admin/tool/behat/cli/init.php --parallel=4
vendor/bin/moodle_behat_parallel_run --tags @local_example
```

## Selectors (Mink)

| Type | Example |
|------|---------|
| `link` | `I follow "Settings"` |
| `button` | `I press "Save changes"` |
| `field` | `I set the field "Name" to "X"` |
| `select` | `I set the field "Role" to "Manager"` |
| `checkbox` | `I check "Visible"` |
| `table_row` | `... in the "Alice" "table_row"` |
| `fieldset` | `... in the "General" "fieldset"` |
| `dialogue` | `... in the "Confirm" "dialogue"` |
| `css_element` | `I click on ".foo .bar" "css_element"` |
| `xpath_element` | `I click on "//button[@data-x='y']" "xpath_element"` |

## Useful built-in steps

```gherkin
And I am on the "Course 1" "course" page logged in as "teacher1"
And I navigate to "Users > Enrolment methods" in current course administration
And I should see "X" in the "block_settings" "block"
And I wait until the page is ready
And I wait "2" seconds                # avoid; prefer wait-until
And I run the scheduled task "\local_example\task\cleanup"
And I run all adhoc tasks
And the following config values are set as admin:
  | config        | value | plugin        |
  | enabled       | 1     | local_example |
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Adding step but not running `behat/cli/init.php` | Re-init after every PHP step change |
| Missing `@javascript` for AJAX UI | Tag scenario `@javascript` and run with browser driver |
| Hard-coded sleeps `I wait 5 seconds` | Use `I wait until ... exists/disappears` |
| Class name mismatch | Must be `behat_<component>` |
| Not switching to dialogue context | Add `in the "..." "dialogue"` selector |
| Forgetting capability tags (`@_file_upload`) | Add so runner can skip on incompatible browsers |
| Editing `behat.yml` directly | Regenerated by `init.php` — config in `config.php` instead |

## Debugging

```bash
# Save HTML+screenshot on failure
$CFG->behat_faildump_path = '/tmp/behat-fails';

# Run a single scenario in foreground browser
vendor/bin/behat --config ... --tags @mytag --stop-on-failure -v

# Pause for inspection
And I should see "this will fail"     # forces a wait you can attach to
```

## CI snippet

```yaml
- name: Behat
  run: |
    php admin/tool/behat/cli/init.php
    vendor/bin/behat --config $MOODLE_DATA/behat/behat.yml \
      --tags @local_example --format=progress
```

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- Selectors: modal title `h5.modal-title` → `h2.modal-title` ([MDL-75699](https://tracker.moodle.org/browse/MDL-75699)); nav items are `.mds-nav-pill` (selected `.mds-nav-pill--selected`), not `.nav-link.active` ([MDL-87830](https://tracker.moodle.org/browse/MDL-87830), [MDL-89294](https://tracker.moodle.org/browse/MDL-89294)); navbar search is an inline field; the `togglesearch` button is only shown on small screens, so click it only if visible (see `behat_search.php`) ([MDL-87834](https://tracker.moodle.org/browse/MDL-87834), [MDL-89010](https://tracker.moodle.org/browse/MDL-89010)).
- Exporter/web-service strings use numeric entities (`&#38;` not `&amp;`), also 5.2.2+ ([MDL-79755](https://tracker.moodle.org/browse/MDL-79755)).
- New step `I set the focus on the "<element>" "<selector>"` (@javascript only) ([MDL-84065](https://tracker.moodle.org/browse/MDL-84065)).
- `--colourmode=dark` on `admin/tool/behat/cli/init.php`/`util.php`; scenarios assuming colour modes are off add `Given the run is not using a colour mode` ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- Steps `the course linear navigation should (not) be visible` ([MDL-87575](https://tracker.moodle.org/browse/MDL-87575)); linear nav is on by default for opted-in formats ([MDL-89406](https://tracker.moodle.org/browse/MDL-89406)).
- `I set portfolio instance "X" to "Y"` deprecated → `I set the portfolio instance "X" to "Y"` ([MDL-89069](https://tracker.moodle.org/browse/MDL-89069)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Behat in Moodle: https://moodledev.io/general/development/tools/behat
- Writing tests: https://moodledev.io/general/development/tools/behat/writing
- Step reference: https://moodledev.io/general/development/tools/behat/writing#step-definitions
- Data generators: https://moodledev.io/docs/apis/subsystems/testing/generators#behat
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-ci-matrix

> Use when adding or updating a GitHub Actions workflow that runs moodle-plugin-ci for a Moodle plugin, or when choosing its Moodle branch x PHP x database matrix. Covers starting from the upstream gha.dist.yml template, checking each branch's PHP and database requirements in environment.xml and core's own CI, service image pins, include/exclude matrix pitfalls, non-blocking rows for Moodle main, and making the plugin pass the checks on its first run. Skip when the task is about running PHPUnit or Behat locally with no CI workflow involved.

# Moodle plugin CI matrix (moodle-plugin-ci on GitHub Actions)

## Overview

`moodlehq/moodle-plugin-ci` runs Moodle's plugin checks (phplint, phpcs, phpdoc, validate, savepoints, mustache, grunt, PHPUnit, Behat) against a real Moodle checkout. It is usually run from GitHub Actions. The workflow itself is short. Most breakage comes from the matrix: a Moodle branch paired with a PHP or database version that branch does not support. Matrices built by pairing the Moodle versions you want with PHP and DB versions that look current keep producing pairs like Moodle 5.0 + PHP 8.1 or Moodle 5.2 + PostgreSQL 13. **Verify every combination against a source. Do not assume any of them.**

## When to Use

- Adding `.github/workflows/ci.yml` (or `moodle-ci.yml`) to a plugin repo
- Adding or dropping a Moodle branch, PHP version or database in an existing matrix
- CI fails at "Initialize containers", at `moodle-plugin-ci install`, or with an environment check error such as "PHP version must be at least ..."
- Adding a non-blocking row for Moodle `main` (the next release, still in development)
- **Skip when:** you are only running PHPUnit/Behat locally (see `moodle-phpunit-testing` / `moodle-behat-testing`), or the repo uses a different CI system

## 1. Start from the upstream template

Do not write the workflow from memory, and do not copy an older sibling repo's workflow. Fetch the current template:

```
https://raw.githubusercontent.com/moodlehq/moodle-plugin-ci/main/gha.dist.yml
```

Keep its job structure and check steps. Change only the matrix, service images, plugin path and triggers. The template installs the tool with `composer create-project ... moodlehq/moodle-plugin-ci ci ^4`, which is moodle-plugin-ci 4.x. Check the moodle-plugin-ci docs if you pin a different major.

## 2. Decide the Moodle branches

- Test the branches the plugin claims to support: `$plugin->requires` and `$plugin->supported` in `version.php`, or the branches the user names. If the user asks for a narrower matrix, a wider `supported` range can stay in `version.php`, but say which branches CI covers.
- A plugin that depends on another plugin needs `moodle-plugin-ci add-plugin owner/repo` (optionally `--branch X`) **before** `install`.

## 3. Verify PHP and DB per branch

For **each** branch, get the requirements from the branch itself:

1. **Minimum PHP and database versions**: that branch's `environment.xml`. The file moved in Moodle 5.1 when the web root became `public/`:
   - 5.1 and later: `public/admin/environment.xml`
   - 5.0 and earlier: `admin/environment.xml`

   ```bash
   git clone --bare --filter=blob:none https://github.com/moodle/moodle.git moodle.git
   git --git-dir moodle.git show MOODLE_502_STABLE:public/admin/environment.xml \
     | awk '/<MOODLE version="5.2"/{s=1} s && /VENDOR name="(mariadb|mysql|postgres)"|PHP version=/{print} /<\/MOODLE>/{s=0}'
   ```

   **Gotcha:** the file holds one `<MOODLE version="X.Y">` block for every release, including newer ones used for upgrade checks. Read the block whose version matches the branch, not the last block in the file.
2. **Maximum PHP**: this is not in `environment.xml`. Read core's own `.github/workflows/push.yml` on the same branch. Its comments mark the "lowest PHP supported" (MySQL job) and "highest PHP supported" (PostgreSQL job). Cross-check with the moodledev.io PHP page.
3. If you cannot verify a combination, leave it out and say so. A smaller verified matrix is better than a broken one. Never infer one branch's requirements from the branches next to it.

### Reference values

These were read from `environment.xml` and core `push.yml` on each branch in September 2026. They go out of date, so recheck them against the sources above before relying on them.

| Branch | PHP (min – max) | MariaDB min | PostgreSQL min | MySQL min |
|---|---|---|---|---|
| `MOODLE_405_STABLE` (4.5) | 8.1 – 8.3 | 10.6.7 | 13 | 8.0 |
| `MOODLE_500_STABLE` (5.0) | 8.2 – 8.4 | 10.11.0 | 14 | 8.4 |
| `MOODLE_501_STABLE` (5.1) | 8.2 – 8.4 | 10.11.0 | 15 | 8.4 |
| `MOODLE_502_STABLE` (5.2) | 8.3 – 8.4 | 10.11.0 | 16 | 8.4 |
| `main` (5.3 **beta**, moving target) | 8.3 – 8.4 | 11.4.0 | 17 | 8.4 |

The 5.3 row comes from the `<MOODLE version="5.3">` block of `main` at the `5.3beta` release commit. A beta's requirements can still change before the stable release. Recheck them when `MOODLE_503_STABLE` is branched. After that, `main` becomes the next development version.

### Pairs that must never appear

- PHP 8.4 with 4.5 (4.5's highest supported PHP is 8.3)
- PHP 8.1 with 5.0 or later; PHP 8.2 with 5.2 or later
- PostgreSQL below 17, or MariaDB below 11.4, with 5.3/`main`

## 4. Build the matrix only from verified pairs

**One branch.** Use a plain cross product with the branch's lowest and highest PHP, and alternate the databases:

```yaml
    strategy:
      fail-fast: false
      matrix:
        include:
          - {moodle-branch: MOODLE_502_STABLE, php: '8.3', database: pgsql}
          - {moodle-branch: MOODLE_502_STABLE, php: '8.4', database: mariadb}
```

**Several branches with different PHP ranges.** Use an object axis, so each branch keeps its own PHP list and is still crossed with every database:

```yaml
      matrix:
        moodle:
          - {branch: MOODLE_405_STABLE, php: '8.1'}
          - {branch: MOODLE_405_STABLE, php: '8.3'}
          - {branch: MOODLE_502_STABLE, php: '8.3'}
          - {branch: MOODLE_502_STABLE, php: '8.4'}
        database: [pgsql, mariadb]
```

Then read `matrix.moodle.branch` and `matrix.moodle.php` in `setup-php` and in the `install` step's env. If you cross flat `php:` and `moodle-branch:` axes instead, prune with `exclude:` and give each entry a comment:

```yaml
        exclude:
          - {moodle-branch: MOODLE_405_STABLE, php: '8.4'}  # 4.5 max PHP is 8.3 (core push.yml)
```

**`include:` trap.** GitHub first tries to merge an `include` entry into existing combinations. If it cannot merge without overwriting an original value, it adds the entry as a **new job with only the keys it lists**. For example, `{moodle-branch: X, php: Y}` added to a matrix that also has `database` becomes a job with an empty `DB`, and `install` then fails. Expand the matrix before you push. Parse the YAML and apply GitHub's documented rules with a short script. With no original axes, every `include` entry is its own job.

## 5. Service images

- Pin each database to the **lowest** version that satisfies **every** branch in the matrix. An image that is too new hides breakage at the minimum version, and one that is too old fails the newest branch. For example, 4.5–5.2 together need `postgres:16` and `mariadb:10.11`.
- The upstream template's floating `postgres:17` / `mariadb:11` images are newer than the minimum for 5.2 and earlier. Pin them explicitly.
- **Health check:** MariaDB 11.x images do not ship `mysqladmin`, so `--health-cmd="mysqladmin ping"` never reports healthy and the job dies at "Initialize containers". Use the template's `healthcheck.sh --connect --innodb_initialized`, or `mariadb-admin ping`.
- To give different rows different images (for example a `main` row that needs PostgreSQL 17), make the image a matrix value with a default:

```yaml
      postgres:
        image: ${{ matrix.database == 'pgsql' && (matrix.pgsql-image || 'postgres:16') || '' }}
```

## 6. Non-blocking row for `main`

Add the next Moodle release as an `include:` row that cannot fail the build:

```yaml
jobs:
  test:
    continue-on-error: ${{ matrix.experimental == true }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - {moodle-branch: MOODLE_502_STABLE, php: '8.3', database: pgsql}
          - {moodle-branch: MOODLE_502_STABLE, php: '8.4', database: mariadb}
          - {moodle-branch: main, php: '8.4', database: pgsql, pgsql-image: 'postgres:17', experimental: true}
```

- Keep any "committed amd/build is stale" check off `main`. The bundles are built with the stable branch's toolchain, and `main`'s grunt output can differ.
- **Artifact names:** the template's `Behat Faildump (${{ join(matrix.*, ', ') }})` joins every matrix value, so an image override like `postgres:17` puts a `:` in the artifact name, which `actions/upload-artifact` rejects. List the fields yourself: `Behat Faildump (${{ matrix.moodle-branch }}, PHP ${{ matrix.php }}, ${{ matrix.database }})`.
- Before changing any matrix key, grep the workflow for every `matrix.` consumer.

## 7. Propose the matrix before writing it

Show the user a table (branch x PHP x DB) with one line on how each axis was checked. Then write the file. A workflow on the default branch runs on every push, so a wrong matrix is visible right away.

## 8. Make the plugin pass on the first run

CI that fails on its first run teaches everyone to ignore it. Before you finish, run each step locally the same way the workflow runs it:

```bash
composer create-project -n --no-dev --prefer-dist moodlehq/moodle-plugin-ci ci ^4
ci/bin/moodle-plugin-ci install --plugin ./local_myplugin --db-host=127.0.0.1 --db-type=pgsql --branch=MOODLE_502_STABLE
ci/bin/moodle-plugin-ci phplint
ci/bin/moodle-plugin-ci phpcs --max-warnings 0
ci/bin/moodle-plugin-ci phpdoc --max-warnings 0
ci/bin/moodle-plugin-ci validate
ci/bin/moodle-plugin-ci savepoints
ci/bin/moodle-plugin-ci mustache
ci/bin/moodle-plugin-ci grunt --max-lint-warnings 0
ci/bin/moodle-plugin-ci phpunit --fail-on-warning
```

- Every step must exit 0. `--max-warnings` is valid only on `phpcs` and `phpdoc`.
- **Run the pipeline as written.** Swapping in a flag that looks equivalent tests a different pipeline. For example, an explicit `install --extra-plugins <dir>` in place of the workflow's `add-plugin` step overrides the path that `add-plugin` already wrote to moodle-plugin-ci's `.env` (`EXTRA_PLUGINS_DIR`), and it can fail on the runner with `Failed to run realpath(...)`. Keep `add-plugin`, and don't add `--extra-plugins`.
- After a local `install`, check `MOODLE_DIR` in `ci/.env`. A failed install leaves it pointing at the previous tree, and later steps then silently test the wrong code.
- Two failures that keep coming back: every Mustache template in `templates/` needs an `Example context (json):` block in its docblock, and lang strings must be sorted by key.
- If the workflow checks `amd/build`, rebuild it with the Node version and grunt of the branch that step runs on.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Crossing all requested branches with "current" PHP | Unsupported pair, install fails the environment check | Check each branch's `environment.xml` and core `push.yml` |
| Reading the last `<MOODLE>` block in `environment.xml` | Picks up a newer release's minimums | Select `version="X.Y"` for the branch |
| Looking for `admin/environment.xml` on 5.1+ | File not found | 5.1+ uses `public/admin/environment.xml` |
| `include` entry missing the `database` key | Extra job with empty `DB` | Use an object axis, or list every key |
| `mysqladmin ping` health check on MariaDB 11 | Containers never become healthy | `healthcheck.sh --connect --innodb_initialized` |
| Using floating DB images (`postgres:17`, `mariadb:11`) for older branches | Hides breakage at the minimum version | Pin the lowest version every branch in the matrix supports |
| `join(matrix.*)` in the artifact name with image values in the matrix | upload-artifact rejects the name | Name the fields explicitly |
| `main` row without `continue-on-error` | Upstream churn breaks the build | `experimental: true` plus job-level `continue-on-error` |
| Replacing `add-plugin` with `--extra-plugins` | Install fails at `realpath` on the runner | Keep the upstream step order |

## References

- https://moodledev.io/general/development/tools/phpunit (PHPUnit in plugins)
- https://moodledev.io/general/releases (release dates and per-version requirements)
- https://moodledev.io/general/development/policies/php (PHP version support policy)
- https://moodlehq.github.io/moodle-plugin-ci/ (moodle-plugin-ci documentation)
- https://github.com/moodlehq/moodle-plugin-ci/blob/main/gha.dist.yml (upstream GitHub Actions template)
- https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow (matrix `include`/`exclude` rules)


---

### moodle-definition-of-done

> Use when about to report a Moodle plugin task complete, open a pull request, or hand work over for review. A definition-of-done checklist covering code style and phpdoc gates (moodle-plugin-ci, phpcs, moodlecheck), PHPUnit and Behat, course lifecycle handling (backup, restore, reset, uninstall), privacy provider coverage, upgrade path, lang strings, release-notes consistency, proving each gate is non-vacuous, and verifying the effect rather than the exit code. Skip when exploring or prototyping with no intent to ship yet.

# Moodle Definition of Done

## Overview

A checklist to walk before claiming any Moodle plugin change is finished. Every
item is reported as **pass**, **fail**, or **not applicable, because ...**.
Evidence comes before assertions: run the command, read its output, and state
what you saw. A gate you did not run is not a pass.

## When to Use

- Before saying "done" on a plugin feature, fix, or refactor
- Before opening a pull request or requesting a review
- Before handing a change to a release step (then also run `moodle-release-preflight`)
- **Skip when:** spiking or prototyping code that will not be merged

## Two rules that govern every item

1. **A gate that printed nothing has proven nothing.** Before trusting a
   "0 findings" result, feed the gate one known-bad input (a missing docblock, an
   unsorted lang key, a deliberately failing test) and confirm it complains. A
   grep pattern that can never match, a test suite that is empty, and a linter
   pointed at the wrong path all look exactly like "clean".
2. **Verify the effect, not the exit code.** A script that did not throw has not
   necessarily worked. Read the DB row, the rendered page, the file on disk, or
   the log line that proves the change happened.

## 1. Code style and phpdoc

Run the same binary CI runs; it has real exit codes:

```bash
for s in phplint phpmd "phpcs --max-warnings 0" "phpdoc --max-warnings 0" validate savepoints; do
    moodle-plugin-ci $s /path/to/plugin > "ci_${s%% *}.log" 2>&1
    echo "$s exit: $?"
done
```

- Run `mustache` and `grunt` against the plugin installed inside a Moodle tree
  (`-m <moodle>`), not the repo path; `mustache` exits 1 'not within basename'
  on a path outside it.
- `phpmd` exits 0 even when it reports violations, so read its log.
- The standalone `local_moodlecheck` CLI (`local/moodlecheck/cli/moodlecheck.php`)
  also exits 0 on errors, and its text output reads `Line 54: ...`. If you parse
  it, grep `Line [0-9]+:` and prove the pattern matches known-bad output once.
- Lint JS and CSS from inside a Moodle tree, at the plugin's real path: core's
  ESLint config scopes AMD rules by path, so a copy elsewhere gets other rules.
- Never trust `grunt ... --force`: it downgrades every later failure to a
  warning and still prints "Done, but with warnings."

**Docblock gotchas moodlecheck catches:**

- A `@param` type containing a space (`array<string, mixed>`, `array{a: int}`)
  is read as type plus name, so it reports an incomplete parameter list. Keep
  `@param` types plain and describe the shape in prose.
- Every function, test helpers included, needs a description sentence and
  complete `@param` / `@return` tags. Signature edits without a matching
  `@param` are the usual miss.
- `@return` is not name-matched (generics are fine there); never write a
  literal `@param` inside docblock prose.

## 2. Recurring review gates

- Every Mustache template has an `Example context (json):` block in its docblock.
- `$string[...]` keys in `lang/en/<component>.php` are sorted alphabetically.
  Assert it with a script over the whole file after any insertion rather than
  picking the insertion point by eye. phpcs checks sorting only with
  `--runtime-set moodleBranch <n>`.
- Every string key the code uses exists. Grep for static keys, and load the
  pages that build keys dynamically (`get_string('status_' . $state, ...)`)
  and check that no `[[` placeholder appears.
- `defined('MOODLE_INTERNAL') || die();` only in files with side effects
  (`lib.php`, `db/*.php`, `settings.php`, `version.php`); moodle-cs flags it in
  plain class files as `MoodleInternalNotNeeded`.
- No scaffolding placeholders left in file headers (`@copyright`, author).
- `version.php` `$plugin->requires` is not below the lowest Moodle branch the
  plugin claims to support.

## 3. Tests

- The plugin's PHPUnit tests pass; Behat features pass if the change touches UI
  or user-visible behavior.
- **Confirm plugin tests actually exist and ran.** `moodle-plugin-ci phpunit`
  over an empty `tests/` directory passes, so "CI is green" is not evidence of
  coverage. Check the run's test count.
- PHPUnit loads all of core, but a web request does not. Code calling a core
  free function (`create_course()`, `fulldelete()`) without the matching
  `require_once` can pass tests and fail in the browser. Hit that path for real.

## 4. Upgrade path

- A `db/install.xml` change is paired with a `db/upgrade.php` step and a savepoint
  (`upgrade_plugin_savepoint()`, or `upgrade_mod_savepoint()` for activities), and the `$plugin->version` build
  number is raised. A schema change must bump `$plugin->version`; the
  human-readable `$plugin->release` is a separate decision.
- Test both paths: a fresh install and an upgrade from the previous release.
  Both must produce the same schema, and the install log must contain no
  `XMLDB has detected` or `Debugging:` lines.
- `moodle-plugin-ci savepoints` passes.

## 5. Course lifecycle

If the plugin stores course- or activity-linked data, check each of these, or
state why it does not apply:

- **Backup and restore:** `backup/moodle2/backup_<mod>_activity_task.class.php`,
  `backup_<mod>_stepslib.php` and the restore counterparts cover every table
  holding course data. A restored course behaves like the original.
- **Restore cleans its input.** A crafted `.mbz` is untrusted input: apply the
  same `clean_param()` / allowlist logic the forms apply.
- **Course reset needs all three callbacks** for an activity module:

```php
function myplugin_reset_course_form_definition(&$mform) {
    $mform->addElement('header', 'mypluginheader', get_string('modulenameplural', 'mod_myplugin'));
    $mform->addElement('advcheckbox', 'reset_myplugin_attempts', get_string('removeattempts', 'mod_myplugin'));
}

function myplugin_reset_course_form_defaults($course) {
    return ['reset_myplugin_attempts' => 1];
}

function myplugin_reset_userdata($data) {
    // Delete user data when $data->reset_myplugin_attempts is set, reset
    // gradebook entries, and return a status array for the reset report.
}
```

  (Core looks these up as `<modname>_reset_...`, without the `mod_` prefix, e.g.
  `forum_reset_userdata()` in `mod/forum/lib.php`.) A lone `reset_userdata`
  leaves the option unreachable in the reset form and grades stale.
- **Uninstall:** add `db/uninstall.php` with `xmldb_<component>_uninstall()` if
  the plugin writes outside its own tables (core grade items, files, config in
  other components) that would otherwise be orphaned.
- **Privacy:** every table and external location declared in `get_metadata()` is
  handled by export and by all three delete paths, and a `provider_test`
  exercises contexts, users, export and delete on generated data. A declared
  table that is never serviced is a compliance defect; a stale table name after
  a rename throws. See `moodle-privacy-gdpr`.

## 6. Security self-review of changed sinks

For every changed entry point (form handler, external function, AJAX call,
`pluginfile` callback):

- `require_login()` / `require_capability()` in the right context, and the
  capability re-checked at asynchronous sinks (scheduled/adhoc tasks, deferred
  grade pushes), not only at request time.
- Ids and values from the client are re-validated server-side, never trusted
  from hidden fields.
- Any client-side limit (size, duration, count) is also enforced server-side.
- Output is escaped at the sink (`s()`, `format_string()`, `format_text()`),
  especially anything that reaches `innerHTML`.
- User-uploaded files are served with `$forcedownload = true` unless their
  MIME type is validated.

Before a release, run `moodle-release-preflight` for the full list of defect
classes reviews have caught, and `moodle-security-audit` for the broad checklist.

## 7. Verified in a running site

Green tests are not the finish line. Observe the changed behavior on a real
Moodle site after installing the plugin and purging caches:

- Read the stored record, the rendered notice, or the screenshot's actual
  content, not just the fact that the command returned.
- Watch values set before a bulk write (restore, import, `update_record()`)
  that can silently overwrite them.
- Check the web server log for PHP warnings and `Debugging:` output from the
  pages you touched, with developer debugging enabled.

## 8. Release notes and docs

- `README.md` still describes current behavior and settings.
- The changelog records user-visible changes, in the format the plugin already
  uses.
- Any user manual or help page that names a changed, renamed, or removed
  setting, field or page is updated. A removal is the dangerous case: the
  doc becomes actively wrong rather than incomplete.
- `$plugin->release` is changed only as part of a deliberate release, and
  `composer.json` (if present) is still valid.

## 9. Clean working state

- Every background process you started (Behat, a PHP built-in server,
  chromedriver, watchers) is stopped, and the port is free. Stop processes by
  exact PID after checking their command line, not by broad `pkill -f` patterns.
- The diff contains only intended files: no build output, logs, `node_modules/`,
  or hand edits to `amd/build/` or `thirdparty/` code.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Reporting "clean" from a gate that printed nothing | Real errors ship; CI fails later | Prove the gate fires on a known-bad input |
| Parsing moodlecheck output with the wrong pattern | "0 findings" forever | Match `Line [0-9]+:` and test the grep |
| `grunt --force` treated as success | Stylelint or ESLint errors hidden as warnings | Run `moodle-plugin-ci grunt` and check its exit code |
| Empty `tests/` with a green CI badge | No coverage, false confidence | Check the reported test count |
| Only `*_reset_userdata` implemented | Reset option never shown; grades stale | Implement all three reset callbacks |
| Restore trusts `.mbz` values | Stored XSS or broken data via crafted backup | Clean restored fields like form input |
| `install.xml` changed without an upgrade step | Upgraded sites differ from fresh installs | Add an `upgrade.php` step plus savepoint |
| Privacy table declared but never exported or deleted | Compliance defect | Cover every table in export and delete, with a test |
| "The script didn't throw" treated as success | Setting not saved, record not deleted | Read back the effect |

## References

- https://moodledev.io/general/development/tools/phpcs
- https://moodledev.io/general/development/policies/codingstyle
- https://moodledev.io/general/development/tools/behat
- https://moodledev.io/general/development/tools/phpunit
- https://moodledev.io/docs/guides/upgrade
- https://moodledev.io/docs/apis/subsystems/backup
- https://moodledev.io/docs/apis/subsystems/privacy
- https://moodledev.io/docs/apis/plugintypes/mod
- https://moodledev.io/general/community/plugincontribution/checklist
- https://moodlehq.github.io/moodle-plugin-ci/


---

### moodle-field-lessons

> Use when writing, reviewing or testing Moodle plugin code, to avoid non-obvious pitfalls learned from shipping real plugins: version.php supported ranges and cross-branch API guards, services.php/tasks.php registration, authorisation and state-collision bugs, output escaping (html_writer, format_string, imported content), DB/XMLDB API small print, backup/restore/reset and DST-safe date shifting, privacy provider drift, lang string placeholders, forms and admin settings, PHPUnit/Behat blind spots, AMD/modal/TinyMCE gotchas, navigation and block placement, and CI false greens. Skip when you need a full checklist for one topic (use the sibling skill named in each section).

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


---

### moodle-hooks-api

> Use when implementing or migrating to the Moodle 4.4+ Hooks API. Covers hook class authoring (db/hooks.php), callback registration, dispatching, replacing legacy magic callbacks (extend_navigation, before_http_headers, etc.), and testing hook listeners.

# Moodle Hooks API

## Overview

Moodle 4.4 introduced a typed Hooks API (`core\hook\manager`) replacing the unmaintainable jungle of magic callback functions like `<plugin>_extend_navigation`, `<plugin>_before_http_headers`, `<plugin>_extend_settings_navigation`, etc. Hooks are real classes with typed payloads, dispatched through `\core\di::get(\core\hook\manager::class)`. Plugins register interest via `db/hooks.php`.

## When to Use

- Adding cross-cutting behavior triggered by core (navigation, page output, user login, course events) without monkey-patching
- Migrating a plugin off legacy magic callbacks (deprecated 4.4+, will be removed)
- Authoring a hook *class* in core or in a plugin that other plugins can listen to
- Writing tests for hook listeners

**Skip when:** the event you care about is a `\core\event\*` (Events 2 API — different system, used for audit/logging). Hooks are for *modifying* behavior; Events are for *reacting to* facts.

## Core concepts

| Concept | Where it lives | Purpose |
|---|---|---|
| Hook class | `classes/hook/<name>.php` | Typed payload, optional setters for listeners to mutate |
| Listener registration | `db/hooks.php` | Maps hook class -> callback (`Class::method`) + priority |
| Dispatcher call | Core or plugin code | `\core\di::get(\core\hook\manager::class)->dispatch(new \plugin\hook\thing(...));` |
| Listener method | Any class | Static or instance method taking the hook instance |

## Listening to a core hook

`db/hooks.php`:

```php
<?php
defined('MOODLE_INTERNAL') || die();

$callbacks = [
    [
        'hook'     => \core\hook\output\before_standard_top_of_body_html_generation::class,
        'callback' => \local_example\hook_listener::class . '::inject_banner',
        'priority' => 100, // higher runs first
    ],
];
```

`classes/hook_listener.php`:

```php
<?php
namespace local_example;

use core\hook\output\before_standard_top_of_body_html_generation as hook;

class hook_listener {
    public static function inject_banner(hook $hook): void {
        global $USER;
        if (isguestuser() || !isloggedin()) {
            return;
        }
        $hook->add_html('<div class="alert alert-info">Hello, ' . s($USER->firstname) . '</div>');
    }
}
```

After adding or changing `db/hooks.php`, purge caches: `php admin/cli/purge_caches.php`.

## Authoring your own hook

`classes/hook/before_widget_render.php`:

```php
<?php
namespace local_example\hook;

use core\hook\described_hook_interface;
use core\hook\stoppable_event_interface;

class before_widget_render implements described_hook_interface, stoppable_event_interface {
    private bool $stopped = false;
    private string $html = '';

    public function __construct(public readonly int $widgetid, public readonly \context $context) {}

    public static function get_hook_description(): string {
        return 'Dispatched before a widget renders. Listeners may append HTML or veto rendering.';
    }

    public static function get_hook_tags(): array {
        return ['output', 'widget'];
    }

    public function add_html(string $html): void { $this->html .= $html; }
    public function get_html(): string { return $this->html; }

    public function stop(): void { $this->stopped = true; }
    public function isPropagationStopped(): bool { return $this->stopped; }
}
```

Dispatch:

```php
$hook = new \local_example\hook\before_widget_render($widgetid, $context);
\core\di::get(\core\hook\manager::class)->dispatch($hook);
if ($hook->isPropagationStopped()) {
    return ''; // veto
}
echo $hook->get_html();
```

## Migrating magic callbacks

| Legacy callback | Replacement hook |
|---|---|
| `<plugin>_extend_navigation` | `\core\hook\navigation\primary_extend` (4.5+) — check core for current name |
| `<plugin>_before_http_headers` | `\core\hook\output\before_http_headers` |
| `<plugin>_before_standard_top_of_body_html` | `\core\hook\output\before_standard_top_of_body_html_generation` |
| `<plugin>_before_footer` | `\core\hook\output\before_footer_html_generation` |
| `<plugin>_after_config` | `\core\hook\after_config` |
| `<plugin>_extend_settings_navigation` | check `\core\hook\navigation\*` for current name |

Migration steps:

1. Search for legacy callbacks: `grep -rn "function.*_extend_navigation\|_before_http_headers\|_after_config" .`
2. For each, find the matching hook class in `lib/classes/hook/` of your Moodle install.
3. Create `db/hooks.php` mapping; move the callback body into a listener class.
4. Delete the legacy function from `lib.php`.
5. Bump `version.php`, purge caches, run tests.

## Testing hook listeners

```php
<?php
namespace local_example;

defined('MOODLE_INTERNAL') || die();

final class hook_listener_test extends \advanced_testcase {
    /** @covers \local_example\hook_listener::inject_banner */
    public function test_banner_injected_for_logged_in_user(): void {
        $this->resetAfterTest();
        $this->setUser($this->getDataGenerator()->create_user());

        $hook = new \core\hook\output\before_standard_top_of_body_html_generation();
        \core\di::get(\core\hook\manager::class)->dispatch($hook);

        $this->assertStringContainsString('Hello,', $hook->get_output());
    }

    public function test_skipped_for_guest(): void {
        $this->resetAfterTest();
        $this->setGuestUser();

        $hook = new \core\hook\output\before_standard_top_of_body_html_generation();
        \core\di::get(\core\hook\manager::class)->dispatch($hook);

        $this->assertStringNotContainsString('Hello,', $hook->get_output());
    }
}
```

## CLI: list registered hooks

```bash
php admin/cli/hooks_list.php          # all hooks + listeners
php admin/cli/hooks_list.php --hook=core\\hook\\output\\before_http_headers
```

## Gotchas

- **Purge caches** after any `db/hooks.php` change. Listener registration is cached.
- **Priority** is a *hint* — order between equal priorities is undefined. Don't rely on it for correctness.
- **Stoppable hooks**: only implement `stoppable_event_interface` if vetoing is meaningful. Most output hooks are not stoppable.
- **Don't dispatch hooks from constructors or `setUp()`** — they can have side effects.
- **DI container**: always resolve `manager` via `\core\di::get(...)`. Don't `new` it.
- **Backporting**: pre-4.4 plugins still need legacy callbacks. Either keep both with a version check, or drop pre-4.4 support and bump `requires` in `version.php`.

## Checklist

- [ ] `db/hooks.php` exists with correct `hook` class FQCN
- [ ] Listener class is autoloadable (under `classes/` with PSR-4)
- [ ] Caches purged after edit
- [ ] Tests cover both the action and the no-op branch
- [ ] Legacy callback removed (or version-gated) after migration
- [ ] `version.php` bumped

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- New hook `\core\hook\email\before_email_to_user`, dispatched by `email_to_user()`: edit `$hook->email` fields, call `$hook->email->add_additional_header()`, or veto sending with `$hook->email->add_block_reason()` ([MDL-69724](https://tracker.moodle.org/browse/MDL-69724)).
- `\core_user\hook\extend_user_menu`: `add_navitem()`/`get_navitems()` deprecated → `add_menu_item()` with `\core_user\output\user_action_menu\{link,divider,header,text}` ([MDL-88938](https://tracker.moodle.org/browse/MDL-88938)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## See also

- `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-mobile-app

> Use when integrating a Moodle plugin with the Moodle Mobile app — db/mobile.php remote templates, Ionic/Angular components delivered server-side, mobile.php view types, addons, push notifications, and offline support.

# Moodle Mobile App Integration

## Overview

Moodle Mobile app (Ionic/Angular) loads plugin UI as **remote templates** declared in `db/mobile.php`. The server returns Mustache-like templates + JS that the app renders inline. No app rebuild required — works on any user's installed app once the site declares the addon.

## When to Use

- Adding plugin UI to mobile app
- Surfacing notifications, dashboard items, or activity views in the app
- Offline-capable content
- Push notifications for plugin events

**Skip when:** plugin only used in browser admin UI.

## Architecture

```
Moodle server                          Moodle Mobile app
─────────────                          ─────────────────
db/mobile.php          ────────────►   Discovers addons via
classes/output/mobile.php              tool_mobile_get_plugins_supporting_mobile

Returns                ────────────►   Renders Ionic components
  templates (.html)
  initial JS
  styles
```

App calls a server function which returns:
- A Mustache-like template (Ionic + custom directives)
- Optional JS to run client-side
- Optional initial data
- Optional offline functions

## db/mobile.php

```php
<?php
defined('MOODLE_INTERNAL') || die();

$addons = [
    'local_example' => [
        'handlers' => [
            'mainmenu' => [
                'displaydata' => [
                    'title' => 'pluginname',
                    'icon'  => 'list',
                    'class' => '',
                ],
                'delegate'    => 'CoreMainMenuDelegate',
                'method'      => 'mobile_main_menu_view',
                'styles'      => [
                    'url'     => '/local/example/mobile/styles.css',
                    'version' => 1,
                ],
                'offlinefunctions' => [
                    'mobile_main_menu_view' => [],
                ],
            ],
            'courseoption' => [
                'displaydata' => [
                    'title' => 'attendance',
                    'class' => '',
                ],
                'delegate' => 'CoreCourseOptionsDelegate',
                'method'   => 'mobile_course_view',
            ],
        ],
        'lang' => [
            ['pluginname', 'local_example'],
            ['attendance', 'local_example'],
        ],
    ],
];
```

### Delegates

| Delegate | Where it shows |
|----------|----------------|
| `CoreMainMenuDelegate` | Main menu (bottom tab bar) |
| `CoreUserDelegate` | User profile page |
| `CoreCourseOptionsDelegate` | Course menu |
| `CoreCourseModuleDelegate` | Activity in course (for `mod_*` plugins) |
| `CoreBlockDelegate` | Sidebar block (for `block_*`) |
| `CoreSettingsDelegate` | App settings page |
| `CoreMessageOutputDelegate` | Message output handler |

## classes/output/mobile.php

```php
<?php
namespace local_example\output;
defined('MOODLE_INTERNAL') || die();

class mobile {

    public static function mobile_main_menu_view(array $args): array {
        global $DB, $USER;

        $items = $DB->get_records('local_example_items',
            ['userid' => $USER->id], 'timecreated DESC', '*', 0, 20);

        return [
            'templates' => [
                [
                    'id'   => 'main',
                    'html' => self::render_main_template(),
                ],
            ],
            'javascript' => self::get_javascript(),
            'otherdata'  => [
                'items' => json_encode(array_values($items)),
                'sesskey' => sesskey(),
            ],
            'files' => [],
        ];
    }

    private static function render_main_template(): string {
        return '
<ion-list>
    <ion-item-divider><ion-label>{{ \'plugin.local_example.attendance\' | translate }}</ion-label></ion-item-divider>
    <ion-item *ngFor="let item of CONTENT_OTHERDATA.items">
        <ion-label>
            <h2>{{ item.name }}</h2>
            <p>{{ item.timecreated | coreFormatDate }}</p>
        </ion-label>
        <ion-button slot="end" (click)="markPresent(item.id)">
            {{ \'plugin.local_example.markpresent\' | translate }}
        </ion-button>
    </ion-item>
</ion-list>';
    }

    private static function get_javascript(): string {
        return "
this.markPresent = (id) => {
    const params = {sessionid: id, userid: this.CoreSitesProvider.getCurrentSite().getUserId()};
    return this.CoreSitesProvider.getCurrentSite()
        .write('local_example_mark_present', params)
        .then((result) => {
            this.CoreDomUtilsProvider.showToast('plugin.local_example.marked', true, 2000);
        });
};";
    }
}
```

## Required web services

The function name in `db/mobile.php` (`'method' => 'mobile_main_menu_view'`) must be exposed as a web service in `db/services.php`:

```php
$functions = [
    'local_example_mobile_main_menu_view' => [
        'classname'    => 'local_example\output\mobile',
        'methodname'   => 'mobile_main_menu_view',
        'description'  => 'Main menu mobile view',
        'type'         => 'read',
        'capabilities' => '',
        'services'     => [MOODLE_OFFICIAL_MOBILE_SERVICE],
    ],
];
```

Plus the AJAX/REST functions used inside the template (`local_example_mark_present`).

## Template syntax

Mobile templates use Angular + Ionic with Moodle directives:

| Directive | Use |
|-----------|-----|
| `{{ 'plugin.local_example.foo' \| translate }}` | Lang string |
| `{{ value \| coreFormatDate }}` | Format unix timestamp |
| `{{ html \| coreFormatText }}` | Format Moodle text |
| `<core-format-text [text]="html" />` | Same, component form |
| `*ngFor="let item of CONTENT_OTHERDATA.items"` | Loop |
| `*ngIf="condition"` | Conditional |
| `(click)="handler()"` | Event |
| `<ion-item>`, `<ion-list>`, `<ion-button>` | Ionic UI |
| `<core-empty-box>` | "Nothing here" placeholder |

`CONTENT_OTHERDATA` = data passed via `'otherdata'`. Strings JSON-decoded automatically.

## JavaScript scope

Helpers injected into JS scope:

| Object | Use |
|--------|-----|
| `this.CoreSitesProvider` | Current site, current user, web service calls |
| `this.CoreDomUtilsProvider` | Toasts, alerts, modals |
| `this.CoreFilepoolProvider` | Cache files for offline |
| `this.CoreUtilsProvider` | Misc helpers |
| `this.refreshContent(false)` | Reload current view |

## Offline support

Add functions to `'offlinefunctions'`:

```php
'offlinefunctions' => [
    'mobile_main_menu_view' => [],
    'local_example_get_items' => [],
],
```

App pre-fetches results during sync — they survive offline. For mutations (mark present), use the offline write API:

```javascript
this.CoreSitesProvider.getCurrentSite().write(
    'local_example_mark_present',
    params,
    {forceOffline: false, getFromCache: false}
).catch((error) => {
    if (this.CoreUtilsProvider.isWebServiceError(error)) {
        return Promise.reject(error);
    }
    // Queue for later sync
    return this.CoreCourseProvider.storeOfflineAction(...);
});
```

## Push notifications

Push delivery via Moodle's airnotifier service. Plugin events trigger notifications via `\core\message\manager::send_message()`. Mobile app receives them automatically when:

- Site has airnotifier configured (Site admin > Mobile > Push notifications)
- User logged into mobile app
- User has the addon's notification preference enabled

## Testing

1. Local dev: install Moodle Mobile app, point at `http://10.0.2.2:8000` (Android emulator) or your LAN IP
2. Site admin > Mobile > Enable web services for mobile devices
3. Log in to app — addon appears
4. To force template re-fetch after server change: pull-to-refresh, or app settings > Synchronisation > Synchronise now
5. Browser-based dev: `npx ionic serve` against [`moodle-mobile-app`](https://github.com/moodlehq/moodleapp) source pointing at your site

## Styles

`local/example/mobile/styles.css` (referenced from `db/mobile.php`):

```css
.local_example-marked {
    color: var(--ion-color-success);
    font-weight: bold;
}
```

Bumping `'version' => N` in `db/mobile.php` invalidates app's cached styles.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Method not exposed as web service | Declare in `db/services.php` with `MOODLE_OFFICIAL_MOBILE_SERVICE` |
| Lang strings not declared in `db/mobile.php` | App can't render `{{ 'plugin...' | translate }}` |
| `<div>` instead of Ionic `<ion-*>` | Use Ionic components for native look |
| Forgetting `MOODLE_OFFICIAL_MOBILE_SERVICE` | Web service callable but not for mobile |
| Heavy DB queries on every refresh | Cache via MUC + invalidate on update |
| Hand-formatting timestamps | Use `coreFormatDate` filter |
| Embedding HTML directly | Use `coreFormatText` for `format_text`-equivalent escaping |
| No `offlinefunctions` for read views | Pre-fetch fails — declare them |
| Not bumping styles `version` | App keeps old CSS |
| Calling `fetch()` directly | Use `CoreSitesProvider` — handles auth + tokens |

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- **Breaking:** `login/token.php` rejects credentials in the query string (POST only), checks the service before authenticating, and drops `appsitecheck` ([MDL-87010](https://tracker.moodle.org/browse/MDL-87010)).
- Deep-link auto-login (token/privatetoken) needs `tool_mobile/enabledeeplinkautologin`, default off ([MDL-88924](https://tracker.moodle.org/browse/MDL-88924)).
- Course-module web services return the standard cm fields (`lang`, `section`, `visible`, `groupmode`, `groupingid`); build `get_*_by_courses` returns from `helper_for_get_mods_by_courses::standard_coursemodule_elements_returns()` ([MDL-87241](https://tracker.moodle.org/browse/MDL-87241)).
- New `mod_forum_set_read_state` ([MDL-87887](https://tracker.moodle.org/browse/MDL-87887)); `gradereport_user_get_grade_items` adds optional `parentcategoryid` ([MDL-64304](https://tracker.moodle.org/browse/MDL-64304)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Mobile addons: https://moodledev.io/general/app/development/plugins-development-guide
- Templates spec: https://moodledev.io/general/app/development/plugins-development-guide/templates
- Delegates list: https://moodledev.io/general/app/development/plugins-development-guide/api-reference
- Offline support: https://moodledev.io/general/app/development/plugins-development-guide/offline
- Push notifications: https://moodledev.io/general/app/development/plugins-development-guide/notifications
- App source: https://github.com/moodlehq/moodleapp
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-performance

> Use when optimizing Moodle plugin performance — MUC caching definitions, query optimization, recordsets vs records, ad-hoc and scheduled tasks, lazy loading, OPcache, query log analysis, $PERF, and performance debug toolbar.

# Moodle Performance

## Overview

Moodle bottlenecks: too many DB queries per request, full-table scans, loading large recordsets into memory, blocking work on the request path. Mitigations: MUC caching, indexes, recordsets, ad-hoc tasks, careful capability checks.

## When to Use

- Page renders slowly / hits per-page query limits
- N+1 query pattern in loops
- Memory-blow on large datasets
- Long-running operations on POST handlers
- Tuning a custom report or scheduled task

## Diagnosis

### Performance debug

`config.php`:

```php
$CFG->debug = (E_ALL | E_STRICT);
$CFG->debugdisplay = 1;
$CFG->perfdebug = 15;     // shows query count + timings in footer
```

Site admin > Development > Debugging > Performance info: enables the footer toolbar (queries, time, memory, MUC hits/misses, sessions reads/writes).

### `$PERF`

```php
global $PERF;
$start = microtime(true);
// ... work ...
$PERF->dbqueries++;       // your manual counter
mtrace('Took ' . round(microtime(true) - $start, 3) . 's');
```

### Slow query log

Postgres / MySQL slow-query log — turn on in dev. Moodle prefixes queries with the source class via `$CFG->dboptions['debug']`.

## MUC — Moodle Universal Cache

3 cache modes:

| Mode | Storage | Lifetime | Use |
|------|---------|----------|-----|
| `MODE_APPLICATION` | Persistent (file/Redis/Memcached) | Until invalidated | Shared computed values |
| `MODE_SESSION` | Session | Session lifetime | Per-user computed values |
| `MODE_REQUEST` | PHP request | Single request | Memoize within request |

### Define a cache

`db/caches.php`:

```php
<?php
defined('MOODLE_INTERNAL') || die();

$definitions = [
    'active_announcements' => [
        'mode'         => cache_store::MODE_APPLICATION,
        'simplekeys'   => true,
        'simpledata'   => true,
        'ttl'          => 600,
        'invalidationevents' => ['changesin_local_announcements'],
    ],
    'user_dashboard' => [
        'mode'       => cache_store::MODE_SESSION,
        'simplekeys' => true,
    ],
];
```

Bump `version.php` after edits.

### Use the cache

```php
$cache = \cache::make('local_announcements', 'active_announcements');
$value = $cache->get('all');
if ($value === false) {
    $value = $this->compute_announcements();
    $cache->set('all', $value);
}
return $value;
```

### Invalidate

```php
\cache_helper::invalidate_by_event('changesin_local_announcements', ['all']);
// or
$cache->delete('all');
$cache->purge();
```

Trigger invalidation from a settings save or an observer.

### `simplekeys` / `simpledata`

- `simplekeys: true` — keys are alphanum (skips hashing) — faster
- `simpledata: true` — values are scalars/arrays of scalars (skips serialization) — much faster

Use both whenever possible.

### Static cache (per-request)

```php
$cache = \cache::make('local_announcements', 'request_lookup');
// MODE_REQUEST in caches.php — auto-cleared at end of request
```

## DB optimization

### Recordsets for large data

```php
// BAD — loads everything into memory
$rows = $DB->get_records('huge_table');
foreach ($rows as $r) { /* ... */ }

// GOOD — streams
$rs = $DB->get_recordset('huge_table');
foreach ($rs as $r) { /* ... */ }
$rs->close();      // ALWAYS close
```

Use `get_recordset_sql` with `LIMIT` + offset for batched processing of millions of rows.

### Avoid N+1

```php
// BAD
foreach ($courses as $c) {
    $teacher = $DB->get_record('user', ['id' => $c->teacherid]);
}

// GOOD — single query
$tids = array_column($courses, 'teacherid');
[$insql, $params] = $DB->get_in_or_equal($tids);
$teachers = $DB->get_records_sql("SELECT * FROM {user} WHERE id $insql", $params);
```

### Indexes

```xml
<INDEX NAME="userid-courseid" UNIQUE="false" FIELDS="userid, courseid"/>
```

Edit via XMLDB editor. Composite index column order matters — most-selective first, leftmost-prefix usable for partial queries.

### `EXPLAIN` your queries

```bash
mysql> EXPLAIN SELECT ... ;
postgres=# EXPLAIN ANALYZE SELECT ... ;
```

Look for:
- `type: ALL` (full scan) — needs index
- `Using filesort` / `Using temporary` — sort spilling
- High `rows` estimate — selectivity issue

## Move work off the request

### Ad-hoc task — fire-and-forget

```php
// classes/task/send_report.php
namespace local_example\task;
class send_report extends \core\task\adhoc_task {
    public function execute(): void {
        $data = $this->get_custom_data();
        // do work
    }
}

// trigger:
$task = new \local_example\task\send_report();
$task->set_custom_data(['userid' => $user->id, 'reportid' => $r->id]);
$task->set_userid($user->id);     // runs as that user
\core\task\manager::queue_adhoc_task($task);
```

Tasks run via cron. Long jobs don't block HTTP request.

### Scheduled task — recurring

`db/tasks.php`:

```php
$tasks = [[
    'classname' => 'local_example\task\cleanup',
    'blocking'  => 0,
    'minute'    => '0',
    'hour'      => '3',
    'day'       => '*',
    'month'     => '*',
    'dayofweek' => '*',
]];
```

```php
class cleanup extends \core\task\scheduled_task {
    public function get_name(): string {
        return get_string('task:cleanup', 'local_example');
    }
    public function execute(): void {
        // ...
    }
}
```

Run cron: `php admin/cli/cron.php`. Production: cron entry every minute.

### Locks for non-overlapping execution

```php
$locktype = 'local_example_cleanup';
$lockfactory = \core\lock\lock_config::get_lock_factory($locktype);
if (!$lock = $lockfactory->get_lock('main', 5)) {
    return;     // another instance running
}
try {
    // ...
} finally {
    $lock->release();
}
```

## Capability check optimization

`has_capability()` is expensive at scale. For lists:

```php
// BAD — N capability checks
foreach ($users as $u) {
    if (has_capability('mod/quiz:attempt', $context, $u)) { /* ... */ }
}

// GOOD — single batched query
$users = get_users_by_capability($context, 'mod/quiz:attempt', 'u.id, u.firstname, u.lastname');
```

## Output / page

```php
$PAGE->set_pagelayout('embedded');     // skips heavy regions when appropriate
$PAGE->add_body_class('skip-some-blocks');
```

Lazy-load AMD modules:

```javascript
// only load when needed
const onClick = async () => {
    const {init} = await import('local_example/heavy');
    init();
};
```

## OPcache + APCu

`php.ini`:

```ini
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0    # production only
opcache.revalidate_freq = 0
```

Moodle benefits massively from OPcache. With `validate_timestamps=0`, restart PHP after deploys.

APCu for Moodle config cache: `$CFG->localcachedir = '/var/cache/moodle';`

## Sessions

- Use Redis sessions in production (`$CFG->session_handler_class = '\core\session\redis'`)
- File sessions don't scale past one app server
- Session writes — see "session lock" issues if multiple AJAX requests block on session

```php
// release session lock early when only reading
\core\session\manager::write_close();
```

## Frontend

- Run `npx grunt amd` — production builds are minified
- Enable browser caching: `$CFG->cachejs = true; $CFG->cachetemplates = true;`
- Theme designer mode (`$CFG->themedesignermode`) — OFF in production (disables CSS cache)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `get_records` on huge table | `get_recordset` + close |
| N+1 in loops | Batch with `get_in_or_equal` |
| Sending email on POST | Move to ad-hoc task |
| `has_capability` per row | `get_users_by_capability` |
| MUC without `simplekeys`/`simpledata` | Set both true when possible |
| MUC `MODE_REQUEST` for cross-request data | Use `MODE_APPLICATION` |
| Long lock without try/finally | Release in `finally` block always |
| Theme designer mode on prod | Off — kills CSS caching |
| `validate_timestamps = 1` | OFF in prod, restart on deploy |
| Forgetting `$rs->close()` | Recordset leaks — always close |
| File sessions on multi-app-server | Switch to Redis/Memcached |

## Profiling

`xhprof` / `tideways`:

```php
// quick on-demand profile
$CFG->profilingenabled = true;
$CFG->profilingautostart = false;
// add ?PROFILEME to URL — view in admin/tool/profiling
```

Per-request: `Site admin > Development > Profiling`.

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- `queue_adhoc_task($task, true)` now returns the existing task id for a duplicate (was `false`); don't test truthiness for "newly queued" ([MDL-86422](https://tracker.moodle.org/browse/MDL-86422)).
- `adhoc_task::set_soft_retry_delay()` reschedules without counting a failure (also 5.2.2+) ([MDL-79763](https://tracker.moodle.org/browse/MDL-79763)); block uninstall deletes instances in an ad-hoc task ([MDL-89289](https://tracker.moodle.org/browse/MDL-89289)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- MUC: https://moodledev.io/docs/apis/subsystems/muc
- Tasks API: https://moodledev.io/docs/apis/core/task
- Performance recommendations: https://docs.moodle.org/en/Performance_recommendations
- DB API recordsets: https://moodledev.io/docs/apis/core/dml#get_recordset
- Profiling: https://moodledev.io/general/development/tools/profiling
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-phpunit-testing

> Use when writing, running, or debugging PHPUnit tests for Moodle plugins or core. Covers advanced_testcase, resetAfterTest, data generators, mocking $DB, testing events/tasks/external functions, and CLI invocation.

# Moodle PHPUnit Testing

## Overview

Moodle ships its own PHPUnit harness with test bootstrap, transactional resets, and data generators. Tests live in `<plugin>/tests/<thing>_test.php` and extend `advanced_testcase`. Never call `parent::setUp()` for DB cleanup — use `$this->resetAfterTest()`.

## When to Use

- Writing unit/integration tests for any Moodle plugin or core API
- Debugging test failures (`Database was modified` errors, isolation issues)
- Adding a data generator (`tests/generator/lib.php`)
- Testing events, scheduled tasks, ad-hoc tasks, external functions
- Setting up CI for Moodle test suites

**Skip when:** writing Behat acceptance tests (use `moodle-behat-testing`).

## First-time setup

```bash
php admin/tool/phpunit/cli/init.php       # writes phpunit.xml + initializes test DB
vendor/bin/phpunit --testsuite local_example_testsuite
```

`phpunit.xml` is regenerated by `init.php` — never hand-edit. Re-run after installing a new plugin.

## Test class skeleton

```php
<?php
namespace local_example;
defined('MOODLE_INTERNAL') || die();

/**
 * @group local_example
 * @covers \local_example\manager
 */
final class manager_test extends \advanced_testcase {

    public function test_create_item(): void {
        $this->resetAfterTest();
        $generator = self::getDataGenerator();
        $course = $generator->create_course();
        $user = $generator->create_user();

        $manager = new manager();
        $id = $manager->create_item($course->id, $user->id, 'hello');

        global $DB;
        $row = $DB->get_record('local_example_items', ['id' => $id], '*', MUST_EXIST);
        $this->assertSame('hello', $row->name);
    }
}
```

Key rules:
- File name: `<thing>_test.php`, class: `<thing>_test`
- `final class` (Moodle policy since 4.2)
- `@covers` annotation required by Moodle CS
- `@group <component>` enables `--group` filtering
- `void` return type on test methods, `: void` on `setUp`
- `self::` (not `$this->`) for static methods like `getDataGenerator()`

## Data generators

Plugin generator at `tests/generator/lib.php`:

```php
<?php
defined('MOODLE_INTERNAL') || die();

class local_example_generator extends component_generator_base {
    public function create_item(array $record = []): \stdClass {
        global $DB, $USER;
        $defaults = [
            'courseid'   => 0,
            'userid'     => $USER->id,
            'name'       => 'Item ' . random_string(8),
            'timecreated'=> time(),
        ];
        $record = (object)array_merge($defaults, $record);
        $record->id = $DB->insert_record('local_example_items', $record);
        return $record;
    }
}
```

Use:

```php
$gen = self::getDataGenerator()->get_plugin_generator('local_example');
$item = $gen->create_item(['name' => 'test']);
```

Activity module generator extends `testing_module_generator` and implements `create_instance()`.

## Common patterns

### Test an event

```php
$sink = $this->redirectEvents();
$manager->do_thing();
$events = $sink->get_events();
$sink->close();
$this->assertCount(1, $events);
$this->assertInstanceOf(\local_example\event\thing_done::class, $events[0]);
```

### Test an email

```php
$sink = $this->redirectEmails();
$manager->notify($user);
$messages = $sink->get_messages();
$this->assertSame($user->email, $messages[0]->to);
```

### Test a scheduled task

```php
$task = new \local_example\task\cleanup();
$task->execute();
// assert side effects
```

### Test an external (web service) function

```php
$this->setUser($user);
$result = \local_example\external\get_items::execute($courseid);
$result = \core_external\external_api::clean_returnvalue(
    \local_example\external\get_items::execute_returns(),
    $result
);
$this->assertCount(2, $result);
```

`clean_returnvalue` is mandatory — catches schema mismatches.

### Test an ad-hoc task

```php
\core\task\manager::queue_adhoc_task(new \local_example\task\send_report());
$this->runAdhocTasks(\local_example\task\send_report::class);
```

### Login as a user

```php
$user = $this->getDataGenerator()->create_user();
$this->setUser($user);             // sets $USER global
$this->setAdminUser();             // shortcut
$this->setGuestUser();
```

### Time-travel

```php
$this->mock_clock_with_frozen(1700000000);    // Moodle 4.4+
// or in older versions, manually set timecreated/timemodified
```

## Running tests

```bash
# Single suite
vendor/bin/phpunit --testsuite local_example_testsuite

# Single file
vendor/bin/phpunit local/example/tests/manager_test.php

# Single method
vendor/bin/phpunit --filter test_create_item local/example/tests/manager_test.php

# By group
vendor/bin/phpunit --group local_example

# Coverage (requires xdebug or pcov)
vendor/bin/phpunit --coverage-html coverage/ local/example/tests
```

## Test database

- Separate DB defined in `config.php`: `$CFG->phpunit_prefix = 'phpu_';`
- Reset between tests via transactions — `$this->resetAfterTest()` enables it
- Schema drift error: re-run `php admin/tool/phpunit/cli/init.php`
- "Database was modified" failure means a test mutated DB without `resetAfterTest()`

## Mocking

Moodle prefers integration tests with the real test DB over mocking `$DB`. When you must mock:

```php
$mockDB = $this->createMock(\moodle_database::class);
$mockDB->method('get_record')->willReturn((object)['id' => 1]);
// inject via DI, never replace global
```

Avoid replacing the global `$DB` — breaks isolation.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `$this->resetAfterTest()` | Add at start of every DB-touching test |
| Class not `final` | Add `final` (Moodle 4.2+ policy) |
| Missing `@covers` | Add `@covers \Fully\Qualified\Class` |
| Hand-editing `phpunit.xml` | Re-run `admin/tool/phpunit/cli/init.php` |
| Using `parent::setUp()` to reset DB | Use `resetAfterTest()` instead |
| Skipping `clean_returnvalue` on external fn | Always wrap external returns to catch schema bugs |
| `$this->getDataGenerator()` (instance) | Moodle prefers `self::getDataGenerator()` (static) |
| Asserting time with `time()` | Use `mock_clock_with_frozen` or compare with tolerance |

## CI snippet (GitHub Actions)

```yaml
- name: PHPUnit
  run: |
    php admin/tool/phpunit/cli/init.php
    vendor/bin/phpunit --testsuite ${{ matrix.suite }}
```

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- Exporter/web-service strings use numeric entities (`&#38;` not `&amp;`), also 5.2.2+: update assertions ([MDL-79755](https://tracker.moodle.org/browse/MDL-79755)).
- `admin/tool/phpunit/cli/util.php` gains `--snapshot[=NAME]`, `--restore=NAME` and `--upgrade`, so CI can restore a cached core install and upgrade in the plugin ([MDL-88495](https://tracker.moodle.org/browse/MDL-88495)).
- Password/auth functions delegate to DI classes `\core\authentication\password` and `\core\authentication`, mockable via `\core\di::set()` ([MDL-88580](https://tracker.moodle.org/browse/MDL-88580)); `#[\DI\Attribute\Inject]` properties are filled by `\core\di::get()`/`make()` ([MDL-89528](https://tracker.moodle.org/browse/MDL-89528)).
- `route_testcase` adds `assert_route_is_scoped()`, `assert_route_is_unscoped()`, `assert_route_required_scopes()` ([MDL-89089](https://tracker.moodle.org/browse/MDL-89089)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- PHPUnit in Moodle: https://moodledev.io/general/development/tools/phpunit
- Data generators: https://moodledev.io/docs/apis/subsystems/testing/generators
- Test writing guide: https://moodledev.io/general/development/policies/testing
- Coverage: https://moodledev.io/general/development/tools/phpunit#code-coverage
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-plugin-development

> Use when creating, modifying, upgrading, or reviewing Moodle plugins (local, mod, block, format, theme, auth, enrol, report, qtype, filter, repository) - covers version.php, db/install.xml + upgrade.php, db/access.php capabilities, db/services.php web services, lang strings, classes/ PSR-4 autoloading, lib.php hooks, settings.php, privacy provider, and Moodle coding standards.

# Moodle Plugin Development

## Overview

Moodle plugins follow strict frankenstyle naming and a fixed file layout. Every plugin lives under a type-specific directory (`local/`, `mod/`, `blocks/`, `course/format/`, `theme/`, `auth/`, `enrol/`, `report/`, `question/type/`, `filter/`, `repository/`, etc.) and is identified by `<type>_<name>` (e.g., `local_school`, `mod_quiz`, `format_schoolgram`).

**Core principle:** Moodle code must declare `defined('MOODLE_INTERNAL') || die();` at top of every PHP file (except classes under `classes/` autoloaded via PSR-4), use `$DB` for all DB access, use `get_string()` for user-facing text, and bump `version.php` for any DB or capability change.

## When to Use

- Creating new plugin under `local/`, `mod/`, `blocks/`, `course/format/`, `theme/`, etc.
- Adding/modifying DB tables → `db/install.xml` + `db/upgrade.php`
- Adding capabilities → `db/access.php`
- Adding web services / external functions → `db/services.php` + `classes/external/`
- Adding scheduled tasks → `db/tasks.php` + `classes/task/`
- Adding events / observers → `db/events.php` + `classes/event/`
- Adding admin settings page → `settings.php`
- Translating strings → `lang/<lang>/<component>.php`
- Reviewing PR for Moodle coding-style compliance
- Bumping `version.php` after schema/capability change

**Skip when:** working purely on frontend AMD modules without backend changes, or non-Moodle PHP code.

## Plugin Skeleton

Minimum files for a `local_<name>` plugin:

```
local/<name>/
  version.php                    # required: component, version, requires, maturity
  lang/en/local_<name>.php       # required: 'pluginname' string
  lib.php                        # optional: hooks (extend_navigation, pluginfile, etc.)
  settings.php                   # optional: admin settings page
  db/
    install.xml                  # DB schema (edit via /admin/tool/xmldb/)
    upgrade.php                  # versioned upgrade steps
    access.php                   # capabilities
    services.php                 # web service definitions
    tasks.php                    # scheduled tasks
    events.php                   # event observers
    caches.php                   # cache definitions
  classes/                       # PSR-4: \local_<name>\foo\bar -> classes/foo/bar.php
    external/                    # external (web service) functions
    task/                        # scheduled task classes
    event/                       # custom events
    privacy/provider.php         # GDPR provider (REQUIRED)
  templates/                     # Mustache templates
  amd/src/                       # ES modules (built via grunt -> amd/build/)
  tests/                         # PHPUnit + Behat
```

## version.php Template

```php
<?php
defined('MOODLE_INTERNAL') || die();

$plugin->component = 'local_example';      // frankenstyle, must match dir
$plugin->version   = 2026042500;           // YYYYMMDDXX, bump on any db/capability change
$plugin->requires  = 2024100700;           // min Moodle version (4.5 LTS); use 2025041400 for 5.0+, 2025100600 for 5.1+, 2026042000 for 5.2+; 5.3beta is 2026091600 (beta only, 5.3.0 value TBD)
$plugin->release   = '1.0.0';
$plugin->maturity  = MATURITY_STABLE;      // ALPHA | BETA | RC | STABLE
$plugin->dependencies = ['mod_quiz' => 2024100700];  // optional
```

## db/install.xml + upgrade.php

- Edit `install.xml` via Moodle XMLDB editor (`Site admin -> Development -> XMLDB editor`) — never hand-edit.
- Every schema change requires:
  1. Bump `$plugin->version` in `version.php`
  2. Add upgrade step in `db/upgrade.php` keyed on old version:

```php
function xmldb_local_example_upgrade($oldversion) {
    global $DB;
    $dbman = $DB->get_manager();

    if ($oldversion < 2026042500) {
        $table = new xmldb_table('local_example_items');
        $field = new xmldb_field('status', XMLDB_TYPE_INTEGER, '4', null,
            XMLDB_NOTNULL, null, '0', 'name');
        if (!$dbman->field_exists($table, $field)) {
            $dbman->add_field($table, $field);
        }
        upgrade_plugin_savepoint(true, 2026042500, 'local', 'example');
    }
    return true;
}
```

## db/access.php Capabilities

```php
$capabilities = [
    'local/example:view' => [
        'captype'      => 'read',
        'contextlevel' => CONTEXT_SYSTEM,
        'archetypes'   => [
            'user'           => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
        ],
    ],
];
```

Runtime check: `require_capability('local/example:view', $context);` or `has_capability(...)`.

## db/services.php (Web Services / AJAX)

```php
$functions = [
    'local_example_get_items' => [
        'classname'    => 'local_example\external\get_items',
        'methodname'   => 'execute',
        'description'  => 'Return items',
        'type'         => 'read',
        'ajax'         => true,
        'capabilities' => 'local/example:view',
    ],
];
```

External class extends `\core_external\external_api` (Moodle 4.2+; older = `external_api`) and defines `execute_parameters()`, `execute()`, `execute_returns()`.

## Lang Strings

`lang/en/local_example.php`:

```php
$string['pluginname']    = 'Example';
$string['example:view']  = 'View example';        // capability strings: <plugin>:<cap>
$string['greeting']      = 'Hello, {$a->name}';   // placeholders
```

Use: `get_string('greeting', 'local_example', ['name' => $user->firstname]);`

## DB Access — Always `$DB`

```php
global $DB;
$DB->get_record('local_example_items', ['id' => $id], '*', MUST_EXIST);
$DB->get_records_sql('SELECT * FROM {local_example_items} WHERE status = ?', [1]);
$DB->insert_record('local_example_items', $obj);
$DB->update_record('local_example_items', $obj);
$DB->delete_records('local_example_items', ['id' => $id]);
```

Never `mysqli_*` / `PDO`. Always use `{tablename}` placeholder (Moodle prefixes). Always bind params, never concatenate.

## Hooks via lib.php

Common Moodle callbacks (function name = `<component>_<hookname>`):

- `local_example_extend_navigation(global_navigation $nav)` — add nav nodes
- `local_example_before_http_headers()` — runs before headers
- `local_example_extend_settings_navigation($settingsnav, $context)`
- `local_example_pluginfile($course, $cm, $context, $filearea, $args, $forcedl, $options)` — serve file area

Moodle 4.4+: prefer the new hooks API (`\core\hook\manager`) over magic callbacks where available.

## Output — Renderers + Templates

- HTML via `$OUTPUT->render_from_template('local_example/foo', $data)` (Mustache: `templates/foo.mustache`)
- Custom renderer: `classes/output/renderer.php` extends `plugin_renderer_base`
- Never `echo` raw HTML in business logic; always go through renderer or template

## Privacy (GDPR) — Required

Every plugin must declare privacy. Minimum (no user data stored):

```php
// classes/privacy/provider.php
namespace local_example\privacy;
defined('MOODLE_INTERNAL') || die();
class provider implements \core_privacy\local\metadata\null_provider {
    public static function get_reason(): string {
        return 'privacy:metadata';
    }
}
```

If plugin stores user data, implement `\core_privacy\local\request\plugin\provider` and define `get_metadata`, `export_user_data`, `delete_data_for_user_in_context`, `delete_data_for_users`, `get_contexts_for_userid`, `get_users_in_context`.

## Quick Reference

| Task | File | Bump version? |
|------|------|---------------|
| Add DB table/field | `db/install.xml` + `db/upgrade.php` | Yes |
| Add capability | `db/access.php` + lang string | Yes |
| Add web service fn | `db/services.php` + `classes/external/` | Yes |
| Add scheduled task | `db/tasks.php` + `classes/task/` | Yes |
| Add event observer | `db/events.php` | Yes |
| Add cache definition | `db/caches.php` | Yes |
| Add string | `lang/en/<component>.php` | No |
| Add settings | `settings.php` | No |
| Add template | `templates/*.mustache` | No |
| Add renderer | `classes/output/renderer.php` | No |

## Coding Standards

- 4-space indent, no tabs
- Opening `<?php` on line 1, no closing `?>` at end of pure PHP files
- `defined('MOODLE_INTERNAL') || die();` immediately after license block (skip in `classes/` PSR-4 files)
- snake_case for functions/vars, PascalCase for classes
- Run `vendor/bin/phpcs --standard=moodle <path>` before commit (`local_codechecker` plugin or `moodle-cs` ruleset)
- All user-facing strings via `get_string()`, never hard-coded English
- GPL-3.0-or-later license header on every PHP file

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hand-editing `install.xml` | Use XMLDB editor at `/admin/tool/xmldb/` |
| Forgetting to bump `version.php` after schema change | Bump + add `upgrade.php` step + `upgrade_plugin_savepoint` |
| Using raw SQL with `{}` quoting wrong | Use `{tablename}` and `?` / named params, never string concat |
| `MOODLE_INTERNAL` check inside `classes/` PSR-4 file | Remove — autoloaded files don't need it |
| Storing user data without privacy provider | Implement `\core_privacy\local\request\plugin\provider` |
| Hard-coded English strings in PHP/Mustache | Move to `lang/en/<component>.php` + `get_string` / `{{#str}}` |
| Forgetting `require_capability()` on entry points | Always `require_login()` + capability check early |
| Not purging caches after `version.php` bump | `php admin/cli/upgrade.php` or Site admin -> Purge caches |
| Direct `$_GET`/`$_POST` access | Use `required_param()` / `optional_param()` with type |
| Missing `sesskey()` check on state-changing requests | Call `require_sesskey()` on POST handlers |

## Plugin Type Cheatsheet

| Type | Dir | Frankenstyle | Extra required files |
|------|-----|--------------|----------------------|
| Local | `local/<name>` | `local_<name>` | none beyond skeleton |
| Activity module | `mod/<name>` | `mod_<name>` | `mod_form.php`, `view.php`, `lib.php` w/ `<name>_add_instance` etc. |
| Block | `blocks/<name>` | `block_<name>` | `block_<name>.php` extends `block_base` |
| Course format | `course/format/<name>` | `format_<name>` | `format.php`, `lib.php` extends `core_courseformat\base` |
| Theme | `theme/<name>` | `theme_<name>` | `config.php`, `scss/`, `layout/` |
| Auth | `auth/<name>` | `auth_<name>` | `auth.php` extends `auth_plugin_base` |
| Enrol | `enrol/<name>` | `enrol_<name>` | `lib.php` extends `enrol_plugin` |
| Question type | `question/type/<name>` | `qtype_<name>` | `questiontype.php`, `question.php`, `renderer.php` |
| Filter | `filter/<name>` | `filter_<name>` | `filter.php` extends `moodle_text_filter` |
| Repository | `repository/<name>` | `repository_<name>` | `lib.php` extends `repository` |

## Security Checklist

- `require_login()` (and `require_capability()`) at top of every entry script
- `require_sesskey()` on every state-changing POST
- `required_param($name, PARAM_INT)` / `optional_param(...)` — never raw `$_REQUEST`
- Use `$DB->...` with placeholders — no SQL concat
- Escape output: `s()`, `format_string()`, `format_text()`
- File serving via `pluginfile.php` + `<component>_pluginfile()` callback, never direct path

## CLI Helpers

```bash
php admin/cli/upgrade.php --non-interactive    # apply pending upgrades
php admin/cli/purge_caches.php                  # clear all caches
php admin/tool/behat/cli/init.php               # init behat tests
vendor/bin/phpunit --testsuite local_example_testsuite
vendor/bin/phpcs --standard=moodle local/example
```

## Testing

- PHPUnit: `tests/<thing>_test.php` extending `advanced_testcase`, use `$this->resetAfterTest()`, generators via `self::getDataGenerator()->get_plugin_generator('local_example')`
- Behat: `tests/behat/*.feature` with `@local_example` tag, step definitions in `tests/behat/behat_local_example.php`

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- **Breaking:** a module returning true for `FEATURE_GROUPMEMBERSONLY` fails install/upgrade; remove the case ([MDL-83231](https://tracker.moodle.org/browse/MDL-83231)).
- **Breaking (report builder):** `set_main_table()` alias is mandatory ([MDL-88397](https://tracker.moodle.org/browse/MDL-88397)); columns are sortable by default, add `->set_is_sortable(false)` where needed ([MDL-87404](https://tracker.moodle.org/browse/MDL-87404)).
- **Breaking:** `duration` element throws if `defaultunit` (default `MINSECS`) is not in `units` ([MDL-89434](https://tracker.moodle.org/browse/MDL-89434)); `mod_assign\event\marker_updated` no longer fired, observe `marker_added`/`marker_removed` ([MDL-87709](https://tracker.moodle.org/browse/MDL-87709)); quiz report subplugins must call `$this->print_action_bar(...)` ([MDL-81096](https://tracker.moodle.org/browse/MDL-81096)).
- **Deprecated:** `user/lib.php` functions → `\core\user::*` (e.g. `\core\user::create_user()`) ([MDL-82650](https://tracker.moodle.org/browse/MDL-82650)); format `get_return_section()` → `get_page_section()`, `get_view_url()` `'sr'` → `'pagesectionid'` ([MDL-86284](https://tracker.moodle.org/browse/MDL-86284)).
- Check `\core\session\manager::supports_cookies()`, not the `NO_MOODLE_COOKIES` constant ([MDL-87174](https://tracker.moodle.org/browse/MDL-87174)).
- Course formats returning true from `uses_linear_navigation()` get the prev/next footer on by default unless they add a `format_<name>/enablelinearnav` setting ([MDL-89406](https://tracker.moodle.org/browse/MDL-89406)).
- Plugin CSS must not assume a light page (experimental dark mode; use `--bs-*` variables) ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- New: `before_email_to_user` hook ([MDL-69724](https://tracker.moodle.org/browse/MDL-69724)); `moodle_exception` `previous:` argument ([MDL-88579](https://tracker.moodle.org/browse/MDL-88579)); REST route scopes `#[scopeset]`/`#[unscoped_resource]` ([MDL-89089](https://tracker.moodle.org/browse/MDL-89089)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Moodle Dev Docs: https://moodledev.io
- Plugin types: https://moodledev.io/docs/apis/plugintypes
- Coding style: https://moodledev.io/general/development/policies/codingstyle
- XMLDB: https://moodledev.io/docs/apis/core/dml/xmldb
- Privacy API: https://moodledev.io/docs/apis/subsystems/privacy
- Hooks API (4.4+): https://moodledev.io/docs/apis/core/hooks
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-plugin-release

> Use when preparing a Moodle plugin release — bumping $plugin->version vs $plugin->release, writing CHANGES.md/changelog entries, staging and tagging an annotated vX.Y.Z release, publishing to the Moodle Plugins directory via GitHub Actions, and confirming the release actually landed. Skip when only doing a schema-change version bump (use /moodle-bump-version) or pre-release QA (use moodle-definition-of-done / moodle-release-preflight).

# Moodle Plugin Release

## Overview

Release mechanics for a Moodle plugin: the version numbers, the notes, the
commit and tag, publishing, and checking that the release really arrived. The
tag freezes a commit and, with a tag-triggered release workflow, **publishes it
the moment it is pushed**, so every step before the push has to be read back,
not assumed.

## When to Use

- A release has been decided on and the version bump approved
- Setting up tag-triggered publishing to the Moodle Plugins directory
- Checking whether a pushed tag actually reached the Plugins directory / Packagist
- **Skip when:** the change only needs a `$plugin->version` bump for a schema
  change (`/moodle-bump-version`), or you are still doing QA (walk
  `moodle-definition-of-done` and `moodle-release-preflight` first).

## 0. Preconditions — stop if any fails

- **The bump was asked for.** Releases are deliberate; don't bump `release`
  as a side effect of other work. Default to a patch (`z`) increment; a minor
  or major bump needs a stated reason.
- **QA is done**: `moodle-definition-of-done` walked for everything going into
  the release; `moodle-release-preflight` walked if the plugin is being
  submitted or its security surface changed.
- **Run every static step of the plugin's own CI locally** and see each exit 0,
  not a subset. phpcs passing does not cover `phpdoc` (moodlecheck) docblock
  rules, and a missing `@param` found by Actions after tagging costs a moved tag:

      grep -n 'moodle-plugin-ci' .github/workflows/*.yml
      for c in phplint phpcpd phpmd validate savepoints; do
          moodle-plugin-ci $c ./; echo "$c exit=$?"
      done
      for c in phpcs phpdoc; do
          moodle-plugin-ci $c --max-warnings 0 ./; echo "$c exit=$?"
      done

  `--max-warnings` exists only on `phpcs` and `phpdoc`; on other commands it
  aborts with "option does not exist", which looks like findings but is a
  broken invocation. `phpmd` exits 0 over violations, so read its output.
  `mustache` and `grunt` need a full Moodle checkout (`-m`), as in CI.

## 1. `version.php` — two different numbers

```php
$plugin->version  = 2026092800;   // YYYYMMDDXX build number — must strictly increase
$plugin->release  = '1.4.2';      // human, semver-like — bumped deliberately
$plugin->requires = 2025041400;   // minimum Moodle build
$plugin->supported = [500, 502];  // RANGE [low, high] of Moodle branches
$plugin->maturity = MATURITY_STABLE;
```

| Number | When it changes | Rule |
|---|---|---|
| `version` | Any schema/upgrade/capability/cache-definition change **must** bump it; every release bumps it | Strictly greater than every previously shipped value; date-serial |
| `release` | Only when cutting a release | Doesn't drag along with a schema bump; patch by default |

A schema change needs a `version` bump plus a matching `upgrade.php` savepoint
(see `/moodle-bump-version`), but does **not** by itself mean a new `release`.

## 2. Release notes — keep all of them consistent

- `CHANGES.md` (if the plugin uses it) — this version's notes; this is what
  reviewers and the Plugins directory see.
- `changelog.md` / `CHANGELOG.md` — prepend `## [x.y.z] - YYYY-MM-DD` with
  `### Added / Changed / Fixed / Security` (Keep a Changelog).
- `README.md` — update if behaviour or requirements changed (supported
  Moodle versions must match `$plugin->supported`).
- Every claim must be verifiable: list only checks you actually ran.

## 3. Stage explicitly — never `git add -A`

Other work may be sitting in the tree. Stage named paths, or re-read
`git status --porcelain` immediately before staging and investigate any file
you don't expect (don't sweep it in, don't delete it).

## 4. Commit, read back, then tag

```bash
git commit -m "Release 1.4.2: <one-line summary>"
git show --stat HEAD          # file list must match the release notes
git tag -a v1.4.2 -m "1.4.2: <one-line summary>"
git describe --tags --exact-match HEAD   # tag is on the commit you just read
```

If the read-back surprises you, delete the tag before it goes anywhere and fix
the commit. Never leave a tag on a commit whose contents you haven't verified.

## 5. Publishing to the Moodle Plugins directory

A tag-triggered workflow using the moodlehq reusable release workflow:

```yaml
# .github/workflows/moodle-release.yml
name: Release Plugin version to Moodle Marketplace
on:
  push:
    tags: ['v*']
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag to be released (e.g. v1.4.0)'
        required: true
jobs:
  release-to-marketplace:
    uses: moodlehq/moodle-plugin-release/.github/workflows/moodle-release.yml@main
    with:
      tag: ${{ inputs.tag }}
    secrets:
      MOODLE_MARKETPLACE_TOKEN: ${{ secrets.MOODLE_MARKETPLACE_TOKEN }}
```

- The plugin must already exist in the directory (first submission is manual);
  the token comes from your moodle.org account and is stored as a repo secret.
- With this workflow present, **pushing a `v*` tag publishes the release**.
  Say so explicitly when handing a push to someone else, so they can hold the
  tag back if they only meant to push the branch.
- A red run usually means a missing or expired token secret; re-run it via
  `workflow_dispatch` with the tag.

## 6. Pushing tags

- Prefer `git push --follow-tags`, or `git push && git push origin v1.4.2`.
- **Avoid pushing many tags in one `git push --tags`:** GitHub does not
  create push events when a single push updates more than three tags, so no
  tag-triggered workflow runs at all. Push tags one at a time or in batches of
  at most three.

## 7. A pushed tag is not a published release

Confirm ingestion downstream; these checks are anonymous HTTPS reads:

- **Plugins directory:** the release workflow run went green, and the version
  appears on the plugin's page.
- **Packagist** (if the plugin is on Composer): the newest version clients
  will actually see:

      curl -s https://repo.packagist.org/p2/<vendor>/<package>.json \
        | python3 -c "import json,sys;d=json.load(sys.stdin);k=list(d['packages'])[0];print(d['packages'][k][0]['version'])"

  Until this prints the new tag, no client-side cache clearing helps.

A failed remote read (auth error, network) is not evidence that something is
missing. Record the release as "committed and tagged, push pending" until the
push is confirmed, never as pushed.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `release` bumped along with every schema change | Release numbers drift from actual releases | Bump `version` for schema; `release` only when releasing |
| Tagging after phpcs alone | CI fails on `phpdoc`/`savepoints` after the tag is public | Run every CI step locally first |
| `git add -A` for the release commit | Unrelated files shipped and tagged | Stage named paths; read back `git show --stat` |
| `--max-warnings` passed to every command | Commands abort; looks like findings | Only `phpcs` and `phpdoc` accept it |
| `git push --tags` with 4+ new tags | No workflow runs, nothing published | Push ≤3 tags per push |
| `$plugin->supported = [502]` | Malformed: core throws `coding_exception` ("Incorrect syntax in plugin supported declaration") | `[502, 502]` for a single branch |
| Treating a pushed tag as released | Clients can't see the version; time lost debugging caches | Check the workflow run and Packagist p2 JSON |

## References

- Version file: https://moodledev.io/docs/apis/commonfiles/version.php
- Plugin release workflow: https://github.com/moodlehq/moodle-plugin-release
- Plugin contribution checklist: https://moodledev.io/general/community/plugincontribution/checklist
- moodle-plugin-ci: https://moodlehq.github.io/moodle-plugin-ci/
- See also: `moodle-definition-of-done`, `moodle-release-preflight`, `moodle-ci-matrix`


---

### moodle-privacy-gdpr

> Use when implementing or reviewing the Moodle privacy provider (GDPR) — null_provider vs request\plugin\provider, get_metadata, export_user_data, delete_data_for_user_in_context, delete_data_for_users, get_contexts_for_userid, get_users_in_context, subsystem links, and core_userlist_provider.

# Moodle Privacy / GDPR

## Overview

Every Moodle plugin **must** declare a privacy provider in `classes/privacy/provider.php`. The provider tells Moodle what user data the plugin stores so that the data export and deletion workflows (Site admin > Users > Privacy) work correctly. Without a provider, the privacy compliance report flags the plugin.

## When to Use

- Creating any new plugin (mandatory)
- Adding a DB table that stores user-identifying data
- Reviewing privacy compliance for plugin directory submission
- Responding to a GDPR data subject request

**Skip when:** the plugin only ships static files / no DB tables — still needs `null_provider` though.

## Decision tree

```
Plugin stores no user-identifying data?
└── implement \core_privacy\local\metadata\null_provider

Plugin stores user data, all of it via core subsystems (files, comments, ratings)?
└── implement \core_privacy\local\metadata\provider
        + link subsystems via add_subsystem_link

Plugin stores user data in its own tables?
└── implement \core_privacy\local\metadata\provider
       + \core_privacy\local\request\plugin\provider
       + \core_privacy\local\request\core_userlist_provider
```

## Null provider (no user data)

```php
<?php
namespace local_example\privacy;
defined('MOODLE_INTERNAL') || die();

class provider implements \core_privacy\local\metadata\null_provider {
    public static function get_reason(): string {
        return 'privacy:metadata';
    }
}
```

Lang string `lang/en/local_example.php`:

```php
$string['privacy:metadata'] = 'The Example plugin does not store any personal data.';
```

## Full provider (plugin tables hold user data)

```php
<?php
namespace local_example\privacy;
defined('MOODLE_INTERNAL') || die();

use core_privacy\local\metadata\collection;
use core_privacy\local\request\approved_contextlist;
use core_privacy\local\request\approved_userlist;
use core_privacy\local\request\contextlist;
use core_privacy\local\request\userlist;
use core_privacy\local\request\writer;

class provider implements
    \core_privacy\local\metadata\provider,
    \core_privacy\local\request\plugin\provider,
    \core_privacy\local\request\core_userlist_provider {

    public static function get_metadata(collection $collection): collection {
        $collection->add_database_table('local_example_items', [
            'userid'      => 'privacy:metadata:items:userid',
            'name'        => 'privacy:metadata:items:name',
            'content'     => 'privacy:metadata:items:content',
            'timecreated' => 'privacy:metadata:items:timecreated',
        ], 'privacy:metadata:items');

        // External system call:
        $collection->add_external_location_link('moodleorg', [
            'username' => 'privacy:metadata:moodleorg:username',
        ], 'privacy:metadata:moodleorg');

        // Subsystem link (files, comments):
        $collection->add_subsystem_link('core_files', [], 'privacy:metadata:filepurpose');

        return $collection;
    }

    public static function get_contexts_for_userid(int $userid): contextlist {
        $contextlist = new contextlist();
        $sql = "SELECT ctx.id
                  FROM {local_example_items} i
                  JOIN {context} ctx ON ctx.contextlevel = :ctxlevel
                                    AND ctx.instanceid = i.courseid
                 WHERE i.userid = :userid";
        $contextlist->add_from_sql($sql, [
            'ctxlevel' => CONTEXT_COURSE,
            'userid'   => $userid,
        ]);
        return $contextlist;
    }

    public static function get_users_in_context(userlist $userlist): void {
        $context = $userlist->get_context();
        if ($context->contextlevel !== CONTEXT_COURSE) {
            return;
        }
        $sql = "SELECT userid FROM {local_example_items} WHERE courseid = :courseid";
        $userlist->add_from_sql('userid', $sql, ['courseid' => $context->instanceid]);
    }

    public static function export_user_data(approved_contextlist $contextlist): void {
        global $DB;
        $user = $contextlist->get_user();
        foreach ($contextlist->get_contexts() as $context) {
            if ($context->contextlevel !== CONTEXT_COURSE) {
                continue;
            }
            $rows = $DB->get_records('local_example_items', [
                'courseid' => $context->instanceid,
                'userid'   => $user->id,
            ]);
            $data = (object)[
                'items' => array_map(fn($r) => [
                    'name'        => $r->name,
                    'content'     => $r->content,
                    'timecreated' => \core_privacy\local\request\transform::datetime($r->timecreated),
                ], $rows),
            ];
            writer::with_context($context)->export_data(
                [get_string('pluginname', 'local_example')],
                $data
            );
        }
    }

    public static function delete_data_for_all_users_in_context(\context $context): void {
        global $DB;
        if ($context->contextlevel !== CONTEXT_COURSE) {
            return;
        }
        $DB->delete_records('local_example_items', ['courseid' => $context->instanceid]);
    }

    public static function delete_data_for_user(approved_contextlist $contextlist): void {
        global $DB;
        $user = $contextlist->get_user();
        foreach ($contextlist->get_contexts() as $context) {
            if ($context->contextlevel !== CONTEXT_COURSE) {
                continue;
            }
            $DB->delete_records('local_example_items', [
                'courseid' => $context->instanceid,
                'userid'   => $user->id,
            ]);
        }
    }

    public static function delete_data_for_users(approved_userlist $userlist): void {
        global $DB;
        $context = $userlist->get_context();
        if ($context->contextlevel !== CONTEXT_COURSE) {
            return;
        }
        [$insql, $params] = $DB->get_in_or_equal($userlist->get_userids(), SQL_PARAMS_NAMED);
        $params['courseid'] = $context->instanceid;
        $DB->delete_records_select('local_example_items',
            "courseid = :courseid AND userid $insql", $params);
    }
}
```

## Required lang strings

```php
$string['privacy:metadata']                       = 'Stores user attendance items.';
$string['privacy:metadata:items']                 = 'Information about user-created items.';
$string['privacy:metadata:items:userid']          = 'The ID of the user who created the item.';
$string['privacy:metadata:items:name']            = 'The name of the item.';
$string['privacy:metadata:items:content']         = 'The body of the item.';
$string['privacy:metadata:items:timecreated']     = 'The time the item was created.';
$string['privacy:metadata:moodleorg']             = 'Items synced to moodle.org.';
$string['privacy:metadata:moodleorg:username']    = 'The username sent to moodle.org.';
$string['privacy:metadata:filepurpose']           = 'Files attached to items.';
```

Every column listed in `add_database_table` and every external location field needs a string.

## Subsystem links

If you use a core subsystem that stores user data on your behalf:

| Subsystem | Constant |
|-----------|----------|
| Files | `'core_files'` |
| Comments | `'core_comment'` |
| Ratings | `'core_rating'` |
| Tags | `'core_tag'` |
| Plagiarism | `'core_plagiarism'` |
| Portfolio | `'core_portfolio'` |
| Logs | `'core_log'` |
| Backup | `'core_backup'` |

```php
$collection->add_subsystem_link('core_files', [], 'privacy:metadata:filepurpose');
```

The subsystem provider handles export/delete; you only declare the link.

## Activity module specifics

Activities also implement `\mod_<name>\privacy\provider` with cm-context awareness. Use `\core_privacy\local\request\helper::get_context_data($context, $user)` to include the activity instance metadata in exports.

## Testing the provider

```php
// tests/privacy/provider_test.php
use core_privacy\tests\provider_testcase;

final class provider_test extends provider_testcase {
    public function test_get_contexts_for_userid(): void {
        $this->resetAfterTest();
        // ... set up data ...
        $contextlist = provider::get_contexts_for_userid($user->id);
        $this->assertCount(1, $contextlist);
    }
}
```

Useful core helper: `\core_privacy\tests\provider_testcase` provides `export_context_data_for_user`, `delete_data_for_user`, etc.

Run all privacy tests:

```bash
vendor/bin/phpunit --group core_privacy
```

## Compliance report

Site admin > Users > Privacy and policies > Plugin privacy registry. Lists every component with its declared metadata. Plugins missing a provider show as **Not yet implemented** (red) — fails moodle.org plugin directory review.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| No provider at all | Add at minimum `null_provider` |
| `null_provider` when DB stores user data | Switch to full provider |
| Missing `core_userlist_provider` | Required since Moodle 3.6+ |
| `add_database_table` columns missing lang strings | Every field needs a `privacy:metadata:...` string |
| `delete_data_for_user_in_context` (old name) | Method is `delete_data_for_user(approved_contextlist)` |
| Forgetting `add_subsystem_link('core_files')` when using files | Add — core files provider handles deletion |
| Returning `transform::datetime` from non-datetime column | Only for unix timestamps |
| `delete_data_for_all_users_in_context` not honoring context level | Always check `$context->contextlevel` first |
| Missing in plugin directory review | All providers required for moodle.org listing |

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- Boost's experimental colour mode stores user preference `theme_boost_colourmode` (declared via `add_user_preference()` and exported in `theme_boost\privacy\provider::export_user_preferences()`) and mirrors it into a cookie of the same name; list it in site cookie notices ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Privacy API: https://moodledev.io/docs/apis/subsystems/privacy
- Implementing the API: https://moodledev.io/docs/apis/subsystems/privacy/api
- Subsystems: https://moodledev.io/docs/apis/subsystems/privacy/api#subsystems
- Testing: https://moodledev.io/docs/apis/subsystems/privacy/api#testing
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-release-preflight

> Use when about to release a Moodle plugin, submit it to the Moodle Plugins directory, or send it for an external security review, or when reviewing security-relevant changes before a tag. A pre-release checklist of defect classes that real Marketplace and security reviews repeatedly caught in shipped plugins despite green CI — client-trusted ids, client-only limits, inline-served uploads, unescaped output sinks, unclean restore, privacy provider drift, incomplete course reset/uninstall, vacuous tests, and minor gates. Skip when writing new feature code or doing a general security audit (use moodle-security-audit).

# Moodle Release Preflight

## Overview

A pre-release self-audit that walks the defect classes external Marketplace and
security reviews have repeatedly found in *shipped* Moodle plugins that green CI and phpcs did not catch. The goal is
to catch the next instance before a reviewer does.

This is not a general security audit. `moodle-security-audit` (and the
`/moodle-security-review` command) teach the full checklist — `require_login`,
sesskey, `$DB` placeholders, SSRF, secrets. This skill is narrower and
empirical: only the classes real reviews caught, each with a concrete check and
fix. Run both before a release; they overlap on purpose at the sinks.

## When to Use

- Before tagging a release or uploading to the Moodle Plugins directory
- Before sending a plugin to an external security or code review
- Reviewing a change that touches request handling, uploads, output, restore,
  privacy, or course reset
- **Skip when:** writing new feature code (use `moodle-plugin-development`) or
  doing a broad security review (use `moodle-security-audit` /
  `/moodle-security-review`)

## How to run it

Scope to the changed files, or the whole plugin for a release. For each class:
run the check, report **pass / fail / N/A-with-reason**, and cite evidence
(file:line, grep output, a test name). "Looks fine" is not a pass. The
deliverable is the report — do not fix unless asked.

**Triage by plugin shape first.** Decide from the file tree, not the prefix:

- **Editor (`tiny_`/`atto_`), filter, block, theme** — usually no request
  handlers, uploads, restore, or stored data. Classes **1, 2, 3, 5, 7 are
  typically N/A**; focus on **4, 6, 8, 9**. Prove N/A with a grep (no
  `required_param`, `send_stored_file`, or `backup/` dir) — don't assume.
- **`mod_`, `assignsubmission_`/`assignfeedback_`, anything with an upload
  endpoint, external functions, or its own tables** — run **all nine**.
- **`local_`, `tool_`, `report_`** — a `local_` with an AJAX endpoint and a
  table is activity-shaped (run all); a passive one is editor-shaped.

```bash
grep -rlnE "required_param|optional_param|send_stored_file" --include='*.php' . ; ls backup db/install.xml 2>/dev/null
```

The positive model for classes 1–3: one shared helper per plugin that loads the
instance from the URL id, calls `validate_context()` / `require_capability()`,
and re-scopes *every* client-supplied id to that instance.

## 1. Trust boundary — never trust a client-supplied id/value

**Seen in reviews:** a report plugin took the target course from a hidden form
field, and its async grade-push task never re-checked the capability →
cross-course gradebook write. An activity trusted a client-sent "type"
value. Another left one id in a `<select>` un-revalidated against the menu that
built it.

Check:
- Grep for request values that drive authorization, storage, or a privileged
  action: `required_param`, `optional_param`, `addElement('hidden'`, `$data->`
  fields from a form, external-function parameters.
- Is authorization derived from the **URL/context** id, not the submitted one?
- Is every submitted id **re-validated** against the allowed set — the same
  records that built the select, or an instance-scoped query?
- Is the capability re-checked at the **async sink** (scheduled/adhoc task,
  service call)? A tampered stored value executes later.

```php
$cm = get_coursemodule_from_id('myplugin', $cmid, 0, false, MUST_EXIST);
$context = context_module::instance($cm->id);
require_login($cm->course, false, $cm);
require_capability('mod/myplugin:grade', $context);

$options = $DB->get_records_menu('myplugin_accounts', ['instanceid' => $cm->instance], '', 'id, name');
if (!array_key_exists($data->accountid, $options)) {
    throw new moodle_exception('invalidaccount', 'mod_myplugin');
}
```

**Question engine:** `$quba->process_all_actions($timenow, $postdata)` processes
the slots named in `$postdata['slots']`, or every slot in the usage if that key
is absent, so a client-shaped `$postdata` can grade every question in one
request. Filter `$postdata` to the allowed slots'
`$quba->get_field_prefix($slot)` keys and set `$postdata['slots']` yourself. A
regression test must include `:sequencecheck` plus a hostile `-submit`, or it
passes vacuously.

### 1b. Two roles writing the same state value into one column

**Seen in reviews:** an owner-facing visibility toggle reused the status value
that moderators used for a takedown. Authors could silently undo
moderation. The readers of the column had been checked; the writers had not.

Check: for every status/enum/flag column the change starts writing, grep every
**other writer** — `grep -rnE "set_field\(.*'status'|update_record|'status' *=>" --include='*.php' .`.
Can two privilege levels put a row into the same state? Then the column records
neither who set it nor who may unset it.

Fix: a **distinct** value per actor (e.g. `hidden_by_moderator`, which the author setter
refuses to touch), or store the actor on the row. Confirm the privileged action
is still reversible through the UI afterwards.

## 2. Client-side-only limits are advisory

**Seen in reviews:** plugins enforced maximum duration and size only in the
browser; a direct POST ignored both.

Check: for each limit (size, duration, count, rate) grep the server endpoint for
a matching guard. If the only enforcement is in `amd/src/*.js`, it is bypassable.

Fix: enforce server-side, independent of JS — `get_user_max_upload_file_size()`
/ `get_max_upload_file_size()`, a duration probe, a `$DB->count_records()` gate.

## 3. Student-uploaded files: force-download unless proven media

**Seen in reviews:** student files were served inline with no content-type
validation, giving stored XSS in the grader's session. Another plugin with the
same gap was protected only because it forced download.

Check:
- `grep -rn "send_stored_file(" --include='*.php' .` — the 4th argument is
  `$forcedownload`. For student files it must be true unless the MIME type is a
  validated audio/video type.
- The upload endpoint validates type/extension against an **allowlist** before
  storing — not just `PARAM_FILE` on the name.
- The `pluginfile` callback allowlists `$filearea`.

```php
$mimetype = $file->get_mimetype();
$ismedia = in_array($mimetype, ['audio/webm', 'audio/ogg', 'video/webm', 'video/mp4'], true);
send_stored_file($file, 0, 0, $forcedownload || !$ismedia, $options);
```

## 4. Escape at the output sink

**Seen in reviews:** a label built with `html_writer::tag('span', $label)` — tag
*content* is not escaped — and then assigned via `innerHTML` → stored XSS.

Check:
- `grep -rnE "html_writer::(tag|div|span|link)\(" --include='*.php' .` — is the
  content argument a stored/user value without `s()` / `format_string()` /
  `format_text()`? Attributes are escaped; content is not.
- `grep -rnE "innerHTML|insertAdjacentHTML|outerHTML|dom\.create\(|setContent\(" amd/src` — then **trace
  what feeds the sink**. HTML from `Templates.renderForPromise()` of a template
  using `{{ }}` (no triple-mustache) is a PASS; string-concatenated HTML is a
  FAIL. A grep hit is not a finding until the feed is traced.
- In TinyMCE plugins, `editor.dom.create('div', {}, html)` is a sink; it passes
  only when `html` comes from `Templates.renderForPromise()`.

Fix: escape at the point of output — `s($label)` — which neutralises the payload
however it entered the DB.

**Content written into core tables is rendered by core — sometimes `noclean`.**
Question text and question-category info render without cleaning, and
`clean_text()` skips HTMLPurifier when the text has no `<`, `>` or `&` (or only
p/em/strong/br tags), so a
`FORMAT_MARKDOWN` link like `[x](javascript:...)` survives and renders as a live
`javascript:` href. For imported or untrusted fields, clean every field yourself
and convert or clamp `FORMAT_MARKDOWN`. Prove "no sink" by grepping core's
output code, not just the plugin.

**Probing a sink with a payload:** write the attacker-shaped data inside a
PHPUnit test with `$this->resetAfterTest()`. A CLI script that includes
`config.php` writes to the live database, even inside a transaction you roll back
by hand.

## 5. Backup/restore is untrusted input

**Seen in reviews:** a plugin's forms cleaned a label (`PARAM_TEXT` plus admin
`validation()`), but the restore step's `process_*()` wrote it verbatim — the
only injection path.

Check: every `process_*()` in `backup/moodle2/restore_*` applies the same
`clean_param()` / allowlist as the interactive form before `insert_record()` /
`update_record()`. Where a unique index exists, restore must upsert, not blindly
insert.

```php
protected function process_myplugin_override($data) {
    global $DB;
    $data = (object) $data;
    $data->label = clean_param($data->label, PARAM_TEXT);
    $data->courseid = $this->get_courseid();
    $DB->insert_record('local_myplugin_override', $data);
}
```

## 6. Privacy provider: declared == handled, and tested

**Seen in reviews:** a provider's discovery methods queried pre-rename table
names → GDPR export and erasure threw `dml_exception`. Another declared a
field in `get_metadata()` but never exported or deleted it.

Check:
- Every table/field in `get_metadata()` appears in **both** an export path and
  every delete path (`delete_data_for_all_users_in_context`,
  `delete_data_for_user`, `delete_data_for_users`).
- After any table rename, grep `classes/privacy/` for the old name.
- A `tests/privacy/provider_test.php` exercises `get_contexts_for_userid`,
  `get_users_in_context`, export and delete against **generated** rows — this
  catches both failures above.

## 7. Course lifecycle completeness

**Seen in reviews:** an activity had `<mod>_reset_userdata()` but no
`_reset_course_form_definition()` / `_reset_course_form_defaults()` → the reset
checkbox never appeared and grades were never reset. Plugins writing core grade
items or redacting core content had no `db/uninstall.php` → orphaned data.

Check (data-storing plugins):
- Reset is the full triad: `myplugin_reset_course_form_definition()`,
  `myplugin_reset_course_form_defaults()`, `myplugin_reset_userdata()` (core
  calls `<modname>_reset_...` with no `mod_` prefix, e.g. `forum_reset_userdata()`)
  — including the gradebook reset.
- `db/uninstall.php` exists if the plugin writes outside its own tables.
- Backup/restore round-trips every field and remaps cross-activity ids in
  `after_restore()`.

## 8. Tests exist and cover the sinks

**Seen in reviews:** several plugins ran `moodle-plugin-ci phpunit` and `behat`
over an **empty** `tests/` directory — the steps pass vacuously. Green CI is not
coverage.

Check: `find tests -name '*_test.php' -o -name '*.feature'` — are there real
tests, and do they cover the paths this audit touched (upload, capability,
privacy)? If CI runs test steps over an empty `tests/`, say so.

## 9. Minor gates (cheap, recurring)

- `error_log()` in a best-effort `catch` — flagged by phpcs. Prefer silent
  handling, `debugging()`, or an event; if kept, document why and add the
  `phpcs:ignore`.
- Scaffolding placeholders left in file headers (a generator's default
  `@copyright` holder, `TODO` package names) — grep for them.
- `version.php` `$plugin->requires` not below the lowest branch in
  `$plugin->supported`.
- Timezones: passing another user's raw `timezone` field (the `99` "server
  default" sentinel) to `get_user_timezone()` or `core_date::get_user_timezone()`
  resolves to the *viewer's* zone — use `core_date::get_server_timezone()` for
  that case. JS wall-clock-to-timestamp by diffing offsets is off
  by the DST amount inside a transition window.

## Common mistakes

| Mistake | Why it fails review | Fix |
|---|---|---|
| Capability checked on the submitted course/cm id | Attacker picks the context | Derive context from the URL id; re-validate submitted ids |
| Capability checked only at request time | Adhoc/scheduled task runs a tampered stored value | Re-check at the sink |
| Limit enforced only in `amd/src` | Direct POST bypasses it | Mirror every limit server-side |
| `send_stored_file(..., false, ...)` for student files | Stored XSS in grader session | Force-download unless validated media |
| `html_writer::tag('span', $userval)` | Content arg is not escaped | `s()` / `format_string()` at the sink |
| Reporting every `innerHTML` grep hit | Template-rendered HTML is already escaped | Trace the feed before reporting |
| Restore writes fields verbatim | `.mbz` is attacker input | Same `clean_param()` as the form |
| Provider tests absent | Renamed tables break export/erase silently | Test with generated rows |
| Reset hook without form definition/defaults | Reset option never shown | Implement the full triad |
| "CI is green" as evidence | Empty `tests/` passes vacuously | List the tests that cover each sink |

## Report format

A short table: class → pass / fail / N/A → evidence. List each fail with
file:line and the fix pattern. Do not fix unless asked.

## Extending this checklist

When a Marketplace or security review finds a defect class not listed here, add
it as a new numbered section in the same shape — what was seen (described
generically), the check, the fix — so the next release is audited against it.

## References

- https://moodledev.io/general/development/policies/security
- https://moodledev.io/general/community/plugincontribution/checklist
- https://moodledev.io/docs/apis/subsystems/privacy
- https://moodledev.io/docs/apis/subsystems/output
- https://moodledev.io/docs/apis/subsystems/files
- https://moodledev.io/docs/apis/subsystems/backup


---

### moodle-security-audit

> Use when reviewing or hardening Moodle plugin code for security — capability checks, sesskey CSRF, input validation, SQL injection, XSS, file serving, SSRF, IDOR, secrets handling, and Moodle's specific anti-patterns.

# Moodle Security Audit

## Overview

Moodle has framework-level protections (capability system, sesskey, `$DB` placeholders, output escaping) — but only if used. Most Moodle plugin CVEs come from skipping these. This skill enforces the checklist.

## When to Use

- Reviewing a PR for security
- Auditing existing plugin code
- Responding to a vulnerability report
- Pre-submission review for moodle.org plugin directory

**Skip when:** writing new feature code (use `moodle-plugin-development` — it covers the basics).

## Audit checklist (every entry script)

```php
require('../../config.php');                            // bootstrap
require_login();                                        // 1. session + cookies
$context = context_course::instance($courseid);         // 2. context
$PAGE->set_context($context);
require_capability('local/example:view', $context);     // 3. capability
require_sesskey();                                      // 4. CSRF (POST only)

$id   = required_param('id', PARAM_INT);                // 5. typed input
$name = optional_param('name', '', PARAM_TEXT);
```

Order matters. `require_login` before `context_*::instance` because login resolves $USER.

## 1. Authentication — `require_login()`

| Variant | Use |
|---------|-----|
| `require_login()` | Logged-in user, any context |
| `require_login($course)` | Logged-in + enrolled in course |
| `require_login($course, true, $cm)` | Logged-in + activity-level access |
| `require_admin()` | Site admin only |
| `require_login(null, false)` | Skips guest auto-login (rare) |

**Bug:** Forgetting `require_login()` on AJAX endpoints. AJAX still needs it — session might be valid but unauthorized.

## 2. Authorization — capabilities

```php
require_capability('local/example:edit', $context);
// or for soft check:
if (!has_capability('local/example:edit', $context)) {
    redirect(...);
}
```

**Bug — IDOR (Insecure Direct Object Reference):**

```php
// BAD — checks site-level cap, but $item belongs to a course user can't access
$item = $DB->get_record('local_example_items', ['id' => $id], '*', MUST_EXIST);
require_capability('local/example:view', context_system::instance());

// GOOD — derive context from the object
$item = $DB->get_record('local_example_items', ['id' => $id], '*', MUST_EXIST);
$context = context_course::instance($item->courseid);
require_capability('local/example:view', $context);
```

Always derive `$context` from the **object** being acted on, not from URL params.

## 3. CSRF — sesskey

Every state-changing request (POST, GET that mutates) needs:

```php
require_sesskey();
```

In forms via `formslib`: `MoodleQuickForm` adds it automatically. In raw HTML forms:

```php
<input type="hidden" name="sesskey" value="<?php echo sesskey(); ?>">
```

In AJAX `core/ajax`: token attached automatically. In raw `fetch`:

```javascript
import {sesskey} from 'core/config';
// include in body
```

**Bug:** GET request that mutates DB without sesskey check.

## 4. Input — never raw `$_GET`/`$_POST`/`$_REQUEST`

```php
$id    = required_param('id', PARAM_INT);                // throws if missing
$name  = optional_param('name', '', PARAM_TEXT);
$ids   = required_param_array('ids', PARAM_INT);
$tags  = optional_param_array('tags', [], PARAM_RAW);
```

`PARAM_RAW` accepts any input — use only with explicit downstream sanitization. `PARAM_TEXT` strips tags; `PARAM_NOTAGS` is similar; `PARAM_CLEANHTML` allows safe HTML.

**Bug:** `$_REQUEST['id']` direct access — bypasses type coercion, vulnerable to type juggling.

## 5. SQL — placeholders only

```php
// GOOD
$DB->get_records_sql('SELECT * FROM {local_example_items} WHERE name = ?', [$name]);
$DB->get_records_sql('... WHERE name = :name', ['name' => $name]);
$DB->get_records('local_example_items', ['courseid' => $cid]);

// BAD — concatenation
$DB->get_records_sql("SELECT * FROM {local_example_items} WHERE name = '$name'");

// SUBTLE BUG — string interpolation in IN clause
$ids = implode(',', $idarray);
$DB->get_records_sql("... WHERE id IN ($ids)");

// FIX — get_in_or_equal
[$insql, $params] = $DB->get_in_or_equal($idarray);
$DB->get_records_sql("... WHERE id $insql", $params);
```

**Bug:** Building `LIKE` patterns by concatenation — use `$DB->sql_like()` + `$DB->sql_like_escape()`.

```php
$pattern = '%' . $DB->sql_like_escape($search) . '%';
$where = $DB->sql_like('name', '?', false);   // case-insensitive
$DB->get_records_sql("SELECT * FROM {t} WHERE $where", [$pattern]);
```

## 6. Output — escape everything

| Function | Use |
|----------|-----|
| `s($str)` | Escape for HTML attribute / text |
| `format_string($str, true, ['context' => $c])` | Plain string with filters (multi-lang, etc.) |
| `format_text($html, FORMAT_HTML, ['context' => $c])` | Rich text — runs filters + cleans |
| `clean_text($str, FORMAT_HTML)` | Strip dangerous tags |
| `html_writer::tag('div', $content, $attrs)` | Build HTML safely |
| `$OUTPUT->render_from_template(...)` | Mustache auto-escapes `{{var}}`; raw via `{{{var}}}` |

**Always pass `context`** to `format_string`/`format_text` — controls which filters run.

**Bug:** `echo $row->name` directly — XSS if name contains tags.

```mustache
{{name}}        ← escaped (safe)
{{{name}}}      ← raw HTML (dangerous unless format_text'd in PHP first)
```

## 7. File serving — `pluginfile.php`

Never expose `$CFG->dataroot` paths directly. File areas served via `pluginfile.php` callback in `lib.php`:

```php
function local_example_pluginfile($course, $cm, $context, $filearea, $args, $forcedl, $options = []) {
    require_login($course, false, $cm);

    if ($filearea !== 'attachments') {
        return false;
    }

    $itemid = (int)array_shift($args);
    require_capability('local/example:viewfile', $context);

    $filename = array_pop($args);
    $filepath = '/' . implode('/', $args) . '/';
    $filepath = $filepath === '//' ? '/' : $filepath;

    $fs = get_file_storage();
    $file = $fs->get_file($context->id, 'local_example', $filearea, $itemid, $filepath, $filename);
    if (!$file || $file->is_directory()) {
        return false;
    }

    send_stored_file($file, 0, 0, $forcedl, $options);
}
```

**Bug:** Skipping capability check inside callback. Yes, `pluginfile.php` does session auth but **not** plugin-specific authorization.

## 8. SSRF / external HTTP

```php
$curl = new \curl();                        // Moodle's curl wrapper
$response = $curl->get($url);
```

`\curl` respects `$CFG->curlsecurityblockedhosts` + `$CFG->curlsecurityallowedport`. **Don't** use raw `curl_exec()` or `file_get_contents($url)` for user-supplied URLs.

```php
// Validate URL belongs to allowed scheme
$url = clean_param($url, PARAM_URL);
if (!$url) {
    throw new moodle_exception('invalidurl');
}
```

## 9. File uploads

- Always go through Moodle's draft area + `file_save_draft_area_files`
- Check MIME via `\core_form\filetypes_util` or `accept` filemanager option
- Set `maxfiles`, `maxbytes`, `accepted_types`
- Never trust client-provided `$_FILES['file']['type']`

## 10. Secrets

- Never commit tokens / API keys to `version.php` / source
- Store in `mdl_config_plugins` via `set_config('apikey', $value, 'local_example')`
- Read with `get_config('local_example', 'apikey')`
- For very sensitive data, encrypt with `\core\encryption` (Moodle 4.0+)

## 11. Logging

User actions trigger events; events flow to logs. **Don't log sensitive data:**

```php
// BAD
debugging("Token: $token");

// GOOD
debugging('Token issued for user ' . $userid);
```

## 12. Redirect open-redirect

```php
// BAD
$next = optional_param('returnto', '', PARAM_URL);
redirect($next);

// GOOD — whitelist or use moodle_url
$next = new moodle_url($next);
if ($next->get_host() !== (new moodle_url($CFG->wwwroot))->get_host()) {
    throw new moodle_exception('invalidurl');
}
redirect($next);
```

## Anti-patterns checklist

| Anti-pattern | Fix |
|--------------|-----|
| `$_GET['id']` | `required_param('id', PARAM_INT)` |
| Raw SQL concat | Placeholders `?` or `:name` |
| `echo $userdata` | `s()` / `format_string` / `format_text` |
| Missing `require_login()` | Add at top of every entry script |
| Missing `require_capability()` | Check on object's context, not arbitrary system |
| Missing `require_sesskey()` on POST | Always |
| Direct `$CFG->dataroot` access | Go through `\file_storage` |
| `eval()` / `create_function()` | Never |
| `unserialize($userdata)` | Never on user input — use JSON |
| `file_get_contents($userurl)` | Use `\curl` wrapper |
| `print_error()` | Deprecated — `throw new \moodle_exception(...)` |
| Hard-coded admin user check (`$USER->id == 2`) | `is_siteadmin()` or capability |
| Trusting `$_SERVER['HTTP_*']` | Check `$_SERVER['SERVER_NAME']` instead; headers spoofable |
| `md5($password)` for new code | `password_hash()` / Moodle's `hash_internal_user_password()` |
| Skipping cap check in `pluginfile` callback | Add explicitly — session auth ≠ authorization |

## Static analysis

```bash
# Moodle's local_codechecker (phpcs)
phpcs --standard=moodle local/example

# Psalm / PHPStan with Moodle stubs (community projects)
# https://github.com/MoodleHQ/moodle-local_codechecker
```

## Reporting vulnerabilities upstream

Found a bug in Moodle core? **Don't open a public issue.** Email `security@moodle.org` per the [Moodle security policy](https://moodle.org/security/).

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- **Breaking:** `login/token.php` is POST-only for credentials (query string throws), checks the service before auth, and drops `appsitecheck` ([MDL-87010](https://tracker.moodle.org/browse/MDL-87010)).
- `'allowcorsrequests' => true` in `db/services.php` sends `Access-Control-Allow-Origin: *` for nologin AJAX; set it only on `loginrequired => false` functions safe cross-origin ([MDL-87150](https://tracker.moodle.org/browse/MDL-87150)).
- REST API: personal access tokens via `\core\api\token_manager` and `moodle/api:createtoken` ([MDL-87706](https://tracker.moodle.org/browse/MDL-87706)); a route with neither `#[scopeset]` nor `#[unscoped_resource]` has no scope restriction, so annotate every route ([MDL-89089](https://tracker.moodle.org/browse/MDL-89089)).
- `\core\di::get(\core_auth\validate_user::class)` centralises pre-login checks: `validate_before_external_login($user)` (maintenance, deleted, unconfirmed, suspended, auth disabled, expired credentials), `validate_before_token_login($user)`, `validate_before_web_login($user)` (suspended and auth-disabled only) ([MDL-88580](https://tracker.moodle.org/browse/MDL-88580)).
- HTMLPurifier now allows `<details>`/`<summary>` ([MDL-88618](https://tracker.moodle.org/browse/MDL-88618)); deep-link auto-login is off by default ([MDL-88924](https://tracker.moodle.org/browse/MDL-88924)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Security overview: https://moodledev.io/general/development/policies/security
- Output escaping: https://moodledev.io/docs/apis/subsystems/output#escaping-content
- Capabilities: https://moodledev.io/docs/apis/subsystems/access
- File API: https://moodledev.io/docs/apis/subsystems/files
- DML placeholders: https://moodledev.io/docs/apis/core/dml#placeholders
- Reporting security: https://moodle.org/security/
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-theme-development

> Use when developing Moodle themes — Boost child themes, theme/yourtheme/config.php, SCSS pre/post hooks, layouts, Mustache template overrides, theme settings, $OUTPUT renderer overrides, and theme designer mode.

# Moodle Theme Development

## Overview

Moodle themes live under `theme/<name>/`. Almost all custom themes inherit from **Boost** (the bundled Bootstrap 5–based theme since 4.0). Override SCSS, layouts, and templates rather than building from scratch.

## When to Use

- Branding a Moodle install (logo, colors, fonts)
- Adding theme-level layout overrides
- Overriding a core Mustache template
- Adding theme settings (admin can configure)
- Per-tenant theming via category themes

**Skip when:** changing plugin UI — modify the plugin's own templates/SCSS.

## Theme skeleton

```
theme/yourtheme/
  config.php                    # required: theme manifest
  version.php                   # required: component, version, requires
  lib.php                       # SCSS hooks, post-process callbacks
  settings.php                  # admin settings
  lang/en/theme_yourtheme.php   # required: pluginname
  scss/
    pre.scss                    # injected BEFORE Boost SCSS (variables)
    post.scss                   # injected AFTER Boost SCSS (overrides)
    preset/                     # full preset alternatives
  layout/
    columns2.php                # override Boost's two-column layout
    drawers.php                 # 4.0+ default layout
    columns1.php
    secure.php
    embedded.php
    login.php
    maintenance.php
  templates/                    # override core Mustache (mirror path)
    core/
      navbar.mustache
  classes/
    output/
      core_renderer.php         # extend Boost's core_renderer
  pix/
    favicon.ico
    logo.png
```

## config.php

```php
<?php
defined('MOODLE_INTERNAL') || die();

$THEME->name             = 'yourtheme';
$THEME->parents          = ['boost'];                           // inherit
$THEME->sheets           = [];                                  // CSS files (SCSS preferred)
$THEME->editor_sheets    = [];
$THEME->scss             = function($theme) {
    return theme_yourtheme_get_main_scss_content($theme);       // see lib.php
};
$THEME->layouts          = [
    // override only the layouts you change; rest inherit from Boost
    'frontpage' => [
        'file' => 'columns2.php',
        'regions' => ['side-pre'],
        'defaultregion' => 'side-pre',
    ],
];
$THEME->enable_dock      = false;
$THEME->yuicssmodules    = [];
$THEME->rendererfactory  = 'theme_overridden_renderer_factory';
$THEME->prescsscallback  = 'theme_yourtheme_get_pre_scss';
$THEME->extrascsscallback = 'theme_yourtheme_get_extra_scss';
$THEME->iconsystem       = \core\output\icon_system::FONTAWESOME;
$THEME->haseditswitch    = true;
$THEME->usescourseindex  = true;
$THEME->precompiledcsscallback = 'theme_yourtheme_get_precompiled_css';
$THEME->addblockposition = BLOCK_ADDBLOCK_POSITION_FLATNAV;
```

## lib.php — SCSS pipeline

```php
<?php
defined('MOODLE_INTERNAL') || die();

function theme_yourtheme_get_main_scss_content($theme): string {
    global $CFG;
    $boost = file_get_contents($CFG->dirroot . '/theme/boost/scss/preset/default.scss');
    $custom = file_get_contents(__DIR__ . '/scss/post.scss');
    return $boost . "\n" . $custom;
}

function theme_yourtheme_get_pre_scss($theme): string {
    $scss = '';
    $configurable = [
        'brandcolor' => ['primary'],
    ];
    foreach ($configurable as $configkey => $vars) {
        $value = $theme->settings->{$configkey} ?? null;
        if (empty($value)) {
            continue;
        }
        foreach ($vars as $var) {
            $scss .= '$' . $var . ': ' . $value . ";\n";
        }
    }
    $scss .= file_get_contents(__DIR__ . '/scss/pre.scss');
    return $scss;
}

function theme_yourtheme_get_extra_scss($theme): string {
    return $theme->settings->scss ?? '';
}
```

`pre.scss` runs before Boost — defines variables (`$primary`, `$body-bg`, etc.).
`post.scss` runs after — overrides classes that already exist.

## Theme settings

`settings.php`:

```php
<?php
defined('MOODLE_INTERNAL') || die();

if ($ADMIN->fulltree) {
    $settings = new theme_boost_admin_settingspage_tabs(
        'themesettingyourtheme',
        get_string('configtitle', 'theme_yourtheme')
    );

    // General tab
    $page = new admin_settingpage('theme_yourtheme_general',
        get_string('generalsettings', 'theme_boost'));

    // Color
    $page->add(new admin_setting_configcolourpicker(
        'theme_yourtheme/brandcolor',
        get_string('brandcolor', 'theme_yourtheme'),
        get_string('brandcolor_desc', 'theme_yourtheme'),
        '#0f6fc5'
    ));

    // Logo
    $page->add(new admin_setting_configstoredfile(
        'theme_yourtheme/logo',
        get_string('logo', 'theme_yourtheme'),
        get_string('logo_desc', 'theme_yourtheme'),
        'logo', 0,
        ['maxfiles' => 1, 'accepted_types' => ['.png', '.svg']]
    ));

    // Raw SCSS
    $page->add(new admin_setting_scsscode(
        'theme_yourtheme/scss',
        get_string('rawscss', 'theme_boost'),
        get_string('rawscss_desc', 'theme_boost'),
        '',
        PARAM_RAW
    ));

    $settings->add($page);
}
```

## Layout files

`layout/columns2.php` controls page structure:

```php
<?php
require_once($CFG->dirroot . '/theme/boost/layout/common.php');

$secondarynavigation = false;
$overflow = '';
if ($PAGE->has_secondary_navigation()) {
    $secondarynavigation = true;
    // ...
}

$templatecontext = [
    'sitename' => format_string($SITE->fullname),
    'output' => $OUTPUT,
    'sidepreblocks' => $blockshtml,
    'hasblocks' => strpos($blockshtml, 'data-block=') !== false,
    'bodyattributes' => $bodyattributes,
    'secondarynavigation' => $secondarynavigation,
];

echo $OUTPUT->render_from_template('theme_yourtheme/columns2', $templatecontext);
```

Mustache layout at `templates/columns2.mustache` — copy from Boost and modify.

## Renderer override

`classes/output/core_renderer.php`:

```php
<?php
namespace theme_yourtheme\output;
defined('MOODLE_INTERNAL') || die();

class core_renderer extends \theme_boost\output\core_renderer {
    public function favicon(): \moodle_url {
        $logo = $this->page->theme->setting_file_url('favicon', 'favicon');
        return $logo ? new \moodle_url($logo) : parent::favicon();
    }
}
```

Activated by `$THEME->rendererfactory = 'theme_overridden_renderer_factory';` in `config.php`.

## Template override

To override `core/navbar.mustache`, copy to `theme/yourtheme/templates/core/navbar.mustache`. Moodle resolves theme templates first.

## Theme designer mode (DEV ONLY)

```php
// config.php
$CFG->themedesignermode = true;
```

Disables CSS caching — every page rebuilds SCSS. **Never** in production — kills performance.

After changing SCSS in production: Site admin > Development > Purge caches > "Theme caches".

## SCSS variables (Boost defaults)

```scss
// pre.scss — override before Boost imports
$primary:   #0f6fc5;
$secondary: #6c757d;
$success:   #5cb85c;
$danger:    #d9534f;
$warning:   #f0ad4e;
$info:      #5bc0de;
$body-bg:   #fff;
$body-color: #1d2125;
$font-family-sans-serif: 'Inter', sans-serif;
$navbar-height: 60px;
```

Full list: `theme/boost/scss/moodle/_variables.scss`.

## Per-category / per-cohort theme

Site admin > Appearance > Themes > Theme settings:
- Allow theme changes per-course/category
- `$THEME->allowedscss` for tenant-controlled SCSS

Programmatic:

```php
$category = $DB->get_record('course_categories', ['id' => $catid]);
$category->theme = 'yourtheme';
$DB->update_record('course_categories', $category);
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Editing Boost directly | Create child theme inheriting Boost |
| Theme designer mode in production | Disable, purge theme caches |
| Forgetting `theme_overridden_renderer_factory` | Renderer overrides won't load |
| Heavy logic in `lib.php` SCSS callback | Cache via `precompiledcsscallback` |
| Hardcoded colors in templates | Use CSS variables / SCSS vars from pre.scss |
| Bumping `version.php` not enough | Also purge caches: `php admin/cli/purge_caches.php` |
| Missing FontAwesome icon system | `$THEME->iconsystem = \core\output\icon_system::FONTAWESOME` |
| Layout file echoes raw HTML | Render via Mustache for theme overridability |
| SCSS contrast ratios fail AA | See `moodle-accessibility` skill |
| Logo / favicon as raw URL | Use `admin_setting_configstoredfile` + `setting_file_url` |

## Build + cache

```bash
# After SCSS changes (production)
php admin/cli/purge_caches.php

# Theme designer mode (dev) — auto-rebuild
# config.php: $CFG->themedesignermode = true;

# Build SCSS to CSS once for inspection
php admin/cli/build_theme_css.php --themes=yourtheme
```

## Moodle 5.3 notes (beta — re-verify at 5.3.0)

- **Breaking:** Classic removed; upgrade migrates its settings to Boost. Re-parent Classic child themes (or install Classic) before upgrading ([MDL-88351](https://tracker.moodle.org/browse/MDL-88351)).
- **Breaking templates:** `theme_boost/drawer` `{{$drawerheadercontent}}` → `{{$drawercontrols}}` ([MDL-89050](https://tracker.moodle.org/browse/MDL-89050)); course-index `cm`/`section` ARIA moved, override both ([MDL-88949](https://tracker.moodle.org/browse/MDL-88949)); collapse-all toggle moved to `core_courseformat/local/content` ([MDL-88410](https://tracker.moodle.org/browse/MDL-88410)); block_timeline is React with no renderer/templates ([MDL-88287](https://tracker.moodle.org/browse/MDL-88287)).
- **Dark mode (experimental):** Boost sets `data-bs-theme` on `<html>`; use `--bs-*`/`--mds-*` variables, keep `scss/moodle/dark.scss` the last import, output `\theme_boost\colour_mode::render_menu()` in custom navbars ([MDL-68037](https://tracker.moodle.org/browse/MDL-68037)).
- **Nav markup:** primary/secondary nav are React `NavPill` (`.mds-nav-pill`, not `.nav-link`/`.moremenu`); navbar overrides should include `core/primarymoremenu` ([MDL-87830](https://tracker.moodle.org/browse/MDL-87830), [MDL-89294](https://tracker.moodle.org/browse/MDL-89294)).
- **Moved templates:** `core/loginform` now in core, re-diff overrides ([MDL-89196](https://tracker.moodle.org/browse/MDL-89196)); modal title is `<h2 class="modal-title fs-5">` ([MDL-75699](https://tracker.moodle.org/browse/MDL-75699)); grade action bars use `core/navigation_action_bar`/`core/action_bar` ([MDL-81096](https://tracker.moodle.org/browse/MDL-81096)); `core/external_content_banner` deprecated ([MDL-89290](https://tracker.moodle.org/browse/MDL-89290)).
- **Admin renderer:** override `notifications_page()` instead of `admin_notifications_page()`; drop banner-method overrides ([MDL-89290](https://tracker.moodle.org/browse/MDL-89290)); `upgradekey_form_page()` → `upgradekey_form_page_with_validation($url, false)` ([MDL-87896](https://tracker.moodle.org/browse/MDL-87896)).
- **Fonts:** default is self-hosted Noto Sans; override `$font-family-sans-serif` to keep system fonts ([MDL-88412](https://tracker.moodle.org/browse/MDL-88412)).
- **JS:** import from `'bootstrap'`, not `theme_boost/bootstrap/*`; exception: `util`/`dom` helpers still load directly (`bootstrap/dom/event-handler` on 5.3+) ([MDL-88766](https://tracker.moodle.org/browse/MDL-88766)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## References

- Themes: https://moodledev.io/docs/apis/plugintypes/theme
- Boost theme: https://moodledev.io/docs/apis/plugintypes/theme/boost
- SCSS: https://moodledev.io/docs/apis/plugintypes/theme#scss
- Layouts: https://moodledev.io/docs/apis/plugintypes/theme/layouts
- Theme settings: https://moodledev.io/docs/apis/plugintypes/theme/settings
- Override templates: https://moodledev.io/docs/apis/subsystems/output/templates#overriding-templates
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-upgrade-migration

> Use when upgrading a Moodle plugin across versions, fixing deprecated API usage, or migrating to Moodle 4.x/5.x (incl. 5.1/5.2/5.3) conventions — print_error, add_to_log, formslib changes, external_api namespace, Hooks API, /public doc-root, Routing Engine, PSR-4 migration, PHP 8.1–8.4 upgrades, and required upgrade.txt notes.

# Moodle Upgrade & Migration

## Overview

Moodle deprecates aggressively but rarely removes. Code from Moodle 2.x often still runs in 4.5 — but uses APIs that emit warnings, fail PHP 8 strict checks, or block plugin directory acceptance. This skill catalogs the migration path from each generation.

## When to Use

- Upgrading a plugin to support a newer Moodle version
- Resolving deprecation warnings
- Migrating to PHP 8.1 / 8.2 / 8.3 / 8.4
- Plugin directory submission rejection ("uses deprecated API")
- Cross-version compat (`$plugin->requires` bump)

## Strategy

1. Bump `$plugin->requires` to your minimum target Moodle version
2. Replace deprecated APIs (table below)
3. Add `upgrade.txt` to document changes
4. Run `phpcs --standard=moodle` + visual smoke test
5. Bump `$plugin->version` + add `db/upgrade.php` step if schema changed

## Deprecation table

| Old | New | Since |
|-----|-----|-------|
| `print_error('errkey', 'comp')` | `throw new \moodle_exception('errkey', 'comp')` | 3.7 |
| `add_to_log()` | Trigger an event class | 2.7 |
| `notify('text', 'notifyproblem')` | `\core\notification::error('text')` | 3.0 |
| `redirect($url, 'msg', 0)` | `redirect($url, 'msg', null, \core\notification::INFO)` | 3.0 |
| `external_api` (bare) | `\core_external\external_api` | 4.2 |
| `external_function_parameters` etc. | `\core_external\...` | 4.2 |
| `$mform->setHelpButton()` | `$mform->addHelpButton()` | 2.0 |
| Magic callbacks `<comp>_extend_navigation` | Hooks API `\core\hook\...` (where applicable) | 4.4 |
| `\core\session\manager::write_close()` | Same — but check if needed | — |
| `mtrace()` in non-CLI | Use `debugging()` or proper logger | — |
| `cleardoubleslashes` | Use `clean_param($url, PARAM_URL)` | — |
| `addslashes()` for DB | Use `$DB->...` placeholders | 2.0 |
| `mysql_*` functions | `$DB` always | 2.0 |
| `userdate($t, '', false)` (no timezone) | Pass timezone or use `core_date::get_user_timezone_object()` | 3.2 |
| `create_function()` | Closures `function() { }` | PHP 7.2 removed |
| `each($array)` | `foreach` | PHP 7.2 removed |
| `assert($string)` | Real expression | PHP 7.2+ |
| `(unset)` cast | Plain `unset()` | PHP 7.2+ |
| `\Mustache_Engine` direct | `$OUTPUT->render_from_template` | — |
| `cm_info::get_modinfo` second arg | API change in 3.4 | 3.4 |
| `format_text` no `context` | Always pass `['context' => $context]` | 2.0 |
| `\html_writer::nonempty_tag` | Use `html_writer::tag` with check | 3.5 |
| `\stdClass` w/o `: \stdClass` return type in PHP 8.1+ | Add return types | 8.1 |

## PHP version migrations

### → PHP 8.0

- `each()`, `create_function()` removed
- `'string' . null` warns — explicit cast
- Negative string offset behavior changed
- Named arguments — Moodle prohibits in public APIs (positional only)

### → PHP 8.1

- Implicit nullable parameters `(string $x = null)` deprecated → `(?string $x = null)`
- `Returning by reference from a void function`
- Internal classes `#[\AllowDynamicProperties]` if you set undeclared properties
- `tempnam`, `parse_url` minor signature changes

### → PHP 8.2

- Dynamic properties on user classes deprecated
- `${...}` string interpolation deprecated → `{$...}`
- `utf8_encode`/`utf8_decode` deprecated

### → PHP 8.3

- `#[\Override]` attribute encouraged
- `Date_Create_From_Format` strict-ness
- Typed class constants supported

### → PHP 8.4

- Implicit nullable params now hard-deprecated: `function f(string $x = null)` → `function f(?string $x = null)`
- Optional param before required param deprecated — reorder so optionals come last
- `E_STRICT` constant removed (was already a no-op)
- `trigger_error(..., E_USER_ERROR)` deprecated — throw an exception instead
- CSV functions: `fputcsv()` / `fgetcsv()` / `str_getcsv()` default `$escape` deprecated — pass `''` explicitly to opt out of legacy escape
- `xml_set_*_handler` string-callable form deprecated — pass `[$obj, 'method']` or first-class callable
- `mysqli_kill()`, `mysqli_refresh()`, `mysqli_ping()` deprecated — irrelevant to Moodle ($DB layer), but flag in custom code
- `DatePeriod` ISO8601 string constructor deprecated → `DatePeriod::createFromISO8601String()`
- `mb_trim()`, `mb_ltrim()`, `mb_rtrim()` added — prefer over manual regex trimming for multibyte

## Moodle 3.x → 4.x

### Theme

- 3.x default theme `Clean` removed; use Boost
- 4.0 introduced course index, secondary navigation, drawers
- Layouts: `frontpage`, `mydashboard`, `incourse`, `course` may need overrides

### Hooks API (4.4+)

Old:
```php
function local_example_extend_navigation(global_navigation $nav) { /* ... */ }
```

New (where supported):
```php
// db/hooks.php
$callbacks = [
    [
        'hook'     => \core\hook\navigation\primary_extend::class,
        'callback' => '\local_example\hooks\navigation::extend_primary',
    ],
];
```

```php
// classes/hooks/navigation.php
namespace local_example\hooks;
class navigation {
    public static function extend_primary(\core\hook\navigation\primary_extend $hook): void {
        $hook->get_primary_view()->add(/* ... */);
    }
}
```

Magic callbacks still work in 4.4+; Hooks API is preferred for new code, required for some new extension points.

### External API namespace

```php
// Pre-4.2
class get_items extends external_api { /* ... */ }

// 4.2+
use core_external\external_api;
use core_external\external_function_parameters;
class get_items extends external_api { /* ... */ }
```

Compatibility shim: 4.2+ aliases bare names to namespaced for back-compat — but new code should use namespaced.

### Course format API

`format_<name>` plugins migrated from `format_base` to `\core_courseformat\base`:

```php
// classes/output/courseformat/content.php
namespace format_yourname\output\courseformat;
class content extends \core_courseformat\output\local\content { /* ... */ }
```

3.x format plugins need full rewrite for 4.0+.

## Moodle 4.x → 5.x

### 5.0 (2025-04)

- PHP 8.2 minimum
- Bootstrap 5 fully replaces Bootstrap 4 utility classes
  - `.sr-only` → `.visually-hidden`
  - `.float-left` → `.float-start`
  - `.ml-*` → `.ms-*`
  - `data-toggle` → `data-bs-toggle`
- Some 4.x deprecations finalize
- New AI subsystem (`\core_ai`)

### 5.1 (2025-10-06)

- **PHP 8.2 min, 8.3 & 8.4 supported.** Sodium ext required. 64-bit only. `max_input_vars` ≥ 5000.
- **`/public` document root.** Web server must point at `<moodleroot>/public`. Plugins still live above (`/local/...`, `/mod/...`) — installer relocates; manual moves needed for in-place upgrades.
- **Routing Engine** (optional, BC-safe) — new request dispatcher + cleaner URLs.
- Deprecations:
  - `file_encode_url()` deprecated → use `moodle_url::make_pluginfile_url()`
  - Device-related theme functions final-deprecated
  - Quiz callback classes/functions deprecated (see `mod/quiz/UPGRADING.md`)
  - `course/changenumsections.php` page removed
  - Course "max sections" setting removed
- DB prefix max length now 10 chars.
- Always check per-component `UPGRADING.md` (5.1 replaces `upgrade.txt` for API change notes).

### 5.2 (2026-04-20)

- **PHP 8.3 min, 8.4 supported.**
- **Upgrade path:** must come from 4.4 or later. Older → upgrade through 4.4/5.0/5.1 first.
- **Oracle DB removed.** Minimums bumped: PostgreSQL 16, MySQL 8.4, MariaDB 10.11, SQL Server 2019.
- **React in core** — base library available via importmaps; new UI code may use React components.
- Composer support for third-party library installation in plugins.
- OpenTelemetry integration.
- Final deprecations:
  - `core/modal_factory`, `core/modal_registry` (AMD) — use `core/modal` directly
  - Pre-PHP 7 style constructors removed (no method named after class)
  - Everything in `lib/deprecatedlib.php` from ≤ 4.4 removed
- New/expanded web services: `site_info`, `mobile_config`, `choice_results`.

### 5.3 (beta — re-verify at 5.3.0)

- **Requirements:** upgrade from 4.4+; PHP 8.3 min; MariaDB 11.4, PostgreSQL 17, MySQL 8.4, SQL Server 2019 ([MDL-86887](https://tracker.moodle.org/browse/MDL-86887)). Beta is `$version = 2026091600.00`, branch `503`.
- **Classic theme removed:** settings migrate to Boost; re-parent Classic child themes or install Classic first ([MDL-88351](https://tracker.moodle.org/browse/MDL-88351)).
- **Now fatal:** `FEATURE_GROUPMEMBERSONLY` true blocks install/upgrade ([MDL-83231](https://tracker.moodle.org/browse/MDL-83231)); legacy `external_*()` functions throw ([MDL-76583](https://tracker.moodle.org/browse/MDL-76583)); `set_main_table()` needs an alias ([MDL-88397](https://tracker.moodle.org/browse/MDL-88397)); `duration` `defaultunit` must be in `units` ([MDL-89434](https://tracker.moodle.org/browse/MDL-89434)).
- **Behaviour:** report columns sortable by default ([MDL-87404](https://tracker.moodle.org/browse/MDL-87404)); `queue_adhoc_task(..., true)` returns an existing id ([MDL-86422](https://tracker.moodle.org/browse/MDL-86422)); `marker_updated` event not fired ([MDL-87709](https://tracker.moodle.org/browse/MDL-87709)); AI token columns moved to `ai_action_register` ([MDL-89123](https://tracker.moodle.org/browse/MDL-89123)); quiz reports must call `print_action_bar()` ([MDL-81096](https://tracker.moodle.org/browse/MDL-81096)).
- **Deprecated:** `user/lib.php` functions → `\core\user::*` ([MDL-82650](https://tracker.moodle.org/browse/MDL-82650)); global `\external_*` names ([MDL-81225](https://tracker.moodle.org/browse/MDL-81225)); `get_return_section()`/`'sr'` → `get_page_section()`/`'pagesectionid'` ([MDL-86284](https://tracker.moodle.org/browse/MDL-86284)); `add_navitem()` → `add_menu_item()` ([MDL-88938](https://tracker.moodle.org/browse/MDL-88938)); `NO_MOODLE_COOKIES` checks → `\core\session\manager::supports_cookies()` ([MDL-87174](https://tracker.moodle.org/browse/MDL-87174)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

Always check `/public/lib/upgrade.txt` (path changed in 5.1) and per-component `UPGRADING.md` for the target version.

## upgrade.txt convention

`<plugin>/upgrade.txt`:

```
This files describes API changes in /local/example/*,
information provided here is intended especially for developers.

=== 1.5.0 ===
* The deprecated function local_example_old_thing() has been removed. Use local_example_new_thing() instead.
* New \local_example\hooks\foo class for the navigation hook (Moodle 4.4+).

=== 1.4.0 ===
* New web service local_example_export_csv.
```

Mirrors Moodle's own `lib/upgrade.txt`. Plugin directory reviewers read it.

## db/upgrade.php cumulative example

```php
<?php
defined('MOODLE_INTERNAL') || die();

function xmldb_local_example_upgrade($oldversion) {
    global $DB;
    $dbman = $DB->get_manager();

    if ($oldversion < 2024010100) {
        $table = new xmldb_table('local_example_items');
        $field = new xmldb_field('status', XMLDB_TYPE_INTEGER, '4', null,
            XMLDB_NOTNULL, null, '0', 'name');
        if (!$dbman->field_exists($table, $field)) {
            $dbman->add_field($table, $field);
        }
        upgrade_plugin_savepoint(true, 2024010100, 'local', 'example');
    }

    if ($oldversion < 2024050100) {
        // Drop legacy index
        $table = new xmldb_table('local_example_items');
        $index = new xmldb_index('legacy_idx', XMLDB_INDEX_NOTUNIQUE, ['userid']);
        if ($dbman->index_exists($table, $index)) {
            $dbman->drop_index($table, $index);
        }
        upgrade_plugin_savepoint(true, 2024050100, 'local', 'example');
    }

    if ($oldversion < 2025010100) {
        // Data migration — chunked to avoid memory blow
        $rs = $DB->get_recordset_select('local_example_items', 'metadata IS NULL');
        foreach ($rs as $r) {
            $r->metadata = json_encode([]);
            $DB->update_record('local_example_items', $r);
        }
        $rs->close();
        upgrade_plugin_savepoint(true, 2025010100, 'local', 'example');
    }

    return true;
}
```

Each `if` block targets one historical version. Never delete past blocks — sites may upgrade through multiple versions.

## Compatibility shim pattern

For plugins supporting multiple Moodle versions:

```php
if (class_exists('\core_external\external_api')) {
    class_alias('\core_external\external_api', '\local_example\compat\external_api');
} else {
    class_alias('\external_api', '\local_example\compat\external_api');
}
```

Or guard with `\core\plugin_manager::get_remote_plugin_info` and `moodle_major_version()`:

```php
if (version_compare(moodle_major_version(), '4.4', '>=')) {
    // Hooks API path
} else {
    // Magic callback fallback
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Bumping `$plugin->requires` without testing on min version | Test against `$plugin->requires` Moodle release |
| Removing `db/upgrade.php` blocks | Keep all historical blocks |
| Skipping `upgrade_plugin_savepoint` | Required — marks step as done |
| Adding new field without `field_exists` guard | Use guard in case site is mid-upgrade |
| Forgetting `upgrade.txt` | Plugin directory reviewers expect it |
| Removing deprecated function used by other plugins | Mark deprecated for one major, then remove |
| Using PHP 8.2 features without bumping `$plugin->requires` Moodle | Match Moodle's PHP min |
| `print_error` left after migration | Replace with `throw new moodle_exception` |
| `external_api` namespace mismatch breaking 4.1 | Use compatibility alias |

## Tools

```bash
# Find deprecated API usage
phpcs --standard=moodle --report=summary local/example
moodle-cs/standards/Moodle/...

# Find PHP version issues
phpcompatinfo analyser:run local/example
phpstan analyse local/example --level=5
```

## References

- Plugin upgrade flow: https://moodledev.io/docs/guides/migrating
- API deprecations: https://moodledev.io/general/development/policies/deprecation
- upgrade.txt format: https://moodledev.io/general/development/policies/codingstyle#upgrade
- Hooks API: https://moodledev.io/docs/apis/core/hooks
- Per-version notes: https://moodledev.io/general/releases
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

### moodle-web-services

> Use when adding/calling Moodle web services or external functions — db/services.php, classes/external/, REST/SOAP/AJAX protocols, tokens, capabilities, parameters/returns schemas, file uploads, and rate limiting.

# Moodle Web Services

## Overview

Moodle exposes plugin functions as web services via `db/services.php` + `classes/external/<fn>.php`. Same function is callable as REST, AJAX, SOAP, or XMLRPC. Authentication is token-based for external clients, session-based for AJAX. All inputs/outputs are typed via `external_value` / `external_single_structure` / `external_multiple_structure`.

## When to Use

- Adding a function callable from mobile app, AMD JS, or external system
- Defining a service (bundle of functions) for an external client
- Issuing/managing tokens
- Returning files via web service
- Debugging "Invalid response value" or "Missing required parameter"

**Skip when:** building UI-only PHP (no remote callers) — use `moodle-plugin-development`.

## File layout

```
<plugin>/
  db/services.php                                # function + service definitions
  classes/external/get_items.php                 # one file per external function
  classes/external/get_items_returns.php         # (optional) split returns
  lang/en/<component>.php                        # service descriptions
```

## db/services.php

```php
<?php
defined('MOODLE_INTERNAL') || die();

$functions = [
    'local_example_get_items' => [
        'classname'    => 'local_example\external\get_items',
        'methodname'   => 'execute',                // default: 'execute'
        'description'  => 'Return items in a course',
        'type'         => 'read',                   // 'read' or 'write'
        'ajax'         => true,                     // callable from core/ajax
        'capabilities' => 'local/example:view',     // checked in addition to per-method checks
        'services'     => [MOODLE_OFFICIAL_MOBILE_SERVICE], // expose to mobile
    ],
    'local_example_mark_present' => [
        'classname'    => 'local_example\external\mark_present',
        'description'  => 'Mark a user present',
        'type'         => 'write',
        'ajax'         => true,
        'capabilities' => 'local/example:mark',
    ],
];

$services = [
    'Example REST API' => [
        'functions'         => array_keys($functions),
        'restrictedusers'   => 1,        // require explicit user assignment
        'enabled'           => 1,
        'shortname'         => 'local_example_api',
        'downloadfiles'     => 1,
        'uploadfiles'       => 0,
    ],
];
```

After editing: bump `version.php`, run `php admin/cli/upgrade.php`.

## External function class

```php
<?php
namespace local_example\external;
defined('MOODLE_INTERNAL') || die();

use core_external\external_api;
use core_external\external_function_parameters;
use core_external\external_value;
use core_external\external_single_structure;
use core_external\external_multiple_structure;

class get_items extends external_api {

    public static function execute_parameters(): external_function_parameters {
        return new external_function_parameters([
            'courseid' => new external_value(PARAM_INT, 'Course ID'),
            'limit'    => new external_value(PARAM_INT, 'Max items', VALUE_DEFAULT, 50),
        ]);
    }

    public static function execute(int $courseid, int $limit = 50): array {
        global $DB, $USER;

        // 1. Validate params (re-runs schema, normalizes types)
        $params = self::validate_parameters(self::execute_parameters(), [
            'courseid' => $courseid,
            'limit'    => $limit,
        ]);

        // 2. Validate context + capability
        $context = \context_course::instance($params['courseid']);
        self::validate_context($context);
        require_capability('local/example:view', $context);

        // 3. Business logic
        $rows = $DB->get_records('local_example_items',
            ['courseid' => $params['courseid']], 'timecreated DESC', '*', 0, $params['limit']);

        return array_values(array_map(fn($r) => [
            'id'   => (int)$r->id,
            'name' => format_string($r->name, true, ['context' => $context]),
            'time' => (int)$r->timecreated,
        ], $rows));
    }

    public static function execute_returns(): external_multiple_structure {
        return new external_multiple_structure(
            new external_single_structure([
                'id'   => new external_value(PARAM_INT, 'ID'),
                'name' => new external_value(PARAM_TEXT, 'Name'),
                'time' => new external_value(PARAM_INT, 'Created timestamp'),
            ])
        );
    }
}
```

**Required ordering inside `execute()`:**
1. `validate_parameters()`
2. `validate_context()`
3. `require_capability()`
4. business logic
5. format output (`format_string`, `format_text` with context)

Skipping any of (1)-(3) is a security bug.

## Moodle version notes

| Moodle | Namespace |
|--------|-----------|
| 4.2+   | `core_external\external_api` (etc.) |
| ≤ 4.1  | bare `external_api` (no namespace) |

For 4.2+ compatibility wrappers, see `lib/classes/external/`.

**Moodle 5.3 (beta — re-verify at 5.3.0):**
- **Breaking:** global `\external_api`, `\external_value` etc. emit renamed-class notices and `external_format_string()`, `external_generate_token()` etc. throw; `use core_external\...` and `\core_external\util::*` ([MDL-81225](https://tracker.moodle.org/browse/MDL-81225), [MDL-76583](https://tracker.moodle.org/browse/MDL-76583)).
- **Breaking:** `login/token.php` is POST-only for credentials and `appsitecheck` is removed ([MDL-87010](https://tracker.moodle.org/browse/MDL-87010)).
- Build mod `get_*_by_courses` returns from `helper_for_get_mods_by_courses::standard_coursemodule_elements_returns()` ([MDL-87241](https://tracker.moodle.org/browse/MDL-87241)).
- Exporter strings use numeric entities (`&#38;`), also 5.2.2+ ([MDL-79755](https://tracker.moodle.org/browse/MDL-79755)); `'allowcorsrequests' => true` for nologin AJAX only ([MDL-87150](https://tracker.moodle.org/browse/MDL-87150)).
- REST routes: OAuth2 scopes `#[scopeset]`/`#[unscoped_resource]` ([MDL-89089](https://tracker.moodle.org/browse/MDL-89089)); tokens via `\core\api\token_manager` ([MDL-87706](https://tracker.moodle.org/browse/MDL-87706)); OAuth2 server depends on `league/oauth2-server` via Composer, so run `composer install` ([MDL-88457](https://tracker.moodle.org/browse/MDL-88457); status checked via `\core\composer`, [MDL-88576](https://tracker.moodle.org/browse/MDL-88576)).
- Full 5.3 catalogue: `moodle-5-3-changes`.

## Parameter types (PARAM_*)

| Constant | Use |
|----------|-----|
| `PARAM_INT` | Integers |
| `PARAM_FLOAT` | Decimals |
| `PARAM_BOOL` | true/false |
| `PARAM_TEXT` | Plain text (strips tags) |
| `PARAM_RAW` | Untrusted text — use only if you re-format on output |
| `PARAM_NOTAGS` | Strips tags, keeps entities |
| `PARAM_CLEANHTML` | Sanitized HTML |
| `PARAM_ALPHANUM` | a-zA-Z0-9 |
| `PARAM_ALPHA` | a-zA-Z |
| `PARAM_USERNAME` | Validates Moodle username |
| `PARAM_EMAIL` | Validates email |
| `PARAM_URL` | Validates URL |
| `PARAM_FILE` | Filename (no path) |
| `PARAM_PATH` | Path (no `..`) |
| `PARAM_COMPONENT` | frankenstyle (`local_example`) |
| `PARAM_CAPABILITY` | `local/example:view` |
| `PARAM_PLUGIN` | `<name>` part |
| `PARAM_AREA` | File area names |
| `PARAM_BASE64` | Base64 |

VALUE_REQUIRED (default), VALUE_OPTIONAL (omit from response if null), VALUE_DEFAULT (with default).

## Calling from REST

```bash
TOKEN=abcdef0123456789
SITE=https://moodle.example.com

curl -X POST "$SITE/webservice/rest/server.php" \
  -d "wstoken=$TOKEN" \
  -d "moodlewsrestformat=json" \
  -d "wsfunction=local_example_get_items" \
  -d "courseid=5" \
  -d "limit=10"
```

Response on error:

```json
{"exception":"invalid_parameter_exception","errorcode":"invalidparameter","message":"..."}
```

## Calling from AJAX (browser)

```javascript
import Ajax from 'core/ajax';

const [items] = await Ajax.call([{
    methodname: 'local_example_get_items',
    args: {courseid: 5, limit: 10},
}]);
```

Requires `'ajax' => true` in `db/services.php`. Session cookie auths; no token needed.

## Token issuance

```bash
# Site admin > Server > Web services > Manage tokens
# or programmatically:
```

```php
$service = $DB->get_record('external_services', ['shortname' => 'local_example_api']);
$token = \core_external\util::generate_token(
    EXTERNAL_TOKEN_PERMANENT,
    $service,
    $userid,
    \context_system::instance()
);
```

## File uploads via web service

1. Upload to draft area: `POST /webservice/upload.php?token=$T&filearea=draft&itemid=0`
2. Returns `itemid`
3. Pass that `itemid` to your function via a `PARAM_INT` param
4. In `execute()`, move from draft to plugin area:

```php
file_save_draft_area_files($params['draftitemid'], $context->id,
    'local_example', 'attachments', $itemid,
    ['subdirs' => 0, 'maxfiles' => 5]);
```

Service must have `uploadfiles => 1`.

## File downloads

Set `downloadfiles => 1` on service, serve via `pluginfile.php`. Token-authed clients append `?token=$T` to pluginfile URLs.

## Rate limiting

No built-in per-token rate limit. Options:
- `\core\session\manager::is_loggedinas()` checks
- Custom middleware via `lib/setuplib.php` hook
- Reverse-proxy rate limit (nginx `limit_req`)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skipping `validate_parameters()` | Always call first — ensures types match schema |
| Skipping `validate_context()` | Required for capability + permission scoping |
| Returning unformatted user text | Run through `format_string()` / `format_text()` with context |
| Schema mismatch in returns | Wrap response with `clean_returnvalue` in tests |
| Wrong namespace pre-4.2 | Use `external_api` bare; for 4.2+ use `core_external\external_api` |
| `'ajax' => true` missing for browser calls | Calls fail with `accessexception` |
| Not bumping `version.php` after `db/services.php` change | Service won't register — bump + upgrade |
| Token user lacks required capability | `require_capability` fails — assign user to service or grant cap |
| `restrictedusers => 1` but no users assigned | Site admin > Server > Web services > Authorised users |
| `MOODLE_OFFICIAL_MOBILE_SERVICE` exposes more than intended | Audit — only add functions safe for mobile app |

## Testing

```php
// tests/external/get_items_test.php
$this->setUser($user);
$result = get_items::execute($course->id, 10);
$result = external_api::clean_returnvalue(get_items::execute_returns(), $result);
$this->assertCount(2, $result);
```

`clean_returnvalue` is mandatory — catches return schema bugs.

## References

- Web services overview: https://moodledev.io/docs/apis/subsystems/external/services
- Writing a function: https://moodledev.io/docs/apis/subsystems/external/writing-a-function
- Calling from JS: https://moodledev.io/docs/apis/subsystems/external/writing-a-service#calling-from-javascript
- File uploads: https://moodledev.io/docs/apis/subsystems/external/files
- Token API: https://moodledev.io/docs/apis/subsystems/external/security
- See also: `moodle-field-lessons` (generalized field lessons for this area)


---

## Agents (specialized review/scaffolding)

### moodle-reviewer

> Use this agent for a deep, Moodle-specific code review of a plugin or PR diff. Checks coding standards, security, privacy, lang strings, version bumps, deprecations, and tests. Returns a structured report.

You are a senior Moodle plugin reviewer. You have deep knowledge of:
- Moodle coding standards (`moodle-cs`, `phpcs --standard=moodle`)
- Frankenstyle conventions and plugin types
- Security checklist (capabilities, sesskey, input validation, output escaping, SQL placeholders, file API)
- Privacy / GDPR provider correctness
- XMLDB and `db/upgrade.php` conventions
- Web services (`db/services.php` + `classes/external/`)
- AMD JavaScript + Mustache templates
- Moodle 4.4+ Hooks API and deprecations
- PHPUnit + Behat patterns
- Accessibility (WCAG 2.1 AA)

## Your job

When invoked, you review the requested plugin or diff and produce a structured report.

## Process

1. **Scope** — confirm what to review (whole plugin / specific files / git diff). If unclear, ask once, then proceed.
2. **Inventory** — `Glob` the plugin to map structure. Identify plugin type from frankenstyle.
3. **Run checks** in this order, accumulating findings:
   1. **Coding standards** — `phpcs --standard=moodle` if available
   2. **Frankenstyle / structure** — required files for the plugin type present?
   3. **Security** — apply the `moodle-security-audit` checklist
   4. **Privacy** — apply the `moodle-privacy-gdpr` checklist (column ↔ provider mapping)
   5. **DB / XMLDB** — every schema change has a `version.php` bump + `upgrade.php` step + `upgrade_plugin_savepoint`
   6. **Lang strings** — no hard-coded English in user-facing PHP/Mustache/JS
   7. **Web services** — `validate_parameters` + `validate_context` + `require_capability` order, `clean_returnvalue` in tests
   8. **Deprecations** — `print_error`, `add_to_log`, `external_api` (bare), magic callbacks where Hooks API exists
   9. **Tests** — PHPUnit `final class`, `@covers`, `resetAfterTest`; Behat tags + data generators
   10. **Accessibility** — Mustache uses semantic HTML; forms have labels; modals use `core/modal`
4. **Produce report** in this format:

```markdown
# Review: <plugin>

## Summary
- Files reviewed: N
- Critical: N | High: N | Medium: N | Low: N
- Recommendation: APPROVE / REQUEST CHANGES / BLOCK

## Critical
- file.php:42 — <issue> — FIX: <action>

## High
- ...

## Medium
- ...

## Low / Style
- ...

## Positive notes
- (things done well — keep doing them)

## Suggested next steps
1. ...
2. ...
```

## Rules

- Be specific: `file:line` references, exact API names.
- Don't speculate — read the file before claiming an issue.
- Distinguish **incorrect** from **subjective**. Mark subjective items "Style".
- For each Critical/High finding, give a fix that compiles.
- Never modify files. You only report.
- If asked to review a git diff, respect the diff scope — don't audit untouched files.
- If `phpcs` isn't available, note it and run all other checks anyway.


---

### moodle-scaffolder

> Use this agent to generate a complete Moodle plugin skeleton from a brief description. Produces all required files for the plugin type with correct frankenstyle, license headers, version, privacy provider, capabilities, lang strings, and a smoke test.

You are a Moodle plugin scaffolding specialist. You generate complete, idiomatic plugin skeletons that pass `phpcs --standard=moodle` out of the box.

## Inputs

You will be given (or asked to obtain):
- **Plugin type** — `local`, `mod`, `block`, `format`, `theme`, `auth`, `enrol`, `report`, `qtype`, `filter`, `repository`
- **Plugin name** — lowercase, no underscore in name part
- **Target Moodle version** — `requires` field
- **What it does** — short description; informs feature surface
- **User data?** — drives privacy provider type
- **Moodle root path** — where to write files

If any are missing, ask once at the start, then proceed.

## Always include

For every plugin, regardless of type:

1. `version.php` with:
   - `$plugin->component` = frankenstyle (`<type>_<name>`)
   - `$plugin->version` = `YYYYMMDD00` (today)
   - `$plugin->requires` = user-supplied
   - `$plugin->maturity = MATURITY_ALPHA;` (caller can bump later)
   - `$plugin->release = '0.1.0';`

2. `lang/en/<component>.php` with at least:
   - `pluginname`
   - `privacy:metadata` (or per-table strings if full provider)

3. `classes/privacy/provider.php`:
   - `null_provider` if "stores user data?" = no
   - Full provider scaffold (per the `moodle-privacy-gdpr` skill) if yes

4. `db/access.php` if the plugin will have any access-controlled action (default: yes, with a single `<plugin>:view` capability)

5. `README.md` with install instructions

6. `CHANGELOG.md` with `## [0.1.0]` initial entry

7. GPL-3.0-or-later license header on every PHP file

8. `defined('MOODLE_INTERNAL') || die();` after license (skip in `classes/` PSR-4 files)

## Type-specific extras

### local
- Optional `lib.php`, `settings.php`

### mod (activity module)
- `mod_form.php` extending `moodleform_mod`
- `view.php`
- `lib.php` with `<name>_supports`, `<name>_add_instance`, `<name>_update_instance`, `<name>_delete_instance`
- `db/install.xml` with the required `<name>` table (id, course, name, intro, introformat, timecreated, timemodified)

### block
- `block_<name>.php` extending `block_base` with `init()`, `get_content()`
- `db/install.xml` empty (or with config table)

### format (course format)
- `format.php`
- `lib.php` with class extending `\core_courseformat\base`
- `classes/output/courseformat/content.php` (4.0+)

### theme
- `config.php` with `$THEME->parents = ['boost']`
- `lib.php` with `theme_<name>_get_main_scss_content`
- `scss/pre.scss`, `scss/post.scss`
- `settings.php` with at least one configurable color

### auth
- `auth.php` extending `auth_plugin_base`

### enrol
- `lib.php` extending `enrol_plugin`

### report
- `index.php`

### qtype (question type)
- `questiontype.php`
- `question.php`
- `renderer.php`
- `edit_<name>_form.php`

### filter
- `filter.php` extending `moodle_text_filter`

### repository
- `lib.php` extending `repository`

## Plus a smoke test

`tests/<name>_test.php`:

```php
<?php
namespace <component>;
defined('MOODLE_INTERNAL') || die();

/**
 * @group <component>
 * @covers \<component>\anything_at_all
 */
final class smoke_test extends \advanced_testcase {
    public function test_plugin_loads(): void {
        $this->resetAfterTest();
        $this->assertTrue(class_exists(\<component>\privacy\provider::class));
    }
}
```

## After writing

1. Print a tree of created files.
2. Print the next-step commands:
   ```bash
   php admin/cli/upgrade.php --non-interactive
   php admin/cli/purge_caches.php
   vendor/bin/phpcs --standard=moodle <typedir>/<name>
   php admin/tool/phpunit/cli/init.php
   vendor/bin/phpunit <typedir>/<name>/tests
   ```
3. Highlight TODO markers placed in stubs the user must fill in (form fields, capability arch types, etc.).

## Rules

- Never overwrite existing files without explicit confirmation.
- Use `Write` for new files only. If a file exists, ask.
- Code must parse — write valid PHP, valid XMLDB.
- Reference the `moodle-plugin-development` skill for layout details and the `moodle-privacy-gdpr` skill for the provider.
- Output is generated, not interactive — produce a complete tree in one pass after collecting inputs.


---

## Commands (one-shot workflows)

### /moodle-bump-version

> Bump version.php and add an upgrade.php step for a Moodle plugin.

You will bump the Moodle plugin version and wire the upgrade step.

User's input: <args>

Steps:

1. Read `version.php` of the target plugin. If path missing, ask which plugin.
2. Compute next version:
   - Current is `YYYYMMDDXX`
   - If today's date > current date portion: new version = `<today>00`
   - If today's date == current date portion: increment the trailing 2 digits
   - If current somehow > today: increment trailing 2 digits anyway, log warning
3. Update `$plugin->version` in `version.php`. Optionally bump `$plugin->release` per semver if user indicates a breaking/feature change.
4. Open `db/upgrade.php` (create if missing — use the template from the `moodle-plugin-development` skill).
5. Add a new `if ($oldversion < <newversion>) { ... upgrade_plugin_savepoint(...); }` block at the END of the function (after existing blocks, before `return true;`).
6. Inside the block, draft the schema/data change based on the reason argument or by asking the user. Use the XMLDB API (`$dbman = $DB->get_manager()`, `field_exists`, `add_field`, `change_field_*`, `add_index`, etc.). Always guard with `*_exists()` checks.
7. Remind the user:
   - Re-export `db/install.xml` from the XMLDB editor (`/admin/tool/xmldb/`)
   - Run `php admin/cli/upgrade.php --non-interactive`
   - Add an entry in `upgrade.txt`
   - Update `CHANGELOG.md` if maintained

Print the diff and the upgrade command to run.

Reference the `moodle-upgrade-migration` skill for cross-version concerns.


---

### /moodle-capability-audit

> Audit a plugin's capabilities — verifies db/access.php declarations match runtime require_capability/has_capability calls, checks risk bitmasks, default archetypes, and lang strings.

Audit capability declarations vs. usage for the plugin at the given path.

User's input: <args>

## Procedure

1. Resolve plugin path (default: cwd). Read `version.php` to get the frankenstyle `$plugin->component`.
2. Activate `moodle-security-audit` and `moodle-plugin-development` skills.
3. Parse `db/access.php` — extract declared capabilities into a set.
4. Grep the plugin for runtime capability use:
   - `require_capability\(['"]([^'"]+)['"]`
   - `has_capability\(['"]([^'"]+)['"]`
   - `require_all_capabilities\(\[(.*?)\]`
   - `require_any_capability\(\[(.*?)\]`
   - In `db/services.php`: `'capabilities' => '...'`
   - In external function `execute_parameters` files: same as above
5. Cross-reference:
   - **Used but not declared** -> ERROR (call will always deny).
   - **Declared but never used** -> WARNING (dead capability, or used via JS/dynamic — flag for human check).
   - **Declared with risk bitmask but riskBitmask=0** for caps that grant write/admin -> WARNING.
6. For each declared capability, verify:
   - `captype` is one of `read`/`write`.
   - `contextlevel` is a valid `CONTEXT_*` constant.
   - `archetypes` includes a sensible default (most caps should have `'manager' => CAP_ALLOW` at minimum).
   - `riskbitmask` uses correct combination: `RISK_SPAM | RISK_XSS | RISK_PERSONAL | RISK_CONFIG | RISK_DATALOSS | RISK_MANAGETRUST`.
   - A lang string exists in `lang/en/<component>.php` as `$string['<component>:<capname-suffix>'] = '...'` AND a `<capname-suffix>_help` if non-trivial.
7. Check for `clone` or `extends` of risky caps (e.g., cloning `moodle/site:config`) — flag.
8. If any cap uses `CONTEXT_SYSTEM` and grants write/admin, ensure `RISK_CONFIG` or `RISK_DATALOSS` is set.

## Output

```markdown
# Capability audit: <component>

## Summary
- Declared: N | Used: N | Orphan declarations: N | Undeclared usages: N
- Risk-bitmask issues: N | Missing lang strings: N

## Errors
- foo.php:88 — `has_capability('<component>:doit', ...)` but `<component>:doit` not in db/access.php

## Warnings
- db/access.php:42 — `<component>:legacy` declared but no runtime usage (dead?)
- db/access.php:60 — `<component>:writeall` is write/CONTEXT_SYSTEM but no RISK_CONFIG flag

## Missing lang strings
- `<component>:doit` — add `$string['<component>:doit'] = '...';` to lang/en/<component>.php
- `<component>:doit_help` — add `_help` for non-trivial caps

## Suggested next steps
1. Add missing declarations to db/access.php, bump version.php
2. Remove dead caps or document why kept (e.g., used by sibling plugin)
3. Add risk bitmasks
4. Run `php admin/cli/upgrade.php` after version bump to apply changes
```

## Rules

- Never modify files — report only.
- Use `grep -rn` from `Bash`; don't rely on heuristics across files without verifying.
- If `db/access.php` doesn't exist, say so — the plugin declares no capabilities. Still scan for hardcoded `has_capability` strings.
- Capability strings can be built dynamically (`"$component:" . $action`) — flag these as "manual review needed".


---

### /moodle-codestyle

> Run phpcs with the Moodle coding standard and fix reported issues.

You will lint and fix Moodle coding-style issues.

User's input: <args>

If empty, ask for a path.

Steps:

1. Detect the linter:
   - Prefer `vendor/bin/phpcs` (project-local Composer)
   - Fall back to `phpcs` on PATH
   - If `local_codechecker` is installed, prefer its bundled standard

2. Run:
   ```bash
   phpcs --standard=moodle --report=summary <path>
   phpcs --standard=moodle --report=full <path>
   ```

3. If `phpcs` is not installed, instruct the user:
   ```bash
   composer require --dev moodlehq/moodle-cs
   # or as a Moodle plugin:
   # https://github.com/moodlehq/moodle-local_codechecker
   ```

4. Categorize findings:
   - **Auto-fixable** — run `phpcbf --standard=moodle <path>` (whitespace, missing newlines, brace style)
   - **Manual** — missing PHPDoc, naming, missing `MOODLE_INTERNAL`, missing license header, etc.

5. After auto-fix, re-run and address remaining issues:
   - **License header missing** — prepend GPL-3.0-or-later header (template from `moodle-plugin-development` skill)
   - **Missing `MOODLE_INTERNAL`** — add after license (skip if file is in `classes/` PSR-4)
   - **Missing PHPDoc on class/method** — generate doc with `@param`, `@return`, `@throws`
   - **Naming** — convert camelCase function names to snake_case (carefully — check call sites)
   - **Line length > 132** — wrap at logical breakpoints
   - **Indent** — convert tabs to 4 spaces

6. After all fixes, re-run `phpcs` and confirm zero issues.

7. Recommend hooking phpcs into CI:
   ```yaml
   - run: vendor/bin/phpcs --standard=moodle --report=full --extensions=php local/example
   ```

Reference the `moodle-plugin-development` skill for coding standard details.


---

### /moodle-mustache-lint

> Lint Mustache templates for accessibility, security, hard-coded strings, and Moodle template conventions.

Lint Mustache templates in the given path (default: `./templates` of current plugin).

User's input: <args>

## Procedure

1. Resolve target path. If `<args>` empty, glob `**/templates/**/*.mustache` under cwd.
2. Activate `moodle-accessibility` and `moodle-theme-development` skills for rule context.
3. For every `.mustache` file, check:

### Security
- No raw triple-stache `{{{ ... }}}` on user-controlled data unless the value is documented as pre-sanitized HTML (`{{{html}}}` from `format_text()` is OK; raw `{{{name}}}` of free-text input is NOT).
- No `<script>` blocks — JS belongs in `amd/src/` modules, loaded via `{{#js}}` block.
- No inline `onclick=` / `onload=` / `javascript:` URLs.

### Accessibility
- Every `<img>` has `alt=""` (decorative) or `alt="{{#str}}..."` (meaningful).
- Every interactive element is a `<button>` or `<a href>`, not `<div onclick>`.
- Form inputs have associated `<label for="...">` or `aria-label`.
- Buttons that only contain icons have `aria-label` or visually-hidden text.
- Heading order is not skipped (no `<h1>` then `<h4>`).
- Color is not the only signal (icons + text, not just red/green).

### Strings & i18n
- No hard-coded user-facing English. Use `{{#str}}identifier, component{{/str}}`.
- Detect: bare English words in text nodes, button labels, placeholders, titles.
- Aria labels likewise must use `{{#str}}`.

### Moodle conventions
- Template starts with a Mustache comment block documenting context variables:
  ```mustache
  {{!
      @template component/templatename

      Description.

      Context variables required for this template:
      * variable - description
  }}
  ```
- Use `{{#pix}}` for icons, not raw `<i class="fa-...">`.
- Use `{{#js}}` for JS init, not inline `<script>`.
- Use `core/modal` markup, not bootstrap-direct `<div class="modal">`.

### Output
Produce a markdown report:

```markdown
# Mustache lint: <path>

## Summary
- Files scanned: N
- Errors: N | Warnings: N | Info: N

## Errors (must fix)
- templates/foo.mustache:12 — raw {{{user_input}}} without sanitization context — use {{user_input}} or document why raw HTML is safe

## Warnings (should fix)
- templates/foo.mustache:34 — hard-coded "Submit" — use {{#str}}submit, core{{/str}}

## Info
- templates/foo.mustache — missing {{! @template ... }} header

## Suggested next steps
1. ...
```

## Rules

- Never modify files — report only.
- Be specific: `file:line` references.
- If the file is non-existent or empty, say so and stop.
- Distinguish definitely-broken (Errors) from style/convention (Warnings/Info).


---

### /moodle-new-plugin

> Scaffold a new Moodle plugin (asks for type + name).

You will scaffold a new Moodle plugin.

User's input: <args>

If `<args>` is empty or missing required values, ask the user:
1. Plugin type (one of: `local`, `mod`, `block`, `format`, `theme`, `auth`, `enrol`, `report`, `qtype`, `filter`, `repository`)
2. Plugin name (lowercase, no underscores in the name part — frankenstyle becomes `<type>_<name>`)
3. Target Moodle minimum version (default: latest LTS)
4. Whether the plugin will store user data (affects privacy provider choice)

Then:

1. Activate the `moodle-plugin-development` skill for conventions.
2. Create the directory under `<moodle-root>/<typedir>/<name>/` (e.g., `local/attendance/`). If `<moodle-root>` is unknown, ask.
3. Generate the minimum required files for the chosen plugin type using the cheatsheet in the `moodle-plugin-development` skill.
4. Always include:
   - `version.php` with correct frankenstyle component, today's date as version (`YYYYMMDD00`), and the user's `requires` value
   - `lang/en/<component>.php` with at least `pluginname`
   - `classes/privacy/provider.php` — `null_provider` if no user data, otherwise scaffold the full provider per the `moodle-privacy-gdpr` skill
   - GPL-3.0-or-later license header on every PHP file
   - `defined('MOODLE_INTERNAL') || die();` after the license block (skip in `classes/` PSR-4 files)
5. If the plugin type has type-specific required files (e.g., `mod_form.php` for activity modules, `block_<name>.php` for blocks), include them.
6. Print a summary of what was created and the next steps the user should take (run `php admin/cli/upgrade.php`, register capabilities, write tests).

Coding standards: 4-space indent, no closing `?>`, snake_case functions, PascalCase classes.

If unsure about a Moodle version difference, prefer the convention used in Moodle 4.5+ and note any back-compat concerns.


---

### /moodle-privacy-audit

> Audit a Moodle plugin's privacy provider for completeness against its DB schema.

You will audit the privacy provider of a Moodle plugin.

User's input: <args>

Activate the `moodle-privacy-gdpr` skill for context.

Steps:

1. Locate the plugin directory. If unclear, ask.
2. Read `db/install.xml` (or `db/upgrade.php` if no install.xml) — list every table and column.
3. Identify columns that store **user-identifying data**:
   - `userid`, `user_id`, `createdby`, `modifiedby`, `usermodified`, `relateduserid`
   - Free-text fields a user authored (`content`, `comment`, `description`, `note`)
   - File pointers (`itemid` referencing `mdl_files`)
   - Identifiers from external systems linked to a user
4. Read `classes/privacy/provider.php`.
5. For each user-identifying column, check:
   - It is declared in `get_metadata()` via `add_database_table` with a lang string
   - The column has an entry in `lang/en/<component>.php` of the form `privacy:metadata:<table>:<column>`
   - `get_contexts_for_userid` returns contexts where the user has data
   - `get_users_in_context` returns users who have data in a given context
   - `export_user_data` exports the data per context
   - `delete_data_for_user` and `delete_data_for_users` delete it correctly
   - `delete_data_for_all_users_in_context` honors context level
6. Check for subsystem usage:
   - If plugin attaches files via the Moodle Files API, `add_subsystem_link('core_files', ...)` must be present
   - Same for `core_comment`, `core_rating`, `core_tag` if used
7. If plugin stores no user data, check it implements `null_provider` correctly (`get_reason` returns lang string key, `privacy:metadata` lang string defined).
8. Output findings:
   - GREEN: present and correct
   - YELLOW: present but incomplete / missing lang strings
   - RED: missing entirely

Propose a fixed `provider.php` and the missing lang strings if there are issues. Don't write the changes without user confirmation if the plugin is non-trivial.


---

### /moodle-security-review

> Run the Moodle security checklist on a file, directory, or entire plugin.

You will perform a Moodle-specific security review.

User's input: <args>

Activate the `moodle-security-audit` skill for context.

If `<args>` is empty, ask for a file or plugin path.

Audit each PHP file in scope against this checklist. Report findings as:

```
[FILE]:[LINE] [SEVERITY] [CATEGORY] — [issue]
                                         FIX: [actionable fix]
```

Severities: `CRITICAL` (exploitable now), `HIGH` (privilege escalation / data leak), `MEDIUM` (defense in depth), `LOW` (style / hardening).

## Checklist

For every entry script (file with `require('config.php')` or similar):
1. `require_login()` present and called early
2. `require_capability()` against the **object's** context (not arbitrary system context)
3. `require_sesskey()` on every state-changing request (POST or mutating GET)
4. All input via `required_param`/`optional_param` with appropriate `PARAM_*` type — flag any `$_GET`/`$_POST`/`$_REQUEST` usage

For every DB call:
5. Uses `$DB->...` — flag any `mysqli_*`, `PDO`, `mysql_*`
6. Placeholders (`?` / `:name`) — flag string concatenation in SQL
7. `LIKE` patterns use `$DB->sql_like()` + `sql_like_escape()`
8. `IN (...)` uses `$DB->get_in_or_equal()`

For every output:
9. User text rendered via `s()`, `format_string()`, `format_text()` — flag direct `echo $var`
10. `format_string`/`format_text` is called with a `context`
11. Mustache `{{{var}}}` raw output is justified (text already passed through `format_text`)

For file APIs:
12. `pluginfile.php` callback re-checks capability inside callback
13. Files are served via `\file_storage` + `send_stored_file`, never raw `$CFG->dataroot`
14. Uploads use `file_save_draft_area_files` with `maxfiles`, `maxbytes`, `accepted_types`

For HTTP / external:
15. Uses `\curl` wrapper, not raw `curl_exec()` / `file_get_contents($url)`
16. Open redirects: `redirect($next)` with `$next` from user input is validated against site host

For secrets:
17. No tokens / API keys committed in source — use `set_config`/`get_config`

For dangerous PHP:
18. No `eval()`, `create_function()`, `unserialize()` on user input
19. `print_error` is replaced with `throw new \moodle_exception()`

For sessions:
20. `\core\session\manager::write_close()` called early on AJAX endpoints that don't write session

After audit:
- Print summary: `N CRITICAL, N HIGH, N MEDIUM, N LOW`
- Offer to apply fixes one-by-one or as a single patch (ask before writing)
- For unresolved CRITICAL findings, recommend not deploying until fixed


---

### /moodle-string-check

> Find hard-coded English strings that should use get_string() / Mustache {{#str}}.

You will find untranslated user-facing strings in a Moodle plugin.

User's input: <args>

If empty, ask for a plugin path.

Steps:

1. Walk the plugin directory.
2. Read `lang/en/<component>.php` — list defined keys.
3. For each `.php`, `.mustache`, and `.js` file, find user-facing strings that are not loaded via `get_string`/`{{#str}}`/`core/str`:

   PHP heuristics:
   - String literals passed to:
     - `$mform->addElement(..., 'X', '<literal>')` — second arg = label
     - `new admin_setting_configtext(..., '<literal>', '<literal>')` — title + description
     - `new \html_table_cell('<literal>')`, `html_writer::tag('p', '<literal>')`
     - `notification::success('<literal>')`, `\core\notification::error('<literal>')`
     - `redirect($url, '<literal>')`
     - `throw new moodle_exception('<literal>')` — should be a string KEY, not a sentence
   - Direct `echo` of literals over 3 words

   Mustache heuristics:
   - Visible text not wrapped in `{{#str}}KEY, COMPONENT{{/str}}`
   - Aria-labels / titles with literal text

   JS heuristics:
   - String literals passed to `Notification.alert()`, `Toast.add()`, error messages thrown
   - Any UI-visible literal not loaded via `core/str` `get_string`/`get_strings`

4. Skip strings that are clearly identifiers (selectors, classes, route keys, lang keys themselves).

5. Output findings as:

```
[FILE]:[LINE] "<literal>"
   → propose key: <plugin>:<snake_case>
   → in lang/en/<component>.php:
       $string['<key>'] = '<literal>';
   → replacement:
       <PHP>: get_string('<key>', '<component>')
       <Mustache>: {{#str}}<key>, <component>{{/str}}
       <JS>: await getString('<key>', '<component>')
```

6. Optionally, propose adding `lang/en/<component>.php` keys in a single edit and substituting the literals.

Reference the `moodle-plugin-development` skill for lang string conventions.


---
