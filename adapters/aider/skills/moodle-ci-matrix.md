# moodle-ci-matrix

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
