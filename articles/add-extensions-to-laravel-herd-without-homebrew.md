---
slug: add-extensions-to-laravel-herd-without-homebrew
issue: add-extensions-to-laravel-herd-without-homebrew
title: "Add extensions to Laravel Herd without Homebrew"
description: You don't HAVE to use Homebrew...
date: 2024-04-02
category: guides
authors:
  - mitchell-davis
published: true
---

We've recently started using [Laravel Herd](https://herd.laravel.com/) on our development machines for most of our
projects.

Herd is a project that takes [Laravel Valet](https://laravel.com/docs/master/valet), which we used for many years, and
wraps it up into a nice UI. In addition, Herd will install precompiled versions of PHP for you, whereas Valet depends on
[Homebrew](https://brew.sh/) to compile PHP on your machine.

For all the benefits it brought the Mac ecosystem, we have experienced many instances where doing a Homebrew update
would cause other dependencies to break on our machines, leading to **hours or even days of lost productivity** to get
everything fixed and running again.

[Laravel Sail](https://laravel.com/docs/master/sail) solves this problem by building all of your dependencies inside a
Docker container which won't be effected by Homebrew, however Sail can be quite heavy to get setup and can be overkill
on simple projects.

Herd gives us a different approach where PHP is precompiled, so you don't need to worry about compiling it yourself, or
dealing with Homebrew in general - it's largely just install Herd, and you're off to the races.

## Default extensions

Because Herd's PHP is precompiled, you only get access to a set of common extensions that are used in most projects.

You can see these if you install Herd, and then run `php -m`:

```
[PHP Modules]
bcmath
bz2
calendar
Core
ctype
curl
date
dba
dom
exif
FFI
fileinfo
filter
ftp
gd
gmp
hash
iconv
imagick
imap
intl
json
ldap
libxml
mbstring
mysqli
mysqlnd
openssl
pcntl
pcre
PDO
pdo_mysql
pdo_pgsql
pdo_sqlite
pgsql
Phar
posix
random
readline
redis
Reflection
session
shmop
SimpleXML
soap
sockets
sodium
SPL
sqlite3
standard
sysvmsg
sysvsem
sysvshm
tokenizer
xml
xmlreader
xmlwriter
xsl
Zend OPcache
zip
zlib

[Zend Modules]
Zend OPcache
```

## The official way to add PHP extensions to Laravel Herd

For a recent talk I was working on, I wanted to parse email files (`.eml`) in realtime using Laravel, and to do that I
needed the [Mailparse](https://www.php.net/manual/en/book.mailparse.php) PHP extension.

The [Herd documentation](https://herd.laravel.com/docs/1/advanced-usage/additional-extensions) explains:

> You may add additional PHP extensions that are not included out of the box with Herd, by installing them via Homebrew
> and pecl.

This immediately gave me flashbacks to the dark days of yore, and I knew this wasn't how I wanted to get started with my
talk - dealing with not only installing Homebrew, then PHP, then running PECL, only to have it probably break right
before my talk 😅

{% call-to-action
title="Looking for help with your Laravel project?"
/%}

## The better way to add PHP extensions to Laravel Herd

I did some research, crawling through GitHub issues, until I found
this [amazing comment](https://github.com/beyondcode/herd-community/issues/145#issuecomment-1740752325)
by [SagarNaliyapara](https://github.com/SagarNaliyapara) on GitHub.

> I downloaded extension manually from
> here - [https://packages.macports.org/php81-xsl/](https://packages.macports.org/php81-xsl/) and then found .so file in
> it, added into php.ini and it fixed the issue for me

Thanks to Sagar's comment, I had now seen the light.

I had never heard of the [MacPorts](https://www.macports.org/) project before, but their mission is to create an
easy-to-use system for compiling, installing, and upgrading open-source software on the Mac operating system - _and boy
have they done it!_

I went searching in their [huge list of packages](https://packages.macports.org/) (trust me there are a lot), and
using `Ctrl+F` sure enough I found the magical `php83-mailparse` that
I was looking for.

Clicking into [that directory](https://packages.macports.org/php83-mailparse/) I could see they had lots of variants for
different environments:

- `php83-mailparse-3.1.6_0.darwin_10.i386.tbz2`
- `php83-mailparse-3.1.6_0.darwin_10.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_11.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_12.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_13.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_14.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_15.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_16.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_17.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_18.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_19.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_20.arm64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_20.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_21.arm64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_21.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_22.arm64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_22.x86_64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_23.arm64.tbz2`
- `php83-mailparse-3.1.6_0.darwin_23.x86_64.tbz2`

From this list, I could see that they only had the [latest version](https://pecl.php.net/package/mailparse) of
Mailparse, 3.1.6, which was perfect for what I needed.

Next I needed to determine my own environment, so that I knew which version to download. From the filenames, it looked
like I needed to determine two things:

- My "Darwin" version
- My architecture (`x86_64` or `arm64`)

Fortunately for us, Herd has done the hard work of getting PHP setup for our machine, so we can piggyback off it and get
both details with a simple command:

```bash
php -i | grep "Build System => "
```

Running that will give you an output that looks like this:

```text
Build System => Darwin oh-162-55-253-165 22.5.0 Darwin Kernel Version 22.5.0: Mon Apr 24 20:53:44 PDT 2023; root:xnu-8796.121.2~5/RELEASE_ARM64_T8103 arm64
```

From this, we can see that for the precompiled version of PHP that we are running, our Darwin version is `22.5.0`, and
we are on an `arm64` architecture.

{% callout type="warning" title="Be careful" %}
Keep in mind you are downloading a library from the Internet and running it on your machine, so be sure that you're
comfortable with the risk associated with that.
{% /callout %}

I then downloaded the corresponding file, `php83-mailparse-3.1.6_0.darwin_22.arm64.tbz2`, and just using Finder I could
decompress the file to reveal the contents. I then just needed to locate the magical `mailparse.so` that I so
desperately needed.

```text
$ tree php83-mailparse-3.1.6_0.darwin_22.arm64
php83-mailparse-3.1.6_0.darwin_22.arm64
├── +COMMENT
├── +CONTENTS
├── +DESC
├── +PORTFILE
├── +STATE
└── opt
    └── local
        ├── lib
        │   └── php83
        │       └── extensions
        │           └── no-debug-non-zts-20230831
        │               └── mailparse.so
        ├── share
        │   └── doc
        │       └── php83-mailparse
        │           ├── CREDITS
        │           └── README.md
        └── var
            └── db
                └── php83
                    └── ~mailparse.ini

13 directories, 9 files
```

You can see the `mailparse.so` extension is contained inside `opt/local/lib/php83/extensions/no-debug-non-zts-20230831`,
so I just went to that directory in Finder, and copied the extension file.

I then realised I wasn't sure what to do next.

### Where should we place it?

Typically, Homebrew will setup somewhere inside our `/opt/homebrew/lib/php` folder to store extensions, but since we
aren't using Homebrew, we don't have a place already established on our machine to store extensions.

Fortunately, PHP can load extensions from anywhere, so it's really up to you for where you want to put them.

I opted to just put the extension directly in the same folder as the Herd `php.ini` for this version of PHP, so that as
I switch between PHP versions, the extension is only enabled for the correctly compiled PHP version that it corresponds
to.

You can find that directory by running this command:

```bash
php --ini | grep "Additional .ini files parsed:"
```

That ended up being this path on my machine:

```text
/Users/mdavis/Library/Application Support/Herd/config/php/83/php.ini
```

I copied the `mailparse.so` from the compressed file we downloaded and uncompressed, and pasted it into the same folder
as the `php.ini` that Herd is reading, which in this case
was `/Users/mdavis/Library/Application Support/Herd/config/php/83/`.

If you're not able to navigate to that file in Finder, try using `Go > Go to Folder...` from the Finder menu bar, and
then paste in the path. You can also use Herd's `Open configuration files` menu bar item, and then navigate to the right
PHP version.

### Enabling the extension

I then opened the `php.ini` in this directory, and with a single line added at the end of the file, I was able to enable
the extension:

```text
extension=/Users/mdavis/Library/Application Support/Herd/config/php/83/mailparse.so
```

I was surprised to see that I didn't need to add double quotes around the path, despite the fact that it contained a
space.

Make sure you save the file, and you're all set.

I then **restarted PHP** from the Herd menu bar using `Stop all` and then `Start all`.

### Taking the extension out of quarantine

Immediately I was presented with a system popup telling me:

> "mailparse.so" can't be opened because Apple cannot check it for malicious software.

This is a protection built into macOS to stop you running executables from unknown (unsigned) developers.

{% callout type="warning" title="No really, be careful" %}
It's worth reiterating that you are trusting software from the Internet, which could do anything on your machine.

I trust the process at MacPorts, so I proceeded with the following.
{% /callout %}

You can get around it by taking the file out of quarantine with this command, importantly noting that **we do** need the
double quotes this time, and substituting in your own path to your extension:

```bash
xattr -d com.apple.quarantine "/Users/mdavis/Library/Application Support/Herd/config/php/83/mailparse.so"
```

I then **restarted PHP** from the Herd menu bar, again using `Stop all` and then `Start all`.

Now, when I run `php -m` to list our installed modules, I now see `mailparse` 🎉

```text
[PHP Modules] [tl! focus]
bcmath [tl! collapse:start]
bz2
calendar
Core
ctype
curl
date
dba
dom
exif
FFI
fileinfo
filter
ftp
gd
gmp
hash
iconv
imagick
imap
intl
json
ldap
libxml [tl! collapse:end]
mailparse [tl! focus]
mbstring [tl! collapse:start]
mysqli 
mysqlnd
openssl
pcntl
pcre
PDO
pdo_mysql
pdo_pgsql
pdo_sqlite
pgsql
Phar
posix
random
readline
redis
Reflection
session
shmop
SimpleXML
soap
sockets
sodium
SPL
sqlite3
standard
sysvmsg
sysvsem
sysvshm
tokenizer
xml
xmlreader
xmlwriter
xsl
Zend OPcache
zip
zlib

[Zend Modules]
Zend OPcache [tl! collapse:end]
```

### This doesn't always work

If you **still** get issues with PHP being unable to load a dynamic library, it's possible that you are missing some
dependencies of the extension, in which case your mileage may vary. I was fortunate with the `mailparse` extension, but
in testing for this article I installed the `uuid` extension and found that it needed another library called `libuuid`.

[MacPorts should have](https://packages.macports.org/) the missing library for you, but at this point it does start to
get a bit tedious to find the right file, decompress, move, and unquarantine the file, so in this case you may actually
want to consider using Homebrew. I wish you luck!

## Finishing up

If you need to install PHP extensions beyond what Laravel Herd comes with out of the box, and don't want to deal with
dependency hell, I can strongly recommend using this method as **it just works**.

We are enormously thankful to the folks behind [MacPorts](https://www.macports.org/) for making this possible.