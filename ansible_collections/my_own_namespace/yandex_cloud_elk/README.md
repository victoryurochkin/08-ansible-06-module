# my_own_namespace.yandex_cloud_elk

Custom Ansible collection for Netology homework.

## Description

This collection contains custom Ansible module my_own_module and role create_file.

The module creates or updates a text file on a host.

## Module

Module name:

my_own_namespace.yandex_cloud_elk.my_own_module

Parameters:

- path: required string, path to the file
- content: required string, file content

Example:

- name: Create file using custom module
  my_own_namespace.yandex_cloud_elk.my_own_module:
    path: /tmp/example.txt
    content: "Hello from custom Ansible module"

## Role

Role name:

my_own_namespace.yandex_cloud_elk.create_file

Default variables:

my_own_module_path: /tmp/my_own_module_role_test.txt
my_own_module_content: "Hello from custom Ansible role"

Example:

- name: Test custom role from collection
  hosts: localhost
  gather_facts: false

  roles:
    - role: my_own_namespace.yandex_cloud_elk.create_file
