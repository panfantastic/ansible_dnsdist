DNSDist (with Domain Filtering)
=========

An Ansible role that sets up dnsdist as a network-wide domain blocker, the way
you would normally reach for Pi-hole, but with a much smaller memory footprint.

The blocklist (one to several million domains) lives in an LMDB database on disk rather
than in dnsdist's heap. dnsdist queries it over a LuaJIT FFI binding, and the
kernel page cache keeps the hot working set in RAM. In practice dnsdist sits at
around 50MB resident with a domain list loaded, versus the ~800MB
a comparable Pi-hole install uses.

Why this role exists
--------------------

There is plenty of prior art for dnsdist blocking, but it falls into two camps,
and neither is quite this:

- Blocklist *content* repos (StevenBlack, Hagezi, OISD and similar) produce the
  domain lists themselves. They are upstream of this role; you feed their output
  in, you do not replace them.
- dnsdist *config examples* (notably dmachard's, which use a CDB database) show
  how to wire blocking into dnsdist by hand. They are snippets to copy, not
  automation, and they load the list differently.

The official PowerDNS `dnsdist-ansible` role is the only widely-used dnsdist
role, but it is a generic installer (repos, listeners, backends, ACLs) with no
blocking story at all.

So the gap this fills is the combination: an automated, memory-conscious,
dnsdist-as-Pi-hole setup, with live blocklist updates that do not require
reloading dnsdist.

LMDB vs CDB
-----------

This role uses LMDB because it is MVCC: the nightly updater can apply deltas to
the live database in place, and dnsdist's per-query read transactions pick up
the new data on the next query with no reload, no file swap, and no service
restart.

That live-update ability has a cost. LMDB is copy-on-write, so a long-lived
reader (which dnsdist is) can prevent reclamation of freed pages, and the
on-disk file can grow over many update cycles. This is manageable (periodic
`mdb_reader_check`, an occasional `mdb_copy -c` compaction, or simply restarting
dnsdist out of hours)

If you do not need live in-place deltas, dmachard's CDB approach is a simpler
backend.


Requirements
------------

Ubuntu 24.04 onwards. No attempt has been made to make this compatible with any
other distribution, though Debian 13 is likely to work out of the box. The role
does not gate on OS family.

Role Variables
--------------

* dnsdist_console_key: a base64-encoded key for setKey(). Defaults to a random
32-character base64 string generated at template time.
* dnsdist_dnstap_collector: dnstap endpoint to send data to
* blocklist_urls_custom: {} # a dict+list of urls where you want to download a bunch of files
* blocklist_domains_custom: [] # domains to be blocked

Customising the config
----------------------

To add your own dnsdist logic, drop a `dnsdist.conf.j2` into
`{{ playbook_dir }}/templates/` based on the role's own template. The role uses
`first_found` and will prefer your playbook-level template over the bundled one,
so you can override the whole config without forking the role.

Monitoring
----------

The bundled config can expose query and response telemetry over a dnstap interface.
Ensure that dnsdist_dnstap_enabled is true to enable.

A typical setup pairs this with dnscollector (https://github.com/dmachard/DNS-collector or my equivalent role to set that up) to gather stats and forward them on
(for example to a Mimir cluster).  If you
label metrics by query name, watch series cardinality, though for a single
resolver it is unlikely to trouble a modestly-sized backend.

Example Playbook
----------------

    - hosts: servers
      roles:
         - name: dnsdist
           vars:
             dnsdist_console_key: "TSR6JSdCYUdgUD1kfndIQVI6R2g6XTd3PE55W2tBYHg="
             dnsdist_dnstap_enabled: true
             dnsdist_dnstap_collector: "127.0.0.1:6000"
             blocklist_urls_custom:
                 domain: wibble.com
                 files:
                    - wibble.txt
             blocklist_domains:
                - pastebin.com

License
-------

BSD

Author Information
------------------

Iain Grant <ansible@panfantastic.co.uk>
