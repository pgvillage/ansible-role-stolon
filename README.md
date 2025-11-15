Stolon
=========

Stolon is a tool to manage HA PostgreSQL clusters.
Stolon uses etcd to store config and manage consensus.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

Requirements
------------

This role aims at using an RPM from the MannemSolutions repo.

Role Variables
--------------

Please see [defaults](https://github.com/pgvillage/ansible-role-stolon/blob/main/defaults/main.yml) for all variables


Dependencies
------------

- pgvillage.etcd


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - pgvillage.stolon

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
