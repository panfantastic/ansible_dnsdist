DNSDist (with Domain Filtering)
=========

This role will configure dnsdist for domain blocking.

Provide your own {{ playbook_dir }}/templates/dnsdist.conf.j2 based on the role's to add your own logic.

Requirements
------------

Ubuntu 24.04 onwards.  No attempt has been made to make this compatible with any other distribution, though Debian 13 is likely to work out of the box.

Role Variables
--------------

default_console_key: a base64 encoded key for setKey().  Defaults to a random string.


Example Playbook
----------------

    - hosts: servers
      roles:
         - name: dnsdist
           vars:
             dnsdist_console_key: "TSR6JSdCYUdgUD1kfndIQVI6R2g6XTd3PE55W2tBYHg="

License
-------

BSD

Author Information
------------------

Iain Grant <ansible@panfantastic.co.uk>
