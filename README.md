# .localhost MAMP Dev Setup

A local development setup for PHP projects:

- macOS
- Apache (`httpd`)
- MySQL
- PHP-FPM (multiple versions)
- `.localhost` domains (no `mkcert` or `dnsmasq` required)

> [!IMPORTANT]
> This guide uses `/Users/<USERNAME>/www` as a custom local web root.  
> Replace `<USERNAME>` with your macOS username and create that directory.

## 1. Install and configure MySQL

Install and start MySQL:

```bash
brew install mysql
brew services start mysql
```

Run this to remove insecure defaults from a fresh local install:

```bash
mysql_secure_installation
```

Use these answers:

- Password validation plugin: no
- Set root password: yes (example: `root` for local dev)
- Remove anonymous users: yes
- Disallow remote root login: yes
- Remove test database: yes
- Reload privilege tables: yes

## 2. Install and configure Apache

Install and start Apache:

```bash
brew install httpd
brew services start httpd
```

Configure Apache:

```bash
vim /opt/homebrew/etc/httpd/httpd.conf
```

1. Find and uncomment the following lines (remove leading `#`):

```apache
LoadModule rewrite_module ...
LoadModule proxy_module ...
LoadModule proxy_fcgi_module ...
```

2. Find and edit these directives to match:

```apache
Listen 80
Include /Users/<USERNAME>/www/httpd-vhosts.conf
```

3. Add this directory access block:

```apache
<Directory "/Users/<USERNAME>/www">
  AllowOverride All
  Require all granted
  DirectoryIndex index.php index.html
</Directory>
```

Restart Apache:

```bash
brew services restart httpd
```

## 3. Install and configure PHP versions

Install multiple PHP versions as needed:

```bash
brew install php@8.3 php@8.4
```

Configure dedicated ports for each PHP-FPM pool:

- `php@8.3` -> `127.0.0.1:9083`
- `php@8.4` -> `127.0.0.1:9084`

`vim /opt/homebrew/etc/php/8.3/php-fpm.d/www.conf`

```ini
listen = 127.0.0.1:9083
pm = ondemand
pm.max_children = 15
```

`vim /opt/homebrew/etc/php/8.4/php-fpm.d/www.conf`

```ini
listen = 127.0.0.1:9084
pm = ondemand
pm.max_children = 15
```

Start PHP-FPM services:

```bash
brew services start php@8.3 php@8.4
```

## 4. Configure a project

Create or edit your vhosts file:

```bash
vim ~/www/httpd-vhosts.conf
```

Add one vhost per project:

```apache
<VirtualHost *:80>
  ServerName example.localhost
  DocumentRoot "/Users/<USERNAME>/www/example"
  AddHandler proxy:fcgi://127.0.0.1:9083 .php
</VirtualHost>
```

Choose the `AddHandler` port per project PHP version:

- PHP 8.3: `AddHandler proxy:fcgi://127.0.0.1:9083 .php`
- PHP 8.4: `AddHandler proxy:fcgi://127.0.0.1:9084 .php`

Restart Apache:

```bash
brew services restart httpd
```

Open website:

```text
http://example.localhost
```

## Follow-up: wildcard domains for worktrees

If you use Git worktrees and want each worktree to run on its own `.localhost` subdomain, check out [Apache Wildcard Domains for Worktrees](apache-wildcard-domains-for-worktrees.md).

> To apply this setup with a coding agent, use the Bleech
> `localhost-mamp-worktrees` skill:
> https://github.com/bleech/skills/tree/main/localhost-mamp-worktrees
