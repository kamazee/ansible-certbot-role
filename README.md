# Ansible Role for Running Certbot on Debian 10

This is an ansible role to install certbot and issue certificates using HTTP-01 challenge.

## Disclaimer
1. It's meant to be reusable across my own projects and might not be that useful to broader audience; however, it might work as an inspiration for your own role.
2. So far, it's only tested with Ansible 2.10 on Debian 10 ("Buster"). If you want to expand the range of supported Ansible versions and/or Linux distributions, feel free to fork and work on it as I don't plan on doing it.

The idea behind is explained [in my blog](https://kurilo.me/en/blog/ansible_certbot_nginx/).

In case you still feel like trying it, read further.

## How To

This role is intended to be included into your existing list of tasks that maintains your webserver configuration, be it in a role or directly in a playbook. I use nginx; although I tried to make the role to be not bound to a specific kind of webserver, there's a chance it is.

So, I assume there's a task that adds a virtual server configuration (`server` in nginx); here is what has to be done in order for this role to work.

1. Configure a web root for ACME challenge.
2. Create a webserver configuration that uses https and references SSL/TLS certificates as if they were already there (use a variable to specify the expected certificate's and key's filename); expect a symlink to the certificate and key to appear there.
3. Include the role and pass few values into it, so it can first create a temporary self-signed certificate so that a webserver can start (or restart, or reload) and handle ACME HTTP challenge; it will set symlinks to certificate and key at your paths that initially point to self-signed certificates and then replace the destinations of those links with real key and certificate signed by Letsencrypt.

**The role will flush ansible handlers** before working with ACME challenge in order to make sure that webserver is capable of handling HTTP challenge. When a real certificate is obtained, it will replace symlink that pointed to a self-signed certificate and reload your webserver (`systemctl reload nginx` by default; the command can be customized).

Point 3 mentions that the role expects few values to be passed in when including; here they are:

| Variable Name | Type | Meaning |
| --- | --- | --- |
| `certbot_letsencrypt_email` | `string` | Email to register Letsencrypt account with |
| `certbot_acme_http_challenge_root` | `string` | Directory in which challenge file `.well-known/acme-challenge/<TOKEN>` will be created. Note that `certbot` creates `.well-known/acme-challenge`, too, so this directory will contain not the token directly but the `.well-known` directory |
| `certbot_domains` | `string[]` (`list` of `string`) | Domains the certificate should be issued for; virtual servers for each of the domains must either redirect to the location that exposes the challenge token (`/.well-known/acme-challenge/<TOKEN>`) or expose it themselves |
| certbot_cert_name | `string` | Name of the certificate. Defines the name under which the certificate is listed in `certbot certificates` output and the location of the certificates and renewal configurations under certbot config root |
| certbot_selfsigned_root | `string` | Path where self-signed temporary certificates will be placed; note that they are not removed when symlink is switched to the real certificates |
| certbot_cert_link_dest | `string` | Filename; this is where your webserver expects the certificate to be; this will be a symlink to the certificate |
| certbot_key_link_dest | `string` | Filename; this is where your webserver expects the key to be; this will be a symlink to the actual key |
| certbot_issue_real_cert | `bool` | Whether to issue a real certificate or not (`false` makes sense in a testing environment where self-signed cert is sufficient, and where it might be challenging to pass an http-01 challenge because inbound connections are cut off) |


### Usage example

Assuming you have something like this in your existing role/playbook:

```yaml
---
- name: Put acme challenge config
  template:
    src: "acme-challenge.conf"
    dest: /etc/nginx/acme-challenge.conf
  notify:
    - Reload nginx

- name: Put site configuration
  template:
    src: example.com.conf.j2
    dest: /etc/nginx/sites-available/example.com.conf.j2
  notify:
    - Reload nginx

# Just add SSL configuration into your example.conf.j2,
# and now it's time for certbot role to kick in:
- name: https for example.com
  include_role:
    name: kamazee.certbot
  vars:
    certbot_domains:
      - "www.{{ example_com_host }}"
      - "{{ example_com_host }}"
    certbot_cert_name: "{{ exmaple_com_certbot_cert_name }}"
    certbot_cert_link_dest: "{{ exmaple_com_certbot_cert_link_dest }}"
    certbot_key_link_dest: "{{ exmaple_com_certbot_key_link_dest }}"
```

Something like this will be in the variables:
```yaml
---
exmaple_com_certbot_cert_name: "example_com"
exmaple_com_certbot_cert_link_dest: /etc/nginx/ssl/example.com/cert.pem
exmaple_com_certbot_key_link_dest: /etc/nginx/ssl/example.com/cert.key
certbot_acme_http_challenge_root: /var/www
```

Here is my `templates/acme-challenge.conf`:
```
location /.well-known/acme-challenge {
    root {{ acme_challenge_root }};
    autoindex off;
}
```

`templates/example.conf.j2` might look like this:
```
server {
    listen 80;
    server_name www.{{ exmaple_com_host }} {{ exmaple_com_host }};

    include "acme-challenge.conf";

    return 301 https://www.{{ exmaple_com_host }}$request_uri;
}

server {
    listen 443 ssl;
    server_name www.{{ exmaple_com_host }};

    ssl_certificate {{ exmaple_com_certbot_cert_link_dest }};
    ssl_certificate_key {{ exmaple_com_certbot_key_link_dest }};

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    root {{ exmaple_com_public_root }};
    index index.html;
    autoindex off;

    error_log {{ exmaple_com_nginx_error_log }};
    access_log {{ exmaple_com_nginx_access_log }};

    include "acme-challenge.conf";
}

server {
    listen 443 ssl;
    server_name {{ exmaple_com_host }};

    ssl_certificate {{ exmaple_com_certbot_cert_link_dest }};
    ssl_certificate_key {{ exmaple_com_certbot_key_link_dest }};

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    include "acme-challenge.conf";

    return 301 https://www.{{ exmaple_com_host }}$request_uri;
}
```

Although boilerplate doesn't look small, you likely already have it all somewhere, so the trick is to just properly put the role.
