---
categories:
  - "administration"
  - "automation"
keywords:
  - "ansible"
tags:
  - "ansible"
  - "rabbitmq"
  - "rest api"
  - "devops"
  - "sensu"
date: "2017-01-20"
title: "Ansible - Interacting with external REST API"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-ansible.jpg"

---

Ansible has many powerful modules. One of which is called [**uri**](http://docs.ansible.com/ansible/uri_module.html) which is capable of sending any kind of `HTTP` request. Using this module, it is fairly simple to allow ansible to intelligently talk to a REST API. This will come in handy during for automation of the [sensu](https://sensuapp.org/) monitoring docker infrastructure I am currently working on.
<!--more-->

<!-- toc -->

# Ansible modules
So Ansible already has modules that can interact directly with rabbitmq. I only need ansible to create a user and vhost that the sensu servers and clients can use. The ones I would normally use if my rabbitmq hosts **were not dockerized** are:

> [**rabbitmq_user**](http://docs.ansible.com/ansible/rabbitmq_user_module.html) - Adds or removes users to RabbitMQ<br />
> [**rabbitmq_vhost**](http://docs.ansible.com/ansible/rabbitmq_vhost_module.html) - Manage the state of a virtual host in RabbitMQ

# RabbitMQ REST API
Unfortunatly, it appears that all these modules cannot interact with remote rabbitmq daemons. Since we are running rabbitmq in a docker container, **we will not have ssh daemon in the docker container for ansible to connect to**. However, installing the admin plugin for rabbitmq exposes a REST API, which we can use to create vhosts and users. Fortunatly, there already is an offical docker image which has the admin plugin already installed: [3.6-management-alpine](https://hub.docker.com/r/_/rabbitmq/)

Now we can simply bring up this rabbitmq docker container and explore the API:

```bash
docker run -d --name rabbitmq-mgmt1 -p 127.0.0.1:15672:15672 rabbitmq:3.6-management-alpine
```

We can use curl to take a look at the vhosts and users:

```bash
$ curl -s -u guest:guest http://127.0.0.1:15672/api/vhosts | python -m json.tool
[
    {
        "name": "/",
        "tracing": false
    }
]

$ curl -s -u guest:guest http://127.0.0.1:15672/api/users | python -m json.tool
[
    {
        "name": "guest",
        "password_hash": "X8uG3fwwu3+R3VCEvv0/XC5WI2YcEqYai3+xtb8CT9OaJ0Fl",
        "hashing_algorithm": "rabbit_password_hashing_sha256",
        "tags": "administrator"
    }
]
```

# Creating Ansible Playbook for REST API Integration
According to the [api-docs](http://hg.rabbitmq.com/rabbitmq-management/raw-file/rabbitmq_v3_3_4/priv/www/api/index.html), these endpoints support PUT requests to insert data. Instead of doing curls, let's build a customizable playbook for Ansible to execute. Lets start by defining the variables we need:

{{< codeblock "rabbitmq-api-playbook.yml" "yaml" >}}
---
- hosts: localhost
  connection: local
  gather_facts: no

  vars:

    rabbitmq_auth_user: "guest"
    rabbitmq_auth_password: "guest"
    rabbimq_rest_api_url: "http://127.0.0.1:15672/api"
    rabbitq_vhost_sensu: "sensu"
    rabbitq_user_sensu: "sensu"

    rabbitq_user_sensu_request_body:
      password: "test"
      tags: ""

    rabbitmq_permissions_sensu_request_body:
      configure: ".*"
      write: ".*"
      read: ".*"
{{< /codeblock >}}

As listed above, we need an rest endpoint to talk to along with the credentials. Also we need the name of the vhost and user we want to create, along with the request body for the user and permissions. Great! Now let's make our tasks.

We want to first list if our vhost is already present. If it is not, the REST endpoint will return a `404` on a `GET` request, if the vhost is not present. We need to store this response, so we can use it later to decide if we should do a second request to `PUT` this vhost on the system. Also, we want the task to fail on any response codes other than `200` (present) or `404` (not found).

{{< codeblock "rabbitmq-api-playbook.yml" "yaml" >}}
  tasks:

    - name: check if sensu vhost is present
      uri:
        url: "{{ rabbimq_rest_api_url }}/vhosts/{{ rabbitq_vhost_sensu }}"
        method: GET
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 200,404
        timeout: 10
      register: request_vhost

    - name: debug
      debug:
        var: request_vhost

    - name: create vhost
      uri:
        url: "{{ rabbimq_rest_api_url }}/vhosts/{{ rabbitq_vhost_sensu }}"
        method: PUT
        HEADER_Content-Type: "application/json"
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 204
        timeout: 10
      when:
        - request_vhost.status == 404
{{< /codeblock >}}

Running this playbook produces the correct result; It will add the vhost if not present, but will skip it if present:

```bash
$ ansible-playbook rabbitmq-api-playbook.yml

PLAY [localhost] ***************************************************************

TASK [check if sensu vhost is present] *****************************************
ok: [localhost]

TASK [debug] *******************************************************************
ok: [localhost] => {
    "request_vhost": {
        "changed": false,
        "content_length": "55",
        "content_type": "application/json",
        "date": "Fri, 10 Jan 2017 21:21:18 GMT",
        "json": {
            "error": "Object Not Found",
            "reason": "\"Not Found\"\n"
        },
        "msg": "HTTP Error 404: Object Not Found",
        "redirected": false,
        "server": "MochiWeb/1.1 WebMachine/1.10.0 (never breaks eye contact)",
        "status": 404,
        "url": "http://127.0.0.1:15672/api/vhosts/sensu",
        "vary": "Accept-Encoding, origin"
    }
}

TASK [create vhost] ************************************************************
ok: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0   

# Running a second time, skips the step called "create vhost"
$ ansible-playbook rabbitmq-api-playbook.yml

PLAY [localhost] ***************************************************************

TASK [check if sensu vhost is present] *****************************************
ok: [localhost]

TASK [debug] *******************************************************************
ok: [localhost] => {
    "request_vhost": {
        "cache_control": "no-cache",
        "changed": false,
        "content_length": "32",
        "content_type": "application/json",
        "date": "Fri, 10 Jan 2017 21:23:39 GMT",
        "json": {
            "name": "sensu",
            "tracing": false
        },
        "msg": "OK (32 bytes)",
        "redirected": false,
        "server": "MochiWeb/1.1 WebMachine/1.10.0 (never breaks eye contact)",
        "status": 200,
        "url": "http://127.0.0.1:15672/api/vhosts/sensu",
        "vary": "Accept-Encoding, origin"
    }
}

TASK [create vhost] ************************************************************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0  
```

Fortunatly, the same logic is needed for creating a user is similiar but requires us to send a body in `json` format for `PUT` requests. Personaly, I love `yaml` and prefer to represent my variables in `yaml` format, then convert the dictionary to `json` format with a filter called `to_json`. The tasks looks like this:

{{< codeblock "rabbitmq-api-playbook.yml" "yaml" >}}
    - name: create user
      uri:
        url: "{{ rabbimq_rest_api_url }}/users/{{ rabbitq_user_sensu }}"
        method: PUT
        HEADER_Content-Type: "application/json"
        body: "{{ rabbitq_user_sensu_request_body | to_json }}"
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 204
        timeout: 10
      when:
        - request_user.status == 404
{{< /codeblock >}}

## The Finished Playbook
OK great! Notice how we have to set the `HEADER_Content-Type` parameter, and how we convert the `rabbitq_user_sensu_request_body` variable to json format. Now we can put everything together and create the full playbook. Also the great thing about this playbook is that its **idempotence**. We can run it many times without it changing anything unless the change is needed.

{{< codeblock "rabbitmq-api-playbook.yml" "yaml" >}}
---

- hosts: localhost
  connection: local
  gather_facts: no

  vars:

    rabbitmq_auth_user: "guest"
    rabbitmq_auth_password: "guest"
    rabbimq_rest_api_url: "http://127.0.0.1:15672/api"
    rabbitq_vhost_sensu: "sensu"
    rabbitq_user_sensu: "sensu"

    rabbitq_user_sensu_request_body:
      password: "test"
      tags: ""

    rabbitmq_permissions_sensu_request_body:
      configure: ".*"
      write: ".*"
      read: ".*"

  tasks:

    - name: check if sensu vhost is present
      uri:
        url: "{{ rabbimq_rest_api_url }}/vhosts/{{ rabbitq_vhost_sensu }}"
        method: GET
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 200,404
        timeout: 10
      register: request_vhost

    - name: debug
      debug:
        var: request_vhost

    - name: create vhost
      uri:
        url: "{{ rabbimq_rest_api_url }}/vhosts/{{ rabbitq_vhost_sensu }}"
        method: PUT
        HEADER_Content-Type: "application/json"
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 204
        timeout: 10
      when:
        - request_vhost.status == 404

    - name: check if sensu user is present
      uri:
        url: "{{ rabbimq_rest_api_url }}/users/{{ rabbitq_user_sensu }}"
        method: GET
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 200,404
        timeout: 10
      register: request_user

    - name: debug
      debug:
        var: request_user

    - name: create user
      uri:
        url: "{{ rabbimq_rest_api_url }}/users/{{ rabbitq_user_sensu }}"
        method: PUT
        HEADER_Content-Type: "application/json"
        body: "{{ rabbitq_user_sensu_request_body | to_json }}"
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 204
        timeout: 10
      when:
        - request_user.status == 404

    - name: check if permissions are present
      uri:
        url: "{{ rabbimq_rest_api_url }}/permissions/{{ rabbitq_vhost_sensu }}/{{ rabbitq_user_sensu }}"
        method: GET
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 200,404
        timeout: 10
      register: request_permission

    - name: debug
      debug:
        var: request_permission

    - name: create permissions
      uri:
        url: "{{ rabbimq_rest_api_url }}/permissions/{{ rabbitq_vhost_sensu }}/{{ rabbitq_user_sensu }}"
        method: PUT
        HEADER_Content-Type: "application/json"
        body: "{{ rabbitmq_permissions_sensu_request_body | to_json }}"
        user: "{{ rabbitmq_auth_user }}"
        password: "{{ rabbitmq_auth_password }}"
        force_basic_auth: yes
        status_code: 204
        timeout: 10
      when:
        - request_permission.status == 404
{{< /codeblock >}}

Ideally, you would create a role for this. Perhaps I will show that in another future post.
