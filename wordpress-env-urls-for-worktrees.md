# WordPress Env URLs for Worktrees

For Bleech projects using Flynt, make `.env`/`LOCAL_URL` the single source of truth for the local domain. This simplifies working with worktrees and works around issues with Git, Vite, and DB cloning:

- Each worktree runs on its own URL, for example `http://worktree.example-website.localhost`.
- Per-worktree URLs live in `.env`, not as local edits to `wp-config.php`.
- `wp-config.php`, `./run`, WP-CLI search-replace, and Flynt’s Vite host config all use `.env`/`LOCAL_URL`.

> To apply this setup with a coding agent, use the Bleech
> `localhost-mamp-worktrees` skill:
> https://github.com/bleech/skills/tree/main/localhost-mamp-worktrees

## Project changes

### 1. Add root `.env` loading

If the project does not already load a root `.env`, add dotenv to the root Composer project:

```sh
composer require vlucas/phpdotenv
```

Commit `composer.json` and `composer.lock`.

### 2. Keep `.env` out of git

Add this to `.gitignore`:

```gitignore
/.env
```

### 3. Use `LOCAL_URL` in `wp-config.php`

Load the root `.env` before `WP_HOME` / `WP_SITEURL` are defined:

```php
require_once __DIR__ . '/vendor/autoload.php';
Dotenv\Dotenv::createUnsafeImmutable(__DIR__, '.env')->safeLoad();

$localUrl = getenv('LOCAL_URL') ?: 'http://[PROJECT].localhost';

define('WP_HOME', $localUrl);
define('WP_SITEURL', $localUrl);
```

Keep existing project-specific config as-is; the important part is that `wp-config.php` contains the committed env-based logic, while each worktree keeps its actual URL in an untracked `.env`.

### 4. Export `LOCAL_URL` in `./run`

Near the top of `./run`, load `.env` and export `LOCAL_URL` so WP-CLI, Composer/NPM hooks, and Vite inherit the same value:

```bash
set -a
[ -f .env ] && . ./.env
set +a

LOCAL_URL="${LOCAL_URL:-http://$PROJECT_NAME.localhost}"
export LOCAL_URL
```

### 5. Use `LOCAL_URL` in Flynt/Vite

In the theme `vite.config.js`, pass the worktree URL to Flynt’s Vite integration:

```js
const wordpressHost = process.env.LOCAL_URL || process.env.VITE_DEV_SERVER_HOST || 'http://[PROJECT].localhost'
```

This assumes Vite is started through `./run`, so `LOCAL_URL` is already exported. If `LOCAL_URL` is not set, existing `VITE_DEV_SERVER_HOST` behavior still works.

## Test with a worktree

Go to your main project:

```sh
cd ~/www/example-website
```

Add a new worktree:

```sh
git worktree add ~/www/example-website-worktree
cd ~/www/example-website-worktree
```

Set the temporary worktree URL in `.env` and run the project install:

```sh
WORKTREE_URL="http://worktree.example-website.localhost"
echo "LOCAL_URL=$WORKTREE_URL" > .env

./run install
```

Open the worktree URL in your browser:

```text
http://worktree.example-website.localhost
```

Expected result: the WordPress site loads from the worktree.

## Optional: supacode setup

Automate the URL setup and project installation on worktree creation with [supacode](https://supacode.sh/).

First, add `supacode.json` to the project's `.gitignore` so the local supacode config stays out of git:

```gitignore
/supacode.json
```

Then create `supacode.json` locally (replace `[PROJECT]`):

```json
{
  "copyIgnoredOnWorktreeCreate": true,
  "runScript": "./run start",
  "setupScript": "WORKTREE=$(basename \"$SUPACODE_WORKTREE_PATH\")\nprintf 'LOCAL_URL=http://%s.[PROJECT].localhost' \"$WORKTREE\" > .env\n\n./run install\nsupacode worktree run"
}
```

