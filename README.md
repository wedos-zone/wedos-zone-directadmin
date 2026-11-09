<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset=".github/logo-dark.webp">
    <img src=".github/logo-light.webp" alt="WEDOS.zone" width="420">
  </picture>
</p>

<p align="center">
  <a href="https://github.com/wedos-zone/wedos-zone-directadmin/releases/latest"><strong>Download the latest release</strong></a>
  · <a href="https://wedos.zone">wedos.zone</a>
</p>

# WEDOS.zone for DirectAdmin

Your DNS zones stay where you manage them — in DirectAdmin. This plugin reports every
change to WEDOS.zone, our global anycast DNS, so your customers' domains are answered
from our network instead of from a single server.

Full documentation: <https://kb.wedos.global/>

---

## Before you install

**1. A WEDOS account with WAPI access.** WAPI is the API of the WEDOS administration.
You enable it there, set a WAPI password (it is not your login password) and allow the
IP address of this server.

**2. Allow this server's IP.** Without it every request is refused. The change takes
**up to 30 minutes** to apply, so do it first and install afterwards.

> If your server has IPv6, it may leave through the IPv6 address even when you only
> allowed the IPv4 one. The plugin tells you which address we saw you coming from —
> allow that one.

**3. Decide how you want to use WEDOS.zone.** The plugin asks on the first screen and
the answer is not a switch you flip later, so it is worth a minute now:

| | Secondary — recommended | Primary |
|---|---|---|
| Who holds the zone | your servers | we do |
| Source of truth | your DNS | DirectAdmin |
| What we send | zone names only | the records themselves |
| What it needs from you | a zone transfer we can pull | nothing on the firewall |

**Secondary** suits you if you already run DNS servers and want us as the public,
resilient layer in front of them. **Primary** suits you if you have no DNS of your own —
for example if your domains currently sit on the registrar's nameservers.

**4. For secondary mode only — let us pull the zones.** In `named.conf`:

```
allow-transfer { 46.28.104.64/27; };
```

Allowing it in `named.conf` is not enough. **The server firewall has to let port 53
through as well** — CSF, for example, needs `53` in `TCP_IN`. The plugin cannot see the
firewall and will not warn you about it.

---

## Install

1. **Admin Level → Extra Features → Plugin Manager → Install new plugin**
2. Choose **URL** and paste:

   ```
   https://wedos.zone/plugins/directadmin/wedos_zone.tar.gz
   ```

3. Open **Extra Features → WEDOS.zone** and answer the question about the mode.
4. Fill in your WAPI account and password. In secondary mode also the primary server IP
   (leave it empty to use the server's own address).
5. The plugin starts in **test mode** — WAPI only checks the commands and changes
   nothing. Leave it on until the screen looks healthy.
6. Turn off test mode and tick **Report zone changes to WEDOS.zone**.

Nothing happens until you do step 6. The plugin is deliberately inert on install.

### Moving from your own scripts

If you already create secondary zones at WEDOS with your own WAPI scripts, the plugin
takes over from them. Nothing has to be deleted or created again.

1. Install the plugin and fill in the same WAPI account your scripts use. Zones belong to
   the account; with a different one the plugin reports them as held by another account.
2. While test mode is still on, run `wz.php reconcile`. It only compares and changes
   nothing. A zone on both sides is taken as it is. A zone only on this server would be
   created. A zone only at WEDOS is only reported, never deleted, so zones of your other
   servers on the same account are safe.
3. Zones that already exist keep the primary IP your scripts set. If it was this
   server's address, there is nothing to do. The plugin does not rewrite it.
4. Switch your scripts off, then turn off test mode and tick **Report zone changes to
   WEDOS.zone**. Two tools adding and removing the same zones would undo each other.

---

## Everyday use

There is nothing to do. Add, change or delete a zone in DirectAdmin as usual and the
plugin reports it. The panel page shows the queue, the hourly budget and anything that
needs your attention.

**Exceptions per zone.** A single zone can run the other way round than the server
default — useful while you are moving a customer, or for a domain that has to stay on
your own DNS.

**Switching zones that already exist.** A zone's mode cannot be changed in place; WEDOS
accepts the request and does nothing. The plugin therefore deletes the zone and creates
it again, on purpose, with a warning and a confirmation. There is a short gap while that
happens, so do it deliberately — one zone at a time, or all of them in a bulk switch that
you can stop at any point.

---

## WEDOS.protection

The **Protection** tab lists the zones on this server and lets you put any of them behind
WEDOS.protection. The plugin writes the records that send traffic through the proxy and
watches them afterwards; the cron does the work, one domain at a time.

You can try it right after installation on up to 250 domains on the Free plan and 25 on
the paid one; higher plans are sold directly by WEDOS. Paid plans need a WEDOS credit
account. Those two limits are a demo: with WhiteLabel they do not apply, and the
protection is shown under your own name instead of a `wedos.global` one — contact us to
set it up. The limit blocks ordering only. Switching protection off and releasing a
domain always work, whatever the count says.

**Switching on** takes about 15–25 minutes and needs nothing from you. The plugin orders
the service, waits until WEDOS is ready and then changes three things in the domain's zone
in DirectAdmin:

- the apex `A` records point to the WEDOS.protection proxy,
- `www` becomes a `CNAME` to the proxy,
- `_acme-challenge` becomes a `CNAME` to WEDOS, which issues the certificate.

These records get a 300-second TTL so that switching off is quick. The domain has to be
delegated to the WEDOS nameservers (or your branded ones), otherwise WEDOS cannot verify
it and the plugin tells you so. Finish any change to the domain's nameservers before you
switch protection on: WEDOS checks the domain at intervals that can be a few hours long,
so a check that catches the change halfway delays switching on until the next one.

**Switching off** puts the original records back first and cancels the service only when
DNS no longer points to the proxy — about 10 minutes. A switch-on that never finishes (for
example a domain that is not registered) can be stopped with **Release**; a domain stuck
anywhere else has **Give up and clean up**.

**Deleting a domain or account switches its protection off.** The service is cancelled
once the domain no longer points to the proxy, so moving an account to another server does
not take the site down: while the copied zone still points to WEDOS.protection, the service
keeps running.

**Records you change by hand are reported, not undone.** Once a day the plugin checks that
the zone of every protected domain still carries the proxy records. If the apex or `www`
was changed, the Protection tab says so; the service is not cancelled.

**Where the proxy sends visitors.** When protection is switched on, the plugin gives WEDOS
the public IPv4 and IPv6 addresses of the apex and `www` from the zone as the origin.
Addresses of WEDOS.protection itself and private addresses are left out: the proxy cannot
reach them. If no address is left, protection is not switched on and the Protection tab
says why. Each protected domain shows its origin.

**After moving an account or changing the server IP, change the origin yourself.** The
proxy keeps sending visitors to the address it got when protection was switched on. Once a
day the plugin compares it with the IP DirectAdmin uses for the domain. When they differ,
the Protection tab shows both addresses and you get a notification. The plugin cannot
change the origin for you, because WAPI cannot. Change the target IP of the protection in
the WEDOS administration, then click **Target changed** (or run
`wz.php protection origin <domain>`). Only IPv4 is compared. Domains whose service was not
created by the plugin show no origin and are not checked.

**Before you switch protection off, make sure the server has a certificate for the
domain.** While the domain is protected, WEDOS serves the certificate. After switching
off, visitors get whatever certificate this server holds for that name — if there is
none, browsers show a warning. The plugin never touches certificates in DirectAdmin;
issuing and renewing them stays where it was before protection was switched on.

### Your own protection hostname (WhiteLabel CNAME)

With WhiteLabel CNAME, `www` of a protected domain points to a name of yours instead of
`<domain>.wedos.global`. WEDOS gives you the protection IP addresses; then:

1. In your own zone, add one wildcard record for a name of your choice, for example
   `*.proxy.example.com` → your protection IPv4 addresses (`A`) and IPv6 addresses
   (`AAAA`). Put **nothing else** under `proxy.example.com`: any other record there stops
   the wildcard from answering for that part of the name.
2. In **Settings → Protection hostname**, enter `proxy.example.com` and save. The plugin
   checks that a random name under it leads to WEDOS.protection and refuses to save it
   otherwise. The same check runs on `wz.php set proxy_jmeno <name>`.

From then on, every domain you switch on gets `www` → `<domain>.proxy.example.com`.
Domains that are already protected keep their records until protection is switched off and
on again. The apex and `_acme-challenge` records are set as before. Empty the field to go
back to the WEDOS name.

---

## Limits worth knowing in advance

- **1000 WAPI requests per hour per account.** The plugin keeps to 800 and leaves the
  rest for whatever else uses the account. When the budget runs out the queue simply
  waits for the next hour; nothing is lost.
- **The shortest TTL we accept is 300 seconds.** Anything shorter is rejected outright,
  so the plugin raises it to 300 and tells you which zones and which records. Your own
  server keeps the original value — see below.
- **NS records at the root of the zone are not sent in primary mode.** We hold the zone,
  so the nameservers are ours. Records on subdomains are sent normally.
- **A change reaches the public nameservers in roughly ten minutes.** Most of that is
  publication on our side, not the plugin.
- **DNSSEC: a zone DirectAdmin signs cannot run in primary mode.** In primary we hold
  the zone and receive individual records, not a signed zone, so the signatures do not
  travel. If the domain has a DS record in the registry, it would fail validation and
  stop resolving. WEDOS can sign a zone only for domains registered with WEDOS.

  The plugin will not send such a zone: it blocks it and reports it, naming the domain.
  Two ways out — turn DNSSEC off for that domain in DirectAdmin, or run that one zone in
  **secondary** mode, where the zone transfer carries the signatures. If you need DNSSEC
  on domains registered elsewhere, secondary mode is the answer.

---

## When something is not right

The panel says what happened and what to do about it. Two cases are worth explaining
here, because they are easy to misread:

**A task is blocked.** Something was refused in a way that retrying will not fix — a
wrong password, an IP that is not allowed, or a zone that belongs to a different WEDOS
account. The plugin stops instead of hammering the API. Fix the cause, then put the task
back in play.

**Two copies of the same zone.** In primary mode we hold the zone. If your own
nameservers are still in the delegation at the registrar, the world gets answers from
two places and they drift apart — a TTL below 300 s, for instance, we raise and your
server does not. Leave only the nameservers that point to us in the delegation.

---

## Updating

**The plugin updates itself, from its own page.** Once a day it asks GitHub whether a
newer version is out. When there is one, a line appears at the top of the WEDOS.zone page
with an **Update** button. The button only writes down that you want it: within a minute
the cron downloads the package, checks our signature, replaces the files and runs the
installer. Nothing is installed without that click.

From the command line:

```
php /usr/local/directadmin/plugins/wedos_zone/scripts/wz.php update
php .../wz.php update --install
```

Your settings, per-zone exceptions, the queue and the zone memory live outside the plugin
directory and survive the update untouched.

**Why not Plugin Manager.** Its *Update* only shows that a newer version exists. Its API
can add a plugin, not replace one, so installing over an existing plugin would mean
deleting it first — and deleting takes the whole plugin directory with it. The update
above does the same job without that step.

**Every package is signed.** The update runs as root on your server, so a file that was
swapped on the way would run as root too. The plugin carries our public key and installs
nothing that does not match it; a package that fails the check is thrown away and the
installed version keeps running. The log says which one it was and why.

Releases, with the package, its signature and this guide:
<https://github.com/wedos-zone/wedos-zone-directadmin/releases>

## Removing

**Plugin Manager → Delete.** Uninstalling removes the WAPI password and the TSIG key —
they have no business staying on disk — but keeps everything else, so installing the
plugin again picks up where you left off and only the password has to be entered once
more. Turn on *Remove all data when the plugin is uninstalled* if you want it all gone.

**Your zones in WEDOS.zone are never deleted.** Removing the plugin is a decision about
this server, not about your customers' DNS.

---

## From the command line

Everything in the panel is available as `wz.php`, run as root:

```
php /usr/local/directadmin/plugins/wedos_zone/scripts/wz.php status
php .../wz.php test-auth                  verify the login and the allowed IP
php .../wz.php queue                      what is waiting
php .../wz.php retry <domain>|--all       put blocked tasks back in play
php .../wz.php reconcile [--apply]        compare this server against WEDOS.zone
php .../wz.php switch <domain> primary|secondary
php .../wz.php migrate primary|secondary  switch every zone
php .../wz.php update [--install]         check for a new version, or install it
```

---

## The log

Everything the plugin does goes to `/var/lib/wedos_zone/wedos_zone.log`, one line per
event, with the level `INFO`, `WARN` or `ERROR`. The log records what was sent and what
came back; it never contains your password.

Follow it live:

```bash
tail -F /var/lib/wedos_zone/wedos_zone.log
```

Only warnings and errors, live or from the whole file:

```bash
tail -F /var/lib/wedos_zone/wedos_zone.log | grep --line-buffered -E ' (WARN|ERROR) '
grep -E ' (WARN|ERROR) ' /var/lib/wedos_zone/wedos_zone.log
```

Use `-F`, not `-f`. At 5 MB the log is renamed to `wedos_zone.log.1` and a new one starts;
`-F` follows the new file, `-f` would stay on the old one.

On the plugin page, Settings shows the end of the log, with *show only those* when there
are warnings.

---

## Getting help

Include the output of `wz.php status` and the last lines of
`/var/lib/wedos_zone/wedos_zone.log`.
