# Apache Wildcard Domains for Worktrees

The goal is to configure Apache so Git worktrees can run at wildcard `.localhost` subdomains like:

```text
http://<WORKTREE>.<PROJECT>.localhost
```

This guide assumes you have the [`localhost-mamp-dev-setup`](https://github.com/bleech/localhost-mamp-dev-setup) configured and only covers host-level Apache routing.

In addition, I recommend configuring your WordPress project to allow setting local URLs from `.env` to work around issues with Git, Vite, and DB cloning. See the following migration tutorial: [`wordpress-env-urls-for-worktrees`](wordpress-env-urls-for-worktrees.md).

> [!IMPORTANT]
> If your project uses the new devstack, you don't need any of this.

---

## 1. Configure Apache for wildcard subdomains

Open Apache's Homebrew config:

```bash
vim /opt/homebrew/etc/httpd/httpd.conf
```

Find this line and uncomment it by removing the leading `#`:

```apache
LoadModule vhost_alias_module lib/httpd/modules/mod_vhost_alias.so
```

This module is required for `VirtualDocumentRoot`, which lets Apache map wildcard subdomains to dynamic folders.

---

## 2. Configure a WordPress project vhost

Open your vhosts file:

```bash
vim ~/www/httpd-vhosts.conf
```

Add this block, replacing `<USERNAME>`:

```apache
<VirtualHost *:80>
  ServerAlias *.example-website.localhost
  VirtualDocumentRoot "/Users/<USERNAME>/www/example-website-%1/wordpress"
  AddHandler proxy:fcgi://127.0.0.1:9085 .php
</VirtualHost>
```

`%1` maps to the `*` of the hostname, so `worktree.example-website.localhost` becomes:

```text
/Users/<USERNAME>/www/example-website-worktree/wordpress
```

Restart Apache:

```bash
brew services restart httpd
```

---

## 3. Test with a worktree

Go to your main project:

```bash
cd ~/www/example-website
```

Add a new worktree:

```bash
git worktree add ~/www/example-website-worktree
cd ~/www/example-website-worktree
```

Set the temporary worktree URL in `wp-config.php` and run the project install:

```bash
WORKTREE_URL="http://worktree.example-website.localhost"
wp config set WP_HOME "$WORKTREE_URL"
wp config set WP_SITEURL "$WORKTREE_URL"

./run install
```

Open the worktree URL in your browser:

```text
http://worktree.example-website.localhost
```

Expected result: the WordPress site loads from the worktree.

## Optional: Supacode setup

If you use Supacode worktrees, route wildcard domains to Supacode's worktree directory instead of `~/www`.

First, ensure Apache can serve Supacode worktrees by adding this directory access block, replacing `<USERNAME>`:

```apache
<Directory "/Users/<USERNAME>/.supacode/repos">
  AllowOverride All
  Require all granted
  DirectoryIndex index.php index.html
</Directory>
```

Then use a Supacode-specific wildcard vhost, replacing `<USERNAME>` and the project name:

```apache
<VirtualHost *:80>
  ServerAlias *.example-website.localhost
  VirtualDocumentRoot "/Users/<USERNAME>/.supacode/repos/example-website/%1/wordpress"
  AddHandler proxy:fcgi://127.0.0.1:9085 .php
</VirtualHost>
```

With this setup, this URL:

```text
http://worktree.example-website.localhost
```

maps to:

```text
/Users/<USERNAME>/.supacode/repos/example-website/worktree/wordpress
```

Restart Apache after changing the config:

```bash
brew services restart httpd
```
