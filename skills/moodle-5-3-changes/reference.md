# Moodle 5.3 — full change catalogue

Companion to [SKILL.md](SKILL.md). Every developer note in the `## 5.3beta`
section of Moodle's root `UPGRADING.md` (Build: 20260916), grouped by
component in the same order, rewritten for plugin and theme developers. Each
entry gives the kind of change, what it means, what to do, and tracker links.
A final section lists developer-relevant 5.3 changes that the beta notes do
not mention.

Where the beta source code disagrees with the notes, the entry says so and
describes the code. Notes marked **Unverified** were taken from the upgrade
notes or tracker without being confirmed in the code. Re-check everything
against 5.3.0 final.

Legend — **Kind:** added / changed / deprecated / removed / fixed.
**Severity:** breaking (fails or visibly breaks), action (fix when porting),
new (opt-in capability), info (no code action for most plugins).

---

## core

### Boost dark colour mode — plugin CSS must not assume a light page

- **Kind:** added. **Severity:** action.
- Boost gains a dark colour mode, switched on by the experimental setting
  `theme_boost | enablecolourmodes` (default mode in
  `theme_boost | defaultcolourmode`). Boost writes `data-bs-theme` on `<html>`.
- Plugin styles should take colours from Bootstrap custom properties
  (`--bs-body-bg`, `--bs-body-color`, `--bs-secondary-bg`, `--bs-tertiary-bg`,
  `--bs-border-color`, `--bs-emphasis-color`, `--bs-link-color`) or design
  system tokens (`--mds-bg-surface-*`, `--mds-text-*`, `--mds-border-*`), and
  prefer utilities (`bg-body`, `bg-body-secondary`, `bg-body-tertiary`,
  `text-body`, `text-body-secondary`, `text-bg-*`, `border`) over `bg-white`
  or `text-dark`. The individual token names come from the notes and were not
  each checked in code (**unverified** list).
- A colour hard-coded on only one half of a foreground/background pair loses
  contrast. SVGs shown via `<img>` keep their baked-in fill; render icons
  through the icon API so they inherit `currentColor`.
- **Action:** audit `styles.css`/SCSS and templates for literal colours,
  `bg-white`, `text-dark`; scope real mode differences under the dark selector.

```css
[data-bs-theme="dark"] .local_myplugin-status { /* dark-only colour */ }
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-68037

### New hook `\core\hook\email\before_email_to_user`

- **Kind:** added. **Severity:** new.
- `email_to_user()` dispatches this hook. Its public `$email` property is a
  `\core\email` whose public fields (`user`, `from`, `subject`, `messagetext`,
  `messagehtml`, `attachment`, `attachname`, `usetrueaddress`, `replyto`,
  `replytoname`, `wordwrapwidth`) callbacks may change. Callbacks can add
  headers with `add_additional_header(string)` and stop the send with
  `add_block_reason(string)`; if any reason is present the mail is not sent.
- **Action:** plugins that filter, rewrite or suppress outgoing mail register
  a callback in `db/hooks.php`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-69724

### New Behat step `I set the focus on the "<element>" "<selector>"`

- **Kind:** added. **Severity:** new.
- Moves keyboard focus to an element without activating it. JavaScript only:
  it throws outside `@javascript` scenarios.

```gherkin
And I set the focus on the "Save changes" "button"
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-84065

### `login/token.php` hardened; `appsitecheck` removed

- **Kind:** removed/changed. **Severity:** breaking.
- The requested service (enabled external service by shortname) is checked
  before the user is authenticated, so `servicenotavailable` comes first.
  Passing `username` or `password` in the query string throws a
  `coding_exception` — credentials must be POSTed. The `appsitecheck`
  parameter (which returned `{"appsitecheck":"ok"}`) is gone.
- **Action:** clients POST credentials and stop using `appsitecheck=1` to
  probe a site. (The tracker says GET credentials are "emptied"; the beta code
  throws instead.)
- **Tracker:** https://tracker.moodle.org/browse/MDL-87010

### Session cookie-support API replaces `NO_MOODLE_COOKIES` checks

- **Kind:** added. **Severity:** action.
- New static methods on `\core\session\manager`: `supports_cookies()`,
  `set_cookies_supported(bool)`, `set_initial_cookie_support()`.
  `NO_MOODLE_COOKIES` defined before `config.php` still sets the initial
  value. To enable cookies and start a session later, call
  `set_cookies_supported(true)` then `start()`. Turning cookies off after they
  were on is not recommended (you must decide whether to end or close the
  session).
- Routed controllers can pass `cookies: true|false` in the route attribute.
  The notes show `\core\router\attributes\route`; in the beta code the
  attribute class is `\core\router\route`.
- **Action:** replace `defined('NO_MOODLE_COOKIES')` checks with
  `supports_cookies()`.

```php
if (!\core\session\manager::supports_cookies()) {
    \core\session\manager::set_cookies_supported(true);
    \core\session\manager::start();
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-87174

### `moodle_page::set_has_sticky_footer()` / `has_sticky_footer()`

- **Kind:** added. **Severity:** new.
- Records that a sticky footer has been rendered so a second is not.
- **Action:** code rendering its own sticky footer calls
  `$PAGE->set_has_sticky_footer(true)` and checks `has_sticky_footer()` first.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87302

### `moodle_page::set_show_navigation_footer(bool)`

- **Kind:** added. **Severity:** new.
- Suppresses the sticky navigation footer on a page (default true). Getter:
  `should_show_navigation_footer()` (not in the notes).

```php
$PAGE->set_show_navigation_footer(false);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-87575

### `\core\api\token_manager` — REST API personal access tokens

- **Kind:** added. **Severity:** new.
- Mints personal access tokens via `issue_token()`, enforcing the secret
  format (`pat_` prefix) and a maximum lifetime of one year (expiry presets,
  default 30 days). Users manage tokens at `/user/personalaccesstokens.php`,
  which needs the new system capability `moodle/api:createtoken`
  (risks CONFIG, DATALOSS, SPAM, PERSONAL, XSS; manager archetype only).
- **Action:** issue PATs through `token_manager`, not the repository classes
  directly; document the capability.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87706

### `action_menu\subpanel` accepts an optional URL

- **Kind:** changed. **Severity:** new.
- `core\output\action_menu\subpanel` gains a fifth optional argument
  `?moodle_url $url = null`, so the item can link somewhere as well as open
  its subpanel. Existing callers are unaffected.

```php
$item = new \core\output\action_menu\subpanel($text, $subpanel, $attributes, $icon,
    new moodle_url('/local/myplugin/index.php'));
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88312

### New `core\output\submenu` renderable

- **Kind:** added. **Severity:** new.
- A renderable/templatable list of items that can be used as the subpanel of
  an `action_menu\subpanel` to build multi-level menus.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88312

### AMD modules `core/import` and `core/component`

- **Kind:** added. **Severity:** new.
- `core/import` default export is a native dynamic `import()` that Babel does
  not rewrite — for ESM specifiers resolved by the import map (e.g.
  `@moodle/lms/...`). `core/component` exports
  `appendToDom(moduleName, properties, element)` and
  `prependToDom(...)`, which create a `data-react-component` /
  `data-react-props` container for React auto-initialisation. Properties must
  be JSON-serialisable.

```js
import nativeImport from 'core/import';
const mod = await nativeImport('@moodle/lms/mod_book/viewer');
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88505

### `scripts/swizzle.mjs` — eject/wrap React components into a theme

- **Kind:** added. **Severity:** new.
- Node CLI: no arguments starts an interactive wizard to eject or wrap a core
  React component into a theme. `manifest generate` defaults undeclared
  components to risky/risky; `manifest set` sets each component's safety
  level (`safe`, `risky`, `prohibited`) separately for eject and wrap.
- **Action:** components shipping React components should generate a
  manifest and set levels. Each component ships a `swizzle.json` in its
  `js/esm/src/` directory (e.g. `blocks/timeline/js/esm/src/swizzle.json`);
  `manifest generate` aggregates them and adds a risky/risky entry for
  undeclared components.

```sh
./scripts/swizzle.mjs manifest generate
./scripts/swizzle.mjs manifest set
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88509

### `moodle_exception` accepts `$previous`

- **Kind:** changed. **Severity:** new.
- Sixth constructor argument `?\Throwable $previous = null` for exception
  chaining. Subclasses overriding the constructor may want to forward it.

```php
throw new moodle_exception('sitemaintenance', 'admin', previous: $e);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88579

### Password and auth-plugin functions delegate to DI classes

- **Kind:** added. **Severity:** new.
- `validate_internal_user_password`, `hash_internal_user_password`,
  `update_internal_user_password`, `password_is_legacy_hash`,
  `get_password_peppers`, `exceeds_password_length` delegate to
  `\core\authentication\password` (`validate`, `hash`, `update`,
  `is_legacy_hash`, `get_peppers`, `exceeds_max_length`).
  `exists_auth_plugin`, `is_enabled_auth`, `get_auth_plugin`,
  `get_enabled_auth_plugins`, `is_internal_auth`, `is_restored_user` delegate
  to `\core\authentication` (`plugin_exists`, `is_enabled`, `get_plugin`,
  `get_enabled_plugins`, `is_internal`, `is_restored_user`).
- These are **instance** methods resolved via DI, even though the notes write
  `Class::method`. The global functions remain; nothing is deprecated.
- **Action:** new code may inject or `\core\di::get()` them (mockable in
  tests).

```php
$auth = \core\di::get(\core\authentication::class)->get_plugin('manual');
$ok = \core\di::get(\core\authentication\password::class)->validate($user, $password);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88580

### `moodle_page::set_supplementary_content()` / `get_supplementary_content()`

- **Kind:** added. **Severity:** new.
- Puts a secondary `action_link` into the sticky footer (`?action_link`).
- **Tracker:** https://tracker.moodle.org/browse/MDL-88601

### `\core\oauth2\server\client_manager`

- **Kind:** added. **Severity:** new.
- Lifecycle of OAuth2 server clients: `create_client`, `get_client`,
  `get_client_by_id`, `update_client`, `disable_client`, `reactivate_client`,
  `delete_client`; secrets (`create_secret`, `get_secrets`, `revoke_secret`)
  and redirect URIs (`get_redirect_uris`, `add_redirect_uri`,
  `remove_redirect_uri`, `validate_redirect_uri`,
  `validate_redirect_uri_format`).
- **Tracker:** https://tracker.moodle.org/browse/MDL-89181

### `core/imagedetails/modal` — alt text and size before embedding

- **Kind:** added. **Severity:** new.
- `getImageDetails(file)` shows an image-details dialogue and resolves to
  `{alt, presentation, width, height}` or `null` on cancel (0 width/height =
  keep original size).
- **Action:** plugins with their own image upload/embed UI should use it and
  store the alt text or decorative flag.

```js
import {getImageDetails} from 'core/imagedetails/modal';
const details = await getImageDetails(file);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89214

### `html_writer::react_component()`

- **Kind:** added. **Severity:** new.
- `react_component(string $modulename, array|string|\stdClass|\JsonSerializable $props): string`
  returns the React mount point; it JSON-encodes the props itself.

```php
echo \core\output\html_writer::react_component('@moodle/lms/block_timeline/Timeline', ['foo' => 1]);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89296

### `\core\output\react_component_renderable` interface

- **Kind:** added. **Severity:** new.
- A renderable implementing it provides `get_react_component_name(): string`
  and `get_react_component_props(renderer_base $renderer): \stdClass`;
  `$OUTPUT->render()` then renders a React component instead of a template.
- `get_react_component_name()` returns the path inside the `lms` package with
  no `@moodle/lms/` prefix, which the renderer adds (unlike
  `html_writer::react_component()`, which takes the full specifier).

```php
final class my_renderable implements \core\output\react_component_renderable {
    public function get_react_component_name(): string {
        return 'local_myplugin/Widget';
    }
    public function get_react_component_props(\core\output\renderer_base $renderer): \stdClass {
        return (object) ['count' => 3];
    }
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89296

### `flexible_table::set_columnheadersattributes()`

- **Kind:** added. **Severity:** new. **Also in 5.2.2+** (backport) — not a
  5.3-only API.

```php
$table->set_columnheadersattributes(['mycolumn' => ['class' => 'visually-hidden']]);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89384

### Attribute-based dependency injection

- **Kind:** added. **Severity:** new.
- The DI container now honours `#[\DI\Attribute\Inject]` on properties for
  objects obtained through `\core\di::get()` (container entry) or
  `\core\di::make()` (new instance each call). Recommended for routed
  controllers; useful in legacy code and factories.

```php
class example_class {
    #[\DI\Attribute\Inject]
    public \core\formatting $formatter;
}
$example = \core\di::get(example_class::class);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89528

### Modal title is `<h2 class="modal-title fs-5">`

- **Kind:** changed. **Severity:** action.
- `core/modal` renders its title as `<h2>` (was `<h5>`), visually unchanged
  via `fs-5`. Headings in modal bodies should start at `<h3>`.
- **Action:** re-level body headings; custom modal headers use the same
  pattern; update selectors on `h5.modal-title`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-75699

### `\core_user::get_users_search_sql()` gains `$allowcustom`, `$custommappings`

- **Kind:** changed. **Severity:** new.
- Optional `bool $allowcustom = false` includes custom profile identity fields;
  `array $custommappings = []` supplies SQL mappings when your query uses
  different joins/prefix. Existing callers unaffected.
- **Tracker:** https://tracker.moodle.org/browse/MDL-81096

### Secondary navigation rendered by React `core/nav/Nav`

- **Kind:** changed. **Severity:** action.
- Secondary navigation is rendered by the React `core/nav/Nav` (via the
  `core/secondarymoremenu` template; props from
  `\core\navigation\output\more_menu`), and primary navigation by
  `core/nav/PrimaryNav`, using the design system `NavPill`. Markup changes,
  e.g. `.mds-nav-pill` / `.mds-nav-pill--selected` instead of `.nav-link`.
  Keyboard handling remains in `core/menu_navigation`.
- **Action:** retarget theme/plugin CSS, JS and Behat selectors.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87830

### `core/notification_base` accepts `headinglevel`

- **Kind:** changed. **Severity:** new.
- Optional integer 1–6 for the notification title heading; default remains
  `<h3 class="h6 mb-0">`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88458

### `core/search_input_auto` search role in a `{{$searchrole}}` block

- **Kind:** changed. **Severity:** new. Also present in 5.2.2+.
- The `role="search"` attributes are wrapped in an overridable block; empty it
  when the input already sits in another search landmark.

```mustache
{{< core/search_input_auto }}{{$searchrole}}{{/searchrole}}{{/ core/search_input_auto }}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88833

### `\core\task\manager::set_scheduled_task_nextruntime()` returns bool

- **Kind:** changed. **Severity:** info.
- Returns false (and changes nothing) if the task lock cannot be obtained or
  the task is already running; true once `nextruntime` is set. Fixes "Run
  ASAP" being overwritten when the task completes. The method itself is new
  in 5.3 (not in 5.2.2+).
- **Action:** callers check the return value.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89200

### Block uninstall deletes instances via an ad-hoc task

- **Kind:** changed. **Severity:** action.
- `\core\plugininfo\block::uninstall_cleanup()` queues
  `\core\task\delete_block_instances_task` (deduplicated), which deletes
  instances and related data in batches; the `block` record is deleted
  immediately. `before_delete()` still runs synchronously, and an overridden
  `instance_delete()` is still called per instance while the code is present.
- **Action:** don't assume instances/contexts are gone when uninstall
  returns; keep per-instance cleanup in `instance_delete()`; uninstall tests
  may need to run ad-hoc tasks.

```php
\core\task\manager::queue_adhoc_task(
    \core\task\delete_block_instances_task::instance($block->name),
    true,
);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89289

### Boost primary navigation uses `core/primarymoremenu`

- **Kind:** changed. **Severity:** action.
- Boost `navbar.mustache` includes `core/primarymoremenu` (mounting
  `core/nav/PrimaryNav`) instead of `core/moremenu`. With JS there is no
  `.moremenu` wrapper and items are `a.mds-nav-pill` (selected:
  `.mds-nav-pill--selected`, `aria-current="page"`) rather than
  `a.nav-link.active`. A server-rendered `core/moremenu_children` fallback is
  included for non-JS clients.
- **Action:** child themes overriding `navbar.mustache` switch the partial;
  update CSS and Behat selectors.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89294

### 25 `user/lib.php` functions deprecated → `\core\user`

- **Kind:** deprecated. **Severity:** action.
- Now wrappers in `lib/deprecatedlib.php` that emit a developer debugging
  notice and delegate. `user/lib.php` keeps only the `USER_FILTER_*`
  constants, `user_page_type_list()` and `core_user_inplace_editable()`.
  Final removal is tracked for 6.0.

| Old | New |
|---|---|
| `user_create_user()` | `\core\user::create_user()` |
| `user_update_user()` | `\core\user::update_user()` |
| `user_delete_user()` | `\core\user::delete_user()` |
| `user_get_users_by_id()` | `\core\user::get_users_by_id()` |
| `user_get_default_fields()` | `\core\user::get_default_fields()` |
| `user_get_user_details()` | `\core\user::get_user_details()` |
| `user_get_user_details_courses()` | `\core\user::get_user_details_courses()` |
| `can_view_user_details_cap()` | `\core\user::can_view_user_details_cap()` |
| `user_count_login_failures()` | `\core\user::count_login_failures()` |
| `user_convert_text_to_menu_items()` | `\core\user::convert_text_to_menu_items()` |
| `user_get_default_homepage_options()` | `\core\user::get_default_homepage_options()` |
| `user_get_user_navigation_info()` | `\core\user::get_user_navigation_info()` |
| `user_add_password_history()` | `\core\user::add_password_history()` |
| `user_is_previously_used_password()` | `\core\user::is_previously_used_password()` |
| `user_remove_user_device()` | `\core\user::remove_user_device()` |
| `user_list_view()` | `\core\user::list_view()` |
| `user_mygrades_url()` | `\core\user::mygrades_url()` |
| `user_can_view_profile()` | `\core\user::can_view_profile()` |
| `user_process_profile_callbacks()` | `\core\user::process_profile_callbacks()` |
| `user_get_tagged_users()` | `\core\user::get_tagged_users()` |
| `user_get_course_lastaccess_sql()` | `\core\user::get_course_lastaccess_sql()` |
| `user_get_user_lastaccess_sql()` | `\core\user::get_user_lastaccess_sql()` |
| `user_get_lastaccess_sql()` | `\core\user::get_lastaccess_sql()` |
| `user_edit_map_field_purpose()` | `\core\user::edit_map_field_purpose()` |
| `user_update_device_public_key()` | `\core_user\devicekey::update_device_public_key()` (a `\core\user::update_device_public_key()` delegate also exists) |

- **Action:** migrate once your minimum is 5.3. The new methods are typed —
  e.g. `create_user(stdClass $user, bool $updatepassword = true, bool $triggerevent = true): int`
  — so cast arrays to objects. Drop `require_once` of `user/lib.php` where it
  was only for these.

```php
$userid = \core\user::create_user((object) $record);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-82650,
  https://tracker.moodle.org/browse/MDL-89001

### `FEATURE_GROUPMEMBERSONLY` no longer supported

- **Kind:** deprecated (effectively removed). **Severity:** breaking.
- The constant is still defined, but `plugin_supports()` with it throws
  `coding_exception`, and installing/upgrading a module whose `_supports()`
  returns true for it throws `plugin_defective_exception`.
- **Action:** delete the `case FEATURE_GROUPMEMBERSONLY:` line.
- **Tracker:** https://tracker.moodle.org/browse/MDL-83231

### `\core\hub\registration::get_dataroot_size()` deprecated

- **Kind:** deprecated. **Severity:** action. Also in 5.2.2+.
- Replaced by `get_filepool_usage()`, which estimates file pool size from the
  files table (deduplicated by content hash, cached) instead of scanning
  dataroot.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88805,
  https://tracker.moodle.org/browse/MDL-88818

### `core/external_content_banner` template deprecated

- **Kind:** deprecated. **Severity:** action.
- Replaced on the admin notifications page by `core_admin/notification_ctas`;
  final deprecation planned for 6.0.
- **Action:** remove theme overrides of it.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

### Fixed: `core_course` route scope strings

- **Kind:** fixed. **Severity:** info.
- The course content and course structure scope classes now resolve their
  summary/description strings (`course_content_{read,write,delete}_scope_{summary,desc}`,
  `course_structure_...`). The route scope API is itself new in 5.3 (see
  "OAuth2 scopes for REST API routes" below).
- **Action:** when writing scope classes, follow the same string naming.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87706

---

## core_admin

### Admin setting classes moved to autoloaded `core_admin\setting\...`

- **Kind:** added (rename). **Severity:** info.
- About 110 classes from `adminlib.php` are now autoloaded. Old global names
  keep working **without** a notice (each new file ends with `class_alias()`
  and the legacy class map points at it).
- Mapping rules:
  - `\admin_setting` → `\core_admin\setting`
  - `\admin_setting_<x>` and `\admin_settings_<x>` →
    `\core_admin\setting\setting\<x>`
  - `\admin_category`, `\admin_externalpage`, `\admin_root` →
    `\core_admin\setting\tree\{category,externalpage,root}`
  - `\admin_page_<x>` → `\core_admin\setting\page\<x>` (`manageblocks`,
    `managefilters`, `managemessageoutputs`, `managemods`,
    `manageportfolios`, `manageqbehaviours`, `manageqtypes`,
    `managerepositories`, `pluginsoverview`)
  - `\admin_settingpage` → `\core_admin\setting\settingpage\settingpage`;
    `\admin_settingdependency` → `\core_admin\setting\settingpage\dependency`
- `admin_setting_` classes covered: `agedigitalconsentmap`, `bloglevel`,
  `check`, `configbackupfilenamemustachetemplate`, `configcheckbox`,
  `configcheckbox_with_advanced`, `configcheckbox_with_lock`,
  `configcolourpicker`, `configdirectory`, `configduration`,
  `configduration_with_advanced`, `configempty`, `configexecutable`,
  `configfile`, `confightmleditor`, `configiplist`, `configmixedhostiplist`,
  `configmulticheckbox`, `configmulticheckbox2`, `configmultiselect`,
  `configmultiselect_modules`, `configpasswordunmask`,
  `configpasswordunmask_with_advanced`, `configportlist`, `configselect`,
  `configselect_autocomplete`, `configselect_with_advanced`,
  `configselect_with_lock`, `configstoredfile`, `configtext`,
  `configtext_with_advanced`, `configtext_with_maxlength`, `configtextarea`,
  `configthemepreset`, `configtime`, `countrycodes`, `courselist_frontpage`,
  `description`, `emoticons`, `enablemobileservice`, `encryptedpassword`,
  `filetypes`, `flag`, `forcetimezone`, `grade_profilereport`,
  `gradecat_combo`, `heading`, `langlist`, `manage_fileconverter_plugins`,
  `manage_plugins`, `manageantiviruses`, `manageauths`,
  `managecontentbankcontenttypes`, `managecustomfields`,
  `managedataformats`, `manageenrols`, `manageexternalservices`,
  `manageformats`, `managemediaplayers`, `managerepository`,
  `managewebserviceprotocols`, `my_grades_report`, `php_extension_enabled`,
  `pickfilters`, `pickroles`, `question_behaviour`, `regradingcheckbox`,
  `requiredpasswordunmask`, `requiredtext`, `savebutton`, `scsscode`,
  `searchsetupinfo`, `servertimezone`, `sitesetcheckbox`, `sitesetselect`,
  `sitesettext`, `special_adminseesall`, `special_backup_auto_destination`,
  `special_backupdays`, `special_calendar_weekend`, `special_coursecontact`,
  `special_debug`, `special_frontpagedesc`, `special_gradebookroles`,
  `special_gradeexport`, `special_gradeexportdefault`,
  `special_gradelimiting`, `special_grademinmaxtouse`,
  `special_gradepointdefault`, `special_gradepointmax`,
  `special_registerauth`, `special_selectsetup`, `users_with_capability`,
  `webservicesoverview`.
- `admin_settings_` classes covered: `country_select`, `coursecat_select`,
  `h5plib_handler_select`, `num_course_sections` (still deprecated since 5.1
  in favour of `admin_setting_configtext`), `sitepolicy_handler_select`.
- **Action:** none yet. 5.3-minimum plugins may use the namespaced names; keep
  the old names while supporting 5.2.

```php
// Moodle 5.3+ only.
$settings->add(new \core_admin\setting\setting\configtext(
    'local_myplugin/apikey',
    get_string('apikey', 'local_myplugin'),
    '',
    '',
    PARAM_TEXT,
));
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-81935

### `core_admin_renderer::warning_with_label()` added

- **Kind:** added. **Severity:** info.
- Protected `warning_with_label(string $message, string $label, string $type = 'warning')`
  prefixes a notification with a bold severity label, then calls `warning()`
  (unchanged). Available to admin renderer subclasses.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

### Admin notifications page regrouped; "From Moodle" cards

- **Kind:** changed. **Severity:** info.
- `/admin/index.php` groups notifications by severity (danger, warning,
  notice) with a count summary (danger, then warning, then notice; stable
  within each group). The
  campaign, services-and-support, Marketplace and feedback banners become one
  card grid (`core_admin\output\notification_ctas`, template
  `core_admin/notification_ctas`). New config-only
  `$CFG->disablenotificationctas` hides individual cards.

```php
// config.php
$CFG->disablenotificationctas = ['marketplace', 'moodlecloud', 'partners', 'feedback'];
```

- **Action:** themes styling the old banners or overriding the renderer
  re-test the page.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

### `core_admin_renderer::upgradekey_form_page()` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `upgradekey_form_page_with_validation(moodle_url $url, bool $upgradekeyerror)`,
  which already exists in 5.2. (The attribute says "since 5.2", but the 5.2.2+
  code does not carry it.)

```php
echo $output->upgradekey_form_page_with_validation($url, false);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-87896

### `core_admin_renderer::admin_notifications_page()` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `notifications_page()`, which takes the same arguments minus
  `$showcampaigncontent`, `$showfeedbackencouragement` and
  `$showservicesandsupport`. The old method still renders and ignores them.
- **Action:** theme renderers override `notifications_page()` instead.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

### Admin banner renderer methods deprecated

- **Kind:** deprecated. **Severity:** action.
- Protected `campaign_content()`, `services_and_support_content()`,
  `userfeedback_encouragement()`, `marketplace_integration_notice()`.
- **Action:** remove overrides; customise `core_admin/notification_ctas`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

### `$CFG->showcampaigncontent` unused; `showservicesandsupportcontent` narrowed

- **Kind:** removed. **Severity:** info.
- `showcampaigncontent` has no effect. `showservicesandsupportcontent` now only
  controls the "Services and support" link in the help popover.
- **Action:** drop `showcampaigncontent` from config and docs.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89290

---

## core_ai

### `ai_action_register.courseid` and `report_aiusage`

- **Kind:** added. **Severity:** new.
- New `courseid` column (int, not null, default 0) filled at log time from
  the action's context: `-1` means the context is not in a course, `0` means
  an older row not yet backfilled (a background backfill runs). New
  `report_aiusage` course report with `report/aiusage:view` and
  `report/aiusage:viewown`.
- **Action:** per-course AI reporting filters on `courseid` and handles `0`
  and `-1`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-80893

### AI token counts moved to `ai_action_register`

- **Kind:** changed. **Severity:** breaking.
- `prompttokens` and the completion token column moved from
  `ai_action_generate_text`, `ai_action_summarise_text`,
  `ai_action_explain_text` to `ai_action_register` (`prompttokens`,
  `completiontokens`). In 5.2 the child-table column was spelled
  `completiontoken` (all three child tables in 5.2).
- **Action:** update SQL/reports to read the register table.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89123

---

## core_auth

### `\core_auth\validate_user` DI service

- **Kind:** added. **Severity:** new.
- Centralised login checks. Composite: `validate_before_web_login()`,
  `validate_before_external_login()`, `validate_before_token_login()`.
  Individual: `validate_maintenance_mode_access()`, `validate_not_deleted()`,
  `validate_is_confirmed()`, `validate_is_not_suspended()`,
  `validate_auth_not_disabled()`, `validate_credentials_not_expired()`,
  `validate_user_is_not_guest_user()`. All take `\stdClass $user` and return
  void; failures throw typed exceptions in `\core_auth\exception\`
  (`access_denied_exception`, `auth_disabled_exception`,
  `credentials_expired_exception`, `maintenance_mode_enabled_exception`,
  `user_deleted_exception`, `user_is_guest_exception`,
  `user_not_confirmed_exception`, `user_suspended_exception`).
- **Action:** 5.3+ auth plugins and custom login/token endpoints can reuse it.

```php
\core\di::get(\core_auth\validate_user::class)->validate_before_web_login($user);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88580

---

## core_badges

### `badge_message_from_template()` takes optional `$user`

- **Kind:** changed. **Severity:** info.
- Third argument `?stdClass $user = null`; when given, user name
  placeholders are filled too.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87859

---

## core_behat

### `--colourmode` run option and `behat_get_colour_mode()`

- **Kind:** added. **Severity:** new.
- `admin/tool/behat/cli/init.php`, `util.php` and `util_single_run.php` accept
  `--colourmode=<mode>` (e.g. `light`, `dark`) to run the whole suite in that
  mode; the run header reports it; `behat_get_colour_mode()` exposes it.
  Precedence: user preference, then run option, then site settings. The
  option also turns colour modes on, so scenarios assuming they are off should
  start with `Given the run is not using a colour mode`. Reference feature:
  `theme/boost/tests/behat/colour_mode_accessibility.feature`.

```sh
php admin/tool/behat/cli/init.php --colourmode=dark
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-68037

---

## core_course

### Activity chooser items can be disabled

- **Kind:** added. **Severity:** new.
- `core_course\local\entity\content_item` accepts optional
  `bool $disabled = false` and `?string $disabledreason = null`; disabled
  items are greyed out with the reason as a tooltip (`is_disabled()` getter).
- **Tracker:** https://tracker.moodle.org/browse/MDL-87373

### `course_navigation` controller helpers

- **Kind:** added. **Severity:** new (for linear navigation features).
- On `core_course\route\controller\course_navigation`:
  - `get_all_section_cms(modinfo $modinfo, section_info $section): array` —
    section's modules in order, including those in subsections.
  - `get_adjacent_section(modinfo $modinfo, section_info $currentsection, string $direction): ?section_info`
    — `'next'` or `'previous'` (anything else throws); null at the end.
  - `is_first_navigable(cm_info $cm, modinfo $modinfo, array $allsectioncms): bool`
    and `is_last_navigable(...same...)` — first/last accessible element of the
    course; both throw `coding_exception` if `$cm` is not in `$allsectioncms`.
  - `get_section(cm_info $cm): ?section_info` — main (non-delegated) section
    containing a module.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88604

### Course-module web services use standard elements

- **Kind:** changed. **Severity:** action.
- Services returning course modules consistently build from
  `core_course\external\helper_for_get_mods_by_courses::standard_coursemodule_elements_returns()`
  and `standard_coursemodule_element_values()`, aligning
  `core_course_get_course_module(_by_instance)` with module services and
  returning the forced language (`lang`). Per the tracker,
  `mod_assign_get_assignments`, `mod_forum_get_forums_by_courses` and
  `mod_h5pactivity_get_h5pactivities_by_courses` gain fields such as
  `section`, `visible`, `groupmode`, `groupingid`, `lang` (exact per-service
  field diff **unverified**).
- **Action:** clients expect the extra fields; activity plugins' own
  `get_*_by_courses` services should use the helper.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87241

### `get_view_url()` option `sr` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `pagesectionid` (section id) instead of `sr` (section number) in
  `core_courseformat\base::get_view_url()`. `sr` still works when
  `pagesectionid` is absent; no debugging notice is emitted. Removal planned
  for 7.0.

```php
$url = $format->get_view_url($section, ['pagesectionid' => $section->id]);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-86284,
  https://tracker.moodle.org/browse/MDL-88498

### `core_courseformat\base::get_return_section()` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `get_page_section(): ?section_info` (null on the course main page).
- **Tracker:** https://tracker.moodle.org/browse/MDL-86284,
  https://tracker.moodle.org/browse/MDL-88498

---

## core_courseformat

### `base::uses_linear_navigation()` opt-in

- **Kind:** added. **Severity:** new.
- Static method, false by default; formats return true to support
  previous/next activity navigation in the sticky footer. `format_topics` and
  `format_weeks` opt in. The notes say the feature stays disabled by default;
  a later change (MDL-89406, below) makes it on by default for opted-in
  formats.

```php
#[\Override]
public static function uses_linear_navigation(): bool {
    return true;
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-87302

### Behat steps for linear navigation

- **Kind:** added. **Severity:** new.
- `Then the course linear navigation should be visible` /
  `should not be visible`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87575

### `inline_help` key for format options

- **Kind:** added. **Severity:** new.
- A format option definition may carry `inline_help`; the edit form then shows
  the help text and docs link persistently under the setting as a static
  element; this is independent of the `help` key (set only `inline_help` to
  avoid also showing the icon). The notes call it a flag set to true, but the code passes the
  value to `get_formatted_help_string()` as a string identifier — so set it to
  a string id with a matching `<id>_help` string (intended usage
  **unverified**).
- **Tracker:** https://tracker.moodle.org/browse/MDL-88669

### Collapse/expand-all toggle moved to `content`

- **Kind:** changed. **Severity:** breaking for overrides.
- `collapsemenu` is exported by `core_courseformat\output\local\content` and
  rendered once above the section list by `core_courseformat/local/content`,
  no longer by `content\section` / `local/content/section`. The section list
  gets class `has-collapsemenu` when present.
- **Action:** move toggle customisations to the content output class or
  template.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88410

### Course index subsection tree semantics moved

- **Kind:** changed. **Severity:** breaking for overrides.
- In `core_courseformat/local/courseindex/cm`, the `li[role=treeitem]` of an
  activity with a delegated section carries `aria-owns`,
  `aria-labelledby` and `aria-expanded`; `courseindex/section` omits the role
  and these attributes for delegated sections, and the section title needs
  `id="courseindexsection{{number}}-title"`. The section JS updates the closest
  `treeitem`, so the two templates are coupled.

```mustache
{{^component}}
role="treeitem"
aria-owns="courseindexcollapse{{number}}"
aria-labelledby="courseindexsection{{number}}-title"
aria-expanded="{{^indexcollapsed}}true{{/indexcollapsed}}{{#indexcollapsed}}false{{/indexcollapsed}}"
{{/component}}
```

- **Action:** update both templates if you override either.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88949

---

## core_external

### Legacy `external_*` classes and functions

- **Kind:** deprecated. **Severity:** breaking.
- The global class names are no longer `class_alias`es in
  `lib/externallib.php`; they resolve through the renamed-classes map, which
  emits a DEBUG_DEVELOPER debugging notice ('Class ... has been renamed for
  the autoloader and is now deprecated') on first use.

| Old class | New class |
|---|---|
| `\external_api` | `\core_external\external_api` |
| `\restricted_context_exception` | `\core_external\restricted_context_exception` |
| `\external_description` | `\core_external\external_description` |
| `\external_value` | `\core_external\external_value` |
| `\external_format_value` | `\core_external\external_format_value` |
| `\external_single_structure` | `\core_external\external_single_structure` |
| `\external_multiple_structure` | `\core_external\external_multiple_structure` |
| `\external_function_parameters` | `\core_external\external_function_parameters` |
| `\external_util` | `\core_external\util` |
| `\external_files` | `\core_external\external_files` |
| `\external_warnings` | `\core_external\external_warnings` |
| `\external_settings` | `\core_external\external_settings` |

- The functions are final-deprecated stubs (deprecated since 4.4) and
  **throw** `coding_exception` when called:

| Old function | Replacement |
|---|---|
| `external_generate_token()` | `\core_external\util::generate_token()` |
| `external_create_service_token()` | `\core_external\util::generate_token()` |
| `external_delete_descriptions()` | `\core_external\util::delete_service_descriptions()` |
| `external_validate_format()` | `\core_external\util::validate_format()` |
| `external_format_string()` | `\core_external\util::format_string()` |
| `external_format_text()` | `\core_external\util::format_text()` |
| `external_generate_token_for_current_user()` | `\core_external\util::generate_token_for_current_user()` |
| `external_log_token_request()` | `\core_external\util::log_token_request()` |

```php
use core_external\external_api;
use core_external\external_function_parameters;
use core_external\external_value;
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-81225,
  https://tracker.moodle.org/browse/MDL-76583

---

## core_filters

### TeX/Algebra images in File Storage

- **Kind:** added. **Severity:** info.
- `filter_tex` and `filter_algebra` store rendered images in file storage
  (system context, component `filter_tex`/`filter_algebra`, filearea
  `rendered_images`, itemid 0) with a `rendered_images` cache definition, not
  in `$CFG->dataroot/filter/{tex,algebra}/`. An upgrade step migrates and
  deletes the old directories.
- **Action:** scripts reading or cleaning those dataroot directories must
  switch to file storage.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87554

---

## core_form

### `duration` element enforces restricted units

- **Kind:** changed. **Severity:** breaking.
- When `units` is given, the effective `defaultunit` (yours, or `MINSECS`)
  must be one of them, otherwise a `coding_exception` is thrown.

```php
$mform->addElement('duration', 'timelimit', get_string('timelimit', 'quiz'),
    ['units' => [HOURSECS, DAYSECS], 'defaultunit' => HOURSECS]);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89434

---

## core_grades

### Scale-less outcomes linked to course modules

- **Kind:** added. **Severity:** new.
- `grade_outcome` gains `add_outcome_to_module(int $courseid, int $cmid): bool`,
  `remove_outcome_from_module(int $courseid, int $cmid): void`, static
  `get_outcomes_in_module(int $cmid, int $courseid): array` and
  `get_used_outcomes_in_course(int $courseid): array`.
- **Action:** don't assume an outcome has a `scaleid`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88881

### Grade action bar templates moved to core

- **Kind:** changed. **Severity:** action.
- `general_action_bar` renders `core/navigation_action_bar` (was
  `core_grades/general_action_bar`); the grader report action bar renders
  `core/action_bar` (was `gradereport_grader/action_bar`). Old templates are
  deprecated (final removal planned for 6.0).
- **Action:** move theme overrides to the core templates.
- **Tracker:** https://tracker.moodle.org/browse/MDL-81096

### Courses with penalised grades frozen on upgrade

- **Kind:** changed. **Severity:** info.
- Courses containing a grade with a deducted penalty are frozen on upgrade so
  a regrade does not change grades; a user with `moodle/grade:manage` chooses
  to keep them or apply the fix (which restores affected raw grades from the
  assignment and regrades the course). Review UI flow **unverified**.
- **Action:** warn admins; plugins applying penalties re-test grading.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89497,
  https://tracker.moodle.org/browse/MDL-88407

### `grade_item::update_deducted_mark()` deprecated

- **Kind:** deprecated. **Severity:** action.
- No replacement; `\core_grades\penalty_manager` applies penalties via
  `grade_item::adjust_raw_grade()`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88407,
  https://tracker.moodle.org/browse/MDL-88663

---

## core_question

### `bulk_action_base::get_action_icon()`

- **Kind:** added. **Severity:** action.
- Returns the pix identifier for the bulk action button in the sticky footer.
  The notes say it must be overridden; the base class returns `i/empty`, so
  without an override the button has an empty icon.
- The class is `\core_question\local\bank\bulk_action_base` (the notes'
  `\core\question\...` namespace is wrong). Core example:
  `\qbank_bulkmove\bulk_move_action::get_action_icon()` returns `i/move_2d`.

```php
public function get_action_icon(): string {
    return 't/move';
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-73051

---

## core_reportbuilder

### `week`, `month`, `year` aggregations and `datebase`

- **Kind:** added. **Severity:** new.
- New aggregations for `TYPE_TIMESTAMP` columns under
  `core_reportbuilder\local\aggregation\`, extending the new abstract
  `datebase` (which `date` now also extends).
- **Action:** custom date aggregations extend `datebase`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-84635

### `prepend_join()` / `prepend_joins()`

- **Kind:** added. **Severity:** new.
- New final methods on the join trait; the base entity prepends its own joins
  to every column, filter and condition it adds.
- **Action:** entities can drop `->add_joins($this->get_joins())` boilerplate.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87405

### `column::add_header_attributes()`

- **Kind:** added. **Severity:** new. Also in 5.2.2+.
- `add_header_attributes(array $attributes): self` for column header HTML
  attributes.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89384

### Columns sortable by default

- **Kind:** changed. **Severity:** breaking.
- `$issortable` now defaults to true.
- **Action:** add `->set_is_sortable(false)` where needed; remove redundant
  `->set_is_sortable(true)`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87404

### `set_main_table_sql()`; `set_main_table()` alias mandatory

- **Kind:** changed. **Severity:** breaking.
- New final `set_main_table_sql(string $tablesql, string $tablealias)` for
  complex SQL as the main table. `set_main_table(string $tablename, string $tablealias)`
  no longer defaults the alias and delegates to the SQL variant.

```php
$this->set_main_table('user', 'u');
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88397

### `column::add_fields()` accepts an array

- **Kind:** changed. **Severity:** new.
- `add_fields(string|array $sql, array $params = [])`. (The notes say "report
  class"; the method is on the column class.)
- **Tracker:** https://tracker.moodle.org/browse/MDL-89004

### `get_main_table()` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `get_main_table_sql()` (with `get_main_table_alias()`); the value
  includes `{braces}` or complex SQL. The deprecated method strips outer
  braces for compatibility.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88397

---

## core_user

### `user_convert_text_to_menu_items()` typed return

- **Kind:** changed. **Severity:** action.
- The notes say it now returns `\core_user\output\user_action_menu\base`
  items. In the beta code the function is a deprecated wrapper (see
  MDL-82650 above) for `\core\user::convert_text_to_menu_items(string $text)`,
  which returns `stdClass` items with `itemtype`/`title`/`url`. The typed item
  classes (`base`, `divider`, `header`, `link`, `text`) do exist.
- **Action:** don't call the deprecated function; re-check the return type
  at 5.3.0 before relying on typed items.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88938,
  https://tracker.moodle.org/browse/MDL-82650

### `extend_user_menu`: `add_navitem()` → `add_menu_item()`

- **Kind:** deprecated. **Severity:** action.
- Callbacks for `\core_user\hook\extend_user_menu` should pass
  `\core_user\output\user_action_menu\` `link`, `divider`, `header` or `text`
  objects to `add_menu_item()`. `add_navitem()` and `get_navitems()`
  (→ `get_menu_items()`) are deprecated. Constructors:
  `link(url $url, string $title, ?string $titleattribute = null, ?pix_icon $pixicon = null, ?string $imgsrc = null, array $classes = [], array $attributes = [])`,
  `header(string $title, array $classes = [], array $attributes = [])`,
  `text(string $content, array $classes = [], array $attributes = [])`,
  `divider(array $classes = [], array $attributes = [])` (inherited from
  abstract `base`).

```php
$hook->add_menu_item(new \core_user\output\user_action_menu\link(
    new \core\url('/local/x/index.php'), get_string('x', 'local_x')));
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88938

---

## core_webservice

### `allowcorsrequests` in `db/services.php`

- **Kind:** added. **Severity:** new.
- Default false. `lib/ajax/service.php` sends
  `Access-Control-Allow-Origin: *` only for no-login AJAX requests where every
  function in the batch has `allowcorsrequests` and `loginrequired => false`.
  Core uses it for the no-login auth functions
  (`core_auth_request_password_reset`, `core_auth_is_minor`,
  `core_auth_is_age_digital_consent_verification_enabled`,
  `core_auth_resend_confirmation_email`, `auth_email_get_signup_settings`,
  `auth_email_signup_user`) and `tool_mobile_get_public_config` /
  `tool_mobile_get_plugins_supporting_mobile` /
  `tool_mobile_get_tokens_for_qr_login`.

```php
'local_myplugin_get_public_info' => [
    'classname' => \local_myplugin\external\get_public_info::class,
    'type' => 'read',
    'ajax' => true,
    'loginrequired' => false,
    'allowcorsrequests' => true,
],
```

- **Action:** only for functions safe to call cross-origin.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87150

---

## aiprovider_gemini

### `gemini31flashimage` model, default for Generate image

- **Kind:** added. **Severity:** info.
- New model class for `gemini-3.1-flash-image`, now the default for the
  generate image action, replacing retiring Imagen 4 endpoints.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89431

### `process_generate_image` branches on endpoint method

- **Kind:** changed. **Severity:** info.
- Endpoints ending `:generateContent` use the Gemini native image protocol;
  others use the Imagen `:predict` protocol. Decided by URL, so custom models
  need the right suffix.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89431

---

## assignfeedback_editpdf

### Windows multi-page PDF conversion fixed

- **Kind:** fixed. **Severity:** info.
- Ghostscript's `%d` page placeholder is now inside the shell-quoted output
  filename, so it survives quoting on Windows.
- **Tracker:** https://tracker.moodle.org/browse/MDL-76966

---

## block_myoverview

### Exporter strings use numeric HTML entities

- **Kind:** changed. **Severity:** action. Also in 5.2.2+.
- Listed under block_myoverview, but the change is in `core\external\exporter`:
  formatted non-HTML strings pass through `clean_string()`, so single-valued
  PARAM_TEXT exporter properties return numeric entities ('multiple'
  properties are not passed through `clean_string()`) (`&#60;`, `&#62;`, `&#34;`, `&#39;`, `&#38;`).
- **Action:** update PHPUnit/Behat assertions and JS comparisons (e.g.
  `&amp;` → `&#38;`).
- **Tracker:** https://tracker.moodle.org/browse/MDL-79755

---

## block_timeline

### Rewritten in ESM + React

- **Kind:** added. **Severity:** breaking for overrides.
- The block mounts `@moodle/lms/block_timeline/Timeline` via
  `html_writer::react_component()`; legacy AMD modules and templates are gone.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88287

### `output\main` and `output\renderer` removed

- **Kind:** removed. **Severity:** breaking.
- Removed without deprecation stubs. (The notes also mention external service
  classes; none could be found in 5.2.2+ or 5.3beta — **unverified**.)
- **Action:** remove `block_timeline_renderer` overrides and uses of
  `block_timeline\output\main`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88287

---

## core\task\adhoc_task

### Soft retry delay

- **Kind:** added. **Severity:** new. Also in 5.2.2+.
- `set_soft_retry_delay(?int $softretrydelay = null)`,
  `get_soft_retry_delay(): ?int`, `is_adhoc_task_delayed(): bool`. Call inside
  `execute()` to be rescheduled without counting as a failure: null =
  exponential backoff, positive = seconds, `<= 0` throws. Attempts are still
  decremented.

```php
public function execute() {
    if (!$this->remote_ready()) {
        $this->set_soft_retry_delay(300);
        return;
    }
    // ...
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-79763

---

## core\task\manager

### `manager::adhoc_task_delayed()`

- **Kind:** added. **Severity:** info. Also in 5.2.2+.
- Reschedules a soft-retried task. Backoff is capped at 24 hours; the notes
  describe it as based on elapsed time, but the code bases it on the remaining
  attempt count. Cron calls it; plugins use `set_soft_retry_delay()`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-79763

---

## editor_tiny

### TinyMCE follows the page colour mode

- **Kind:** added. **Severity:** action.
- The skin is chosen from `data-bs-theme` at editor setup (no live switch
  until reload). The attribute is copied to the editor container, the sink and
  the content iframe's `<html>`, so `content_css` can respond. Moodle toolbar
  icons (`svg[data-buttonsource="moodle"]`) are lightened by a CSS filter in
  dark mode.
- **Action:** TinyMCE plugins use theme colour tokens; draw monochrome icons
  dark so the filter can lighten them.
- **Tracker:** https://tracker.moodle.org/browse/MDL-68037

### Accordion and advlist enabled; `<details>`/`<summary>` allowed

- **Kind:** added. **Severity:** info.
- Bundled TinyMCE `accordion` and `advlist` plugins enabled by default;
  HTMLPurifier allows `<details>`, `<summary>` and `lower-greek` list style.
- **Action:** HTML post-processing should expect these elements.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88618

---

## enrol_manual

### `enrol_manual_plugin::enrol_cohort()` deprecated

- **Kind:** deprecated. **Severity:** action.
- No replacement (unused; lacked group support). Enrol members yourself or
  use enrol_cohort.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89439

---

## gradereport_user

### `gradereport_user_get_grade_items` returns `parentcategoryid`

- **Kind:** changed. **Severity:** info.
- Optional `parentcategoryid` (int) for category grade items. Additive.
- **Tracker:** https://tracker.moodle.org/browse/MDL-64304

---

## mod_assign

### `\mod_assign\override_manager` and override web services

- **Kind:** added. **Severity:** new.
- Override logic moved into `\mod_assign\override_manager`
  (`delete_all_overrides()`, `delete_overrides_by_id()`,
  `move_group_override()`, `reorder_group_overrides()`, …). New external
  functions `mod_assign_save_overrides`, `mod_assign_get_overrides`,
  `mod_assign_delete_overrides`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-86513

### `assign::calculate_penalised_grade()` applies grade-item scaling

- **Kind:** changed. **Severity:** action.
- `calculate_penalised_grade(stdClass $grade, ?\grade_grade $usergraderecord = null): array`
  applies grade-item factors so the result matches the gradebook final grade;
  pass the `grade_grade` record to save queries. Already in 5.2.2+; 5.3 adds
  skipping the scaling for courses frozen on the legacy penalty calculation.
- **Action:** remove your own scaling of the result.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88407

### Feedback batch operations: `confirmationyes`

- **Kind:** changed. **Severity:** new. Also in 5.2.2+.
- `get_grading_batch_operation_details()` may return `confirmationyes` for the
  confirm button text (default `savechanges`).
- **Tracker:** https://tracker.moodle.org/browse/MDL-88688

### Override methods deprecated

- **Kind:** deprecated. **Severity:** action.
- `assign::delete_override()`, `assign::delete_all_overrides()`, and global
  `move_group_override()`, `reorder_group_overrides()`. Replacements on
  `override_manager`: `delete_overrides_by_id(array $ids, ...)` (the notes say
  `delete_override`, which does not exist), `delete_all_overrides()`,
  `move_group_override()`, `reorder_group_overrides()`. Removal planned for
  6.0 (MDL-87324).
- **Tracker:** https://tracker.moodle.org/browse/MDL-86513

### `ASSIGN_MULTIMARKING_MAX_MARKERS` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `ASSIGN_MULTIMARKING_DEFAULT_MAX_MARKERS`. Removal planned for 7.0.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87709,
  https://tracker.moodle.org/browse/MDL-89401

### `get_allocated_markers()` / `update_allocated_markers()` deprecated

- **Kind:** deprecated. **Severity:** action.
- Use `get_marker_allocations(int $studentid, bool $includeplaceholders = true)`
  and `update_marker_allocations(int $studentid, array $modifiedallocations)` —
  the second argument has a different shape.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87709

### `mod_assign\event\marker_updated` no longer triggered

- **Kind:** deprecated. **Severity:** breaking for observers.
- Fired only by deprecated code. Observe `mod_assign\event\marker_added` and
  `mod_assign\event\marker_removed`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87709

---

## mod_forum

### `discussion::get_discussion_navigation_buttons()`

- **Kind:** added. **Severity:** info.
- Renderer method returning data for the discussion navigation template.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88602

### `mod_forum_set_read_state` web service

- **Kind:** added (listed under Changed). **Severity:** new.
- Marks individual posts read/unread when manual read tracking is on.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87887

---

## mod_quiz

### `mod_quiz_get_users_in_report` web service

- **Kind:** added. **Severity:** new.
- Returns the users in a quiz report for the AJAX user search; it calls the
  report's `has_permission()` and `setup_report_data()`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-81096

### Report navigation bar hooks

- **Kind:** added. **Severity:** new.
- `report_base` gains
  `print_action_bar(string $reportmode, ?attempts_report_options $options = null, ?\cm_info $cm = null, ?\moodle_url $url = null): void`
  (report selector, group selector, user search, initials bar),
  `setup_report_data(stdClass $quiz, \cm_info $cm, stdClass $course, ?context $context): array`
  and `has_permission(context $context): void`. JS: `mod_quiz/searchwidget/user`.
  The base `setup_report_data()` has no default for `$context`; core
  overrides use `= null`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-81096

### `duedate` on `quiz` and `quiz_overrides`

- **Kind:** added. **Severity:** info.
- `quiz.duedate` (int, default 0); `quiz_overrides.duedate` (nullable; null =
  quiz default).
- **Action:** code copying, backing up or reporting quiz/override records
  accounts for it.
- **Tracker:** https://tracker.moodle.org/browse/MDL-82521

### Custom quiz reports must print the action bar

- **Kind:** changed. **Severity:** breaking.
- The group selector is no longer printed by the standard header code.
  Minimal fix: call `print_action_bar()`. If you call
  `print_standard_header_and_messages()` without implementing
  `setup_report_data()`, pass `null` as the options argument or the AJAX user
  search fails. Full adoption: override `setup_report_data()` returning
  `[$options, $table, $allowedjoins]`, and move custom capability checks from
  `display()` into `has_permission()` (the web service calls it).

```php
$this->print_action_bar('myreport', null, $cm, $reporturl);
```

```php
public function has_permission(context $context): void {
    require_capability('quiz/myreport:view', $context);
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-81096

---

## mod_workshop

### Portfolio Behat step deprecated

- **Kind:** deprecated. **Severity:** action.
- `I set portfolio instance "X" to "Y"` → `I set the portfolio instance "X" to "Y"`
  (now in `behat_portfolio`). Removal planned for 7.0.

```gherkin
And I set the portfolio instance "File download" to "Enabled and visible"
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89069

---

## theme

### Classic theme removed

- **Kind:** removed. **Severity:** breaking.
- If Classic's files are absent at upgrade and Classic is installed, Classic
  is uninstalled (`lib/db/upgrade.php` step 2026080700.01). Settings are
  migrated to Boost only when Classic was the site default theme
  (`upgrade_migrate_classic_theme_to_boost()` returns early otherwise). Only
  customised values are copied: unaddableblocks (merged with Boost's
  navigation, settings, course_list), brandcolor, scsspre, scss,
  backgroundimage and loginbackgroundimage. Uninstalling resets course,
  category, cohort and user theme selections. Presets and `navbardark` are not migrated;
  side-post block positions are not kept. A forced `$CFG->theme = 'classic'`
  must be changed by hand. Installing Classic separately before the upgrade
  skips migration.
- **Action:** re-parent Classic child themes or install Classic from its
  separate repository first; review migrated SCSS that uses Classic variables.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88351

---

## theme_boost

### Light/dark colour modes — `theme_boost\colour_mode`

- **Kind:** added. **Severity:** action.
- Built on Bootstrap 5.3 colour modes; mode written to `data-bs-theme`.
  Entry point `theme_boost\colour_mode` (`is_enabled()`, `get_current_mode()`,
  `render_menu(\renderer_base $output)`). Off unless the experimental
  `theme_boost/enablecolourmodes` is on. SCSS uses custom properties for
  grays, white, black, body background/colour; `$card-bg`,
  `$card-border-color`, `$state-*-bg/border` default to custom properties; the
  dark palette `scss/moodle/dark.scss` must stay the last import. Mode stored
  as a user preference and mirrored in cookie `theme_boost_colourmode`
  (`light|dark|auto`; written by `theme_boost/colourmode` JS with attributes
  from `colour_mode::get_cookie_attributes()`).
- **Action:** child themes check presets that override card/state variables,
  use custom properties, output `colour_mode::render_menu()` in custom
  navbars, keep `dark.scss` last; document the cookie in the privacy/cookie
  notice.
- **Tracker:** https://tracker.moodle.org/browse/MDL-68037

### Noto Sans cyrillic subsets

- **Kind:** added. **Severity:** info.
- `cyrillic` and `cyrillic-ext` subsets (normal/italic, weights 100–900) bundled.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89024

### Noto Sans JP for `:lang(ja)`

- **Kind:** added. **Severity:** info.
- Applied only to content in `lang="ja"`, so it is served only for Japanese.

```scss
:lang(ja):not(.fa, .fas, .far, .fab, [class*="fa-"]) {
    font-family: $font-family-sans-serif-jp;
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89024

### Default typeface is self-hosted Noto Sans

- **Kind:** changed. **Severity:** action.
- Replaces the system-ui stack; `@font-face` in `scss/moodle/fonts.scss`;
  `$font-family-sans-serif` built from the `$mds-font-family-base` token plus
  fallbacks (`!default`).
- **Action:** child themes wanting system fonts override
  `$font-family-sans-serif`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88412

### `drawer` template: `drawerheadercontent` → `drawercontrols`

- **Kind:** changed. **Severity:** breaking for overrides.
- The course index drawer has a single collapse/expand-all toggle
  (`theme_boost/courseindexdrawercontrols` template rewritten; the
  `{{$drawerheadercontent}}` block in `theme_boost/drawer` is replaced by
  `{{$drawercontrols}}`).

```mustache
{{$drawercontrols}}{{/drawercontrols}}
```

- **Action:** re-diff overrides of `courseindexdrawercontrols.mustache`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89050

### `core/loginform` moved from Boost to core

- **Kind:** changed. **Severity:** action.
- The Boost version now lives in `lib/templates/loginform.mustache`.
- **Action:** re-diff theme overrides against the new core template.
- **Tracker:** https://tracker.moodle.org/browse/MDL-89196

### Don't import `theme_boost/bootstrap/*`

- **Kind:** deprecated. **Severity:** action.
- `theme_boost/amd/src/bootstrap/` is gone; old module ids survive only as
  import-map aliases until 7.0. Import from the Bootstrap package.

```js
// Moodle 5.2 and earlier (supported until 7.0):
import {Tooltip} from 'theme_boost/index';
// Moodle 5.3 and later:
import {Tooltip} from 'bootstrap';
// Private dom/util helpers, 5.3 and later (not public Bootstrap API):
import EventHandler from 'bootstrap/dom/event-handler';
```

- **Action:** replace `theme_boost/bootstrap/<x>` imports; rebuild AMD.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88766

---

## tiny_premium

### TinyMCE Premium Markdown plugin

- **Kind:** added. **Severity:** info.
- Support for the Premium `markdown` plugin with capability
  `tiny/premium:usemarkdown`.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88547

---

## tiny_recordrtc

### Recordings can always be downloaded

- **Kind:** added. **Severity:** info.
- A download button in the recorder.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88603

### Too-large recordings offer download

- **Kind:** changed. **Severity:** info.
- Users are offered a download when a recording is too large to upload
  (size-limit path **unverified**).
- **Tracker:** https://tracker.moodle.org/browse/MDL-88603

### Duration metadata created on stop

- **Kind:** fixed. **Severity:** info.
- Duration metadata is generated when recording stops, for an accurate
  duration.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88603

---

## tool_mobile

### `get_subscription_information()` gains `&$errormessage`

- **Kind:** changed. **Severity:** new.
- `\tool_mobile\api::get_subscription_information($forcecache = false, $ignorecache = false, $timeout = 10, &$errormessage = '')`
  — receives an error description when contacting the Apps Portal fails.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88458

---

## tool_task

### `\core\task\manager::task_is_scheduled()` deprecated

- **Kind:** deprecated. **Severity:** info.
- Use `get_queued_adhoc_task_record($task, false)` (the old wrapper passed
  `includefailed = false`). The method is `protected`, so this only affects
  code subclassing `\core\task\manager`; the deprecation is not final.

```php
$exists = false !== \core\task\manager::get_queued_adhoc_task_record($task, false);
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-86422

### `queue_adhoc_task()` returns a task id

- **Kind:** fixed. **Severity:** action.
- Returns `int|false`: the new task's id, or with `$checkforexisting` the id
  of the matching existing task (matched by a new identity hash; exhausted
  tasks are revived), or false if the component is deprecated or a DB error
  occurred. In 5.2 a duplicate returned false.
- **Action:** stop treating a truthy result as "newly queued".
- **Tracker:** https://tracker.moodle.org/browse/MDL-86422

---

## Not in the 5.3beta upgrade notes

Developer-relevant 5.3 changes found in the code, environment file or tracker
but absent from the beta `UPGRADING.md`.

### Version identifiers

- 5.3beta: `$version = 2026091600.00`, `$release = '5.3beta (Build: 20260916)'`,
  `$branch = '503'`. Re-check the number at 5.3.0 before using it in
  `$plugin->requires`.

### Server requirements

- `admin/environment.xml` 5.3 block: PHP 8.3.0; MariaDB 11.4 (was 10.11),
  MySQL 8.4, PostgreSQL 17 (was 16), SQL Server 15.0; upgrade requires 4.4;
  PHP extension list unchanged from 5.2. The moodledev.io 5.3 release page
  showed older values at the time of the beta; the environment file is
  authoritative.
- **Action:** drop MariaDB < 11.4 and PostgreSQL < 17 from 5.3 CI jobs.
- **Tracker:** https://tracker.moodle.org/browse/MDL-86887

### Linear navigation on by default (per-format site setting)

- `\core_courseformat\local\linearnavigationsettings::is_linear_navigation_enabled()`
  reads the site config `format_<name>/enablelinearnav`. The earlier
  `enablelinearnav` course format option was removed (it never shipped in a
  stable release) and an upgrade step deletes its rows. A format that returns
  true from `uses_linear_navigation()` but defines no such setting is treated
  as **enabled**.
- **Action:** add the admin setting if your format needs an off switch;
  activity modules rendering their own prev/next controls check
  `is_linear_navigation_enabled()`.

```php
// course/format/myformat/settings.php
// Same shape as format_topics.
$settings->add(new admin_setting_configselect(
    'format_myformat/enablelinearnav',
    new lang_string('linearnavigationsettings', 'core_courseformat'),
    new lang_string('linearnavigationsettings_help', 'core_courseformat'),
    1,
    [1 => get_string('yes'), 0 => get_string('no')],
));
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89406,
  https://tracker.moodle.org/browse/MDL-87302,
  https://tracker.moodle.org/browse/MDL-84921

### OAuth2 scopes for REST API routes

- Scope classes extend `\core\router\scope\abstract_scope` with class
  attributes `identifier_attribute`, `summary_attribute`,
  `description_attribute`, and live in each component's
  `classes/route/scope/` (auto-discovered, cached). Route methods declare
  required scopes with repeatable `#[\core\router\scope\scopeset(...)]` (AND
  within a set, OR across sets) or opt out with
  `#[\core\router\scope\unscoped_resource]`. Tokens lacking a satisfying set
  are rejected; a set naming an unknown scope fails safe. A route with neither
  attribute has **no** scope restriction at runtime — declaring scopes is a
  convention enforced by tests, not by the router. `route_testcase` gains
  `assert_route_is_scoped()`, `assert_route_is_unscoped()`,
  `assert_route_required_scopes()`.

```php
namespace local_myplugin\route\scope;

use core\router\scope\abstract_scope;
use core\router\scope\description_attribute;
use core\router\scope\identifier_attribute;
use core\router\scope\summary_attribute;

#[identifier_attribute('read')]
#[summary_attribute('myplugin_read_scope_summary', 'local_myplugin')]
#[description_attribute('myplugin_read_scope_desc', 'local_myplugin')]
class read extends abstract_scope {
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-89089,
  https://tracker.moodle.org/browse/MDL-89710

### Composer runtime status API

- `\core\composer` (via DI): `is_installed()`, `get_status()`,
  `get_package_status(string $package)` (`->installed`). Core's
  `composer.json` requires packages that are not bundled (e.g.
  `league/oauth2-server`); OAuth2 server features degrade with a warning until
  `composer install` has been run.

```php
$composer = \core\di::get(\core\composer::class);
if (!$composer->get_package_status('league/oauth2-server')->installed) {
    // Degrade gracefully.
}
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88576

### PHPUnit site upgrade, snapshot and restore

- `admin/tool/phpunit/cli/util.php` gains `--upgrade`, `--snapshot[=NAME]`,
  `--snapshot --list` and `--restore=NAME`, so CI can restore a cached core
  install and just upgrade after adding a plugin.

```sh
php public/admin/tool/phpunit/cli/util.php --snapshot=core
php public/admin/tool/phpunit/cli/util.php --snapshot --list
# --restore needs the exact name printed by --list, not the short name.
php public/admin/tool/phpunit/cli/util.php --restore=<name from --list>
php public/admin/tool/phpunit/cli/util.php --upgrade
```

- **Tracker:** https://tracker.moodle.org/browse/MDL-88495

### `tool_mobile/enabledeeplinkautologin` (default off)

- New setting "Enable auto-login in deep links", exposed in
  `tool_mobile_get_public_config`. The app only auto-logs-in from deep links
  carrying `token`/`privatetoken` when it is enabled.
- **Action:** integrations generating such links tell admins to enable it.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88924

### Inline navbar search in Boost

- `theme_boost\output\core_renderer::search_box()` renders the new
  `core/search_input_navbar_inline` template (always-visible form, all
  widths); the navbar divider was removed.
- **Action:** re-check theme overrides of `search_box()`/`navbar.mustache`
  and Behat that clicked the search toggle.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87834,
  https://tracker.moodle.org/browse/MDL-89010

### Outline notification and message icons

- Icon map: `core:i/notifications` → `fa-regular fa-bell`;
  `core:t/message` → `fa-regular fa-message`.
- **Action:** themes restyling these icons verify appearance.
- **Tracker:** https://tracker.moodle.org/browse/MDL-87836

### Centred layout; drawers below the navbar

- On large screens drawers anchor to `.main-inner` rather than the viewport;
  drawers no longer overlay the navbar, keyboard users can tab back to it, and
  the drawer closes on focus loss.
- **Action:** child themes with custom drawer/layout SCSS or JS re-test.
- **Tracker:** https://tracker.moodle.org/browse/MDL-88411,
  https://tracker.moodle.org/browse/MDL-89076

### New Site administration and profile links

- Site administration links to the site content bank, site participants and
  user notes; profile Miscellaneous section gains a Tags link. Behat features
  that used the navigation block to reach these pages can be simplified
  (**unverified** in detail).
- **Tracker:** https://tracker.moodle.org/browse/MDL-89309,
  https://tracker.moodle.org/browse/MDL-88655,
  https://tracker.moodle.org/browse/MDL-88687
