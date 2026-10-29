---
name: moodle-release-preflight
description: Use when about to release a Moodle plugin, submit it to the Moodle Plugins directory, or send it for an external security review, or when reviewing security-relevant changes before a tag. A pre-release checklist of defect classes that real Marketplace and security reviews repeatedly caught in shipped plugins despite green CI — client-trusted ids, client-only limits, inline-served uploads, unescaped output sinks, unclean restore, privacy provider drift, incomplete course reset/uninstall, vacuous tests, minor gates, an extended checklist (SQL from mobile args, unserialize, IDOR, CSRF, noclean, SSRF, group leaks), and authored definitions or imports that reach other users (settings read by name, drafts). Skip when writing new feature code or doing a general security audit (use moodle-security-audit).
---

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
  handlers, uploads, restore, or stored data. Classes **1, 2, 3, 5, 7, 11 are
  typically N/A**; focus on **4, 6, 8, 9**. Prove N/A with a grep (no
  `required_param`, `send_stored_file`, or `backup/` dir) — don't assume.
- **`mod_`, `assignsubmission_`/`assignfeedback_`, anything with an upload
  endpoint, external functions, or its own tables** — run **all nine**, then
  classes 10 and 11.
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

### 1c. Building a core form's data in code skips its freezes

**Seen in reviews:** a plugin read a core form's defaults, overlaid its own values and saved
without posting the form. Core had `hardFreeze()`d a capability-gated field for that user
(`course_edit_form` freezes `downloadcontent` and `visible`), and the overlay set it anyway.

Check: wherever code fills a core form's data itself (`_defaultValues`, `get_data()` from a
mocked submit, an array passed straight to `create_course()`), does it skip frozen elements and
take the pinned value from `_constantValues`? `create_course()` and similar APIs check no
capability of their own.

Fix: skip any element where `$mform->getElement($name)->isFrozen()`, and use core's constant
for it. Test with a user who lacks the gating capability.

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
- `grep -rnE "send_(stored_)?file\(" --include='*.php' .` — both serve files inline unless told
  otherwise (`send_file()` too: a grep for `send_stored_file` alone misses it). The 4th argument
  of `send_stored_file()` is `$forcedownload`. For student files it must be true unless the MIME type is a
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
- **Core helpers that print their text as HTML:** `html_writer::select()` option labels
  (`select_option()` puts the label in tag content), and formslib element labels, `select`
  option texts and `advcheckbox` texts (core's form templates print `{{{label}}}`,
  `{{{text}}}` and `{{{.}}}` raw). Stored or authored text passed to any of them needs `s()`.

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
- **A `LICENSE` file at the repository root**, holding the full text of the licence the plugin declares
  (GPL-3.0-or-later for a Moodle plugin: the GNU GPL version 3 text). It must be tracked and must ship in
  the distribution ZIP, so check all three:

  ```bash
  git ls-files --error-unmatch LICENSE                  # tracked (a README link or LICENSE.md alone is not enough)
  grep -q 'Version 3, 29 June 2007' LICENSE              # the GPLv3 text, matching the file headers and composer.json
  git check-attr export-ignore -- LICENSE                # must NOT say "set": export-ignore drops it from the ZIP
  ```

  Fail it if the file is missing, named differently only (`LICENSE.md`, `COPYING`), untracked, a stub
  pointing elsewhere, a different licence from the one the headers declare, or export-ignored. Third-party
  notices (a `LICENSES/` folder, `thirdpartylibs.xml` entries) are in addition to it, not instead.
- **Only the English language pack ships.** A Plugins directory (Marketplace) release carries `lang/en/`
  only; other languages are contributed through AMOS and arrive with Moodle's language packs. Keep
  translations (`lang/ja/` and the like) out of the repository, e.g. in the developer's own notes, and
  check both the working tree and the release tag:

  ```bash
  git ls-files lang | grep -v '^lang/en/'                         # must print nothing
  git ls-tree -r --name-only vX.Y.Z -- lang | grep -v '^lang/en/'   # the tag about to be submitted
  ```

  Fail it on any other language folder. An `export-ignore` line does not count as a pass: the tag
  itself must not carry it, since some installers fetch the tagged tree rather than the ZIP.
- **Test helpers in production code.** `moodleform::mock_submit()` replaces `$_POST` and `$_FILES`
  (and `$_GET` for a GET form); code that uses it outside tests saves and restores all of them
  in a `finally`. Reflection into protected core members needs a test that fails when they change.
- **Admin-facing messages in literal English** (validation problems, import reports): reviews flag
  them. Use language strings with placeholders. Two messages may share a string only when its
  `{$a}` means the same thing in both.
- Timezones: passing another user's raw `timezone` field (the `99` "server
  default" sentinel) to `get_user_timezone()` or `core_date::get_user_timezone()`
  resolves to the *viewer's* zone — use `core_date::get_server_timezone()` for
  that case. JS wall-clock-to-timestamp by diffing offsets is off
  by the DST amount inside a transition window.

## 10. Extended security checklist

**Seen in a wider audit:** classes 1–9 come from reviews of a small set of
plugins. A static audit of several dozen widely installed third-party plugins
found more: SQL from mobile `$args`, `unserialize()` of restored data, IDOR and
client-supplied `userid`, GET-without-sesskey writes, `noclean` on student or LLM
text, SSRF through PDF renderers, and SEPARATEGROUPS leaks.

Check: for any plugin with request entry points, walk the checklist at the end
of [reference.md](reference.md) and report each line as pass / fail / N/A.
Do it after classes 1–9; where the two overlap, report the finding once.

## 11. Authored definitions and imports that reach other users

**Seen in reviews:** a plugin let admins write content definitions (questions, defaults, labels)
that teachers then used. A default could name a site setting, `"config:<plugin>/<name>"`, which
the plugin resolved with `get_config()` for **any** name, so `config:core/dbpass` came back filled
into a form field. An import in "replace" mode kept the replaced item enabled, so an untrusted file
went live at once.

Check, for any plugin whose stored definitions, templates or imports are shown to other users:
- **Settings read by name:** if data can name a setting, is the name limited to an allowlist of
  what it is for (e.g. course and activity defaults)? Never values from `config.php`
  (`$CFG->config_php_settings`, `$CFG->forced_plugin_settings`) and never names like
  pass|secret|key|token|salt. Is the same rule applied at **every** entry path: editor, JSON
  editor, import, AI proposal, and the runtime resolver for rows stored before the rule?
- **Imports land as drafts:** new, copied **and replaced** items wait for a review step before
  other users see them.
- **The capability that authors them** carries `RISK_XSS` (as well as `RISK_CONFIG` if it
  changes site behaviour), because what it writes is rendered for others.

Fix: one allow-rule shared by the validator and the resolver, tested with a forbidden name in a
stored row; set the draft status explicitly in every import branch.

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
| No root `LICENSE`, or only `LICENSE.md`/`COPYING`, or export-ignored | The GPL text must ship with the code | Track the full GPLv3 text as `LICENSE`; keep it out of export-ignore |
| `lang/ja/` (or any non-English pack) in the release | The Plugins directory takes `lang/en` only; translations go through AMOS | Keep only `lang/en/` in the repo and the tag |
| `s()` left off a formslib label or `html_writer::select()` option | Core prints both as HTML | Escape authored or stored text before passing it |
| Data can name any setting for `get_config()` | `config:core/dbpass` reaches a form field | Allowlist by purpose; refuse config.php and credential names everywhere |
| A replace-import keeps the item live | Untrusted content reaches users without review | Every import lands as a draft |
| Overlaying values on a core form's defaults | Frozen, capability-gated fields get set | Skip frozen elements; use core's constant |

## Report format

A short table: class → pass / fail / N/A → evidence, with class 10 as one row
per reference.md checklist group, and class 11 as one row. List each fail with file:line and the fix
pattern. Do not fix unless asked.

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
