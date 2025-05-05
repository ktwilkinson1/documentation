---
description: Terminate SSL/TLS-encrypted traffic from clients, relieving your upstream
  TCP servers of the computational load.
docs: DOCS-438
title: SSL Termination for TCP Upstream Servers
toc: true
weight: 200
type:
- how-to
---

This article explains how to set up an SSL termination for F5 NGINX Plus. It also shows how to set up a load-balanced group of servers that accept TCP connections. 

## What is SSL Termination?

Termination is when NGINX Plus closes the client connection and forwards the client data over a new unencrypted connection to servers in an upstream group. SSL terminations act as the server-side SSL endpoint for connections with clients. It decrypts requests and encrypts responses that backend servers would usually do. NGINX plus R6 and later performs SSL termination for TCP connections and HTTP connections. 

## Prerequisites

- [NGINX Plus R6]({{< ref "nginx/releases.md#r6 " >}}) or later
- A load-balanced [upstream group]({{< ref "nginx/admin-guide/load-balancer/tcp-udp-load-balancer.md" >}}) with several TCP servers
- SSL certificates and a private key (obtained or self-generated)

## Obtaining SSL Certificates

First, you will need to obtain server certificates and a private key to put on the server. Find Certificates using a trusted certificate authority (CA) or generate it with an SSL library like OpenSSL.

## Configuring NGINX Plus

To configure SSL termination, add the following directives to the NGINX Plus configuration:

### Enabling SSL

To enable SSL, specify the `ssl` parameter of the [listen](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen) directive for the TCP server that passes connections to an upstream server group:

```nginx
stream {

    server {
        listen     12345 ssl;
        proxy_pass backend;
        #...
    }
}
```

### Adding SSL Certificates

To add SSL certificates, specify the path to the certificates with the [ssl_certificate](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate) directive (use PEM format). Specify the path to the private key in the [ssl_certificate_key](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate_key) directive:

```nginx
server {
    #...
    ssl_certificate        /etc/ssl/certs/server.crt;
    ssl_certificate_key    /etc/ssl/certs/server.key;
}
```

[ssl_protocols](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_protocols) and [ssl_ciphers](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_ciphers) directives can limit connections. They can also include only the strong versions and ciphers of SSL/TLS:

```nginx
server {
    #...
    ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers    HIGH:!aNULL:!MD5;
}
```

The [ssl_ciphers](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_ciphers) directive tells NGINX to inform the SSL library which ciphers it prefers.

## Speeding up Secure TCP Connections

Implementing SSL/TLS uses the SSL handshake operation, a series of messages between the client and server that verifies the connection is trusted. This is CPU-intensive and can hinder server performance. The default timeout for the SSL handshake is 60 seconds, but can be redefined with the [ssl_handshake_timeout](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_handshake_timeout) directive. Setting this value too low or too high can result in a handshake failure or a longer wait for the handshake to complete:

```nginx
server {
    #...
    ssl_handshake_timeout 10s;
}
```

### Optimizing the SSL Session Cache

You can improve performance by reducing the number of handshakes. This is done by creating a cache of the session parameters that apply to each SSL/TLS connection. Caching is set with the [ssl_session_cache](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_session_cache) directive and can improve performance:

```nginx
ssl_session_cache;
```

NGINX Plus uses the the cache built in your SSL library by default, the session cache. Such a cache can only be used by one worker process and can cause memory fragmentation. To share the cache among all worker processes, set the `ssl_session_cache` directive to shared . This speeds up later connections because the connection setup information is already known:

```nginx
ssl_session_cache shared:SSL:1m;
```

As a reference, a 1-MB shared cache can hold approximately 4,000 sessions.

NGINX Plus retains cached session parameters for five minutes by default. Increasing the value of the [ssl_session_timeout](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_session_timeout) to several hours reuses cached session parameters and reduces the number of handshakes. This can improve performance, but the cache needs to be bigger to accommodate the larger number of cached parameters. For the 4-hour timeout in the following example, a 20-MB cache is appropriate:appropriate:

```nginx
ssl_session_timeout 4h;
```

If the timeout length is increased, you need a larger cache to store sessions, for example, 20 MB:

```nginx
server {
    #...
    ssl_session_cache   shared:SSL:20m;
    ssl_session_timeout 4h;
}
```

These lines create an in-memory cache of 20 MB to store session information, and instruct NGINX Plus to reuse session parameters from the cache for 4 hours after the moment they were added.

### Session Tickets

Session tickets are an alternative to the session cache. Session information is stored on the client side, eliminating the need for a server-side cache to store session information. When a client resumes interaction with the backend server, it presents the session ticket and re-negotiation is not necessary. Set the [ssl_session_tickets](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_session_tickets) directive to `on`:

```nginx
server {
    #...
    ssl_session_tickets on;
}
```

When using session tickets for an upstream group, each upstream server must be initialized with the same session key. It’s a best practice to change session keys frequently, we recommend that you implement a mechanism to rotate the shared key across all upstream servers:

```nginx
server {
    #...
    ssl_session_tickets on;
    ssl_session_ticket_key /etc/ssl/session_ticket_keys/current.key;
    ssl_session_ticket_key /etc/ssl/session_ticket_keys/previous.key;
}
```

## Complete Example

```nginx
stream {
    upstream stream_backend {
         server backend1.example.com:12345;
         server backend2.example.com:12345;
         server backend3.example.com:12345;
    }

    server {
        listen                12345 ssl;
        proxy_pass            stream_backend;

        ssl_certificate       /etc/ssl/certs/server.crt;
        ssl_certificate_key   /etc/ssl/certs/server.key;
        ssl_protocols         SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers           HIGH:!aNULL:!MD5;
        ssl_session_cache     shared:SSL:20m;
        ssl_session_timeout   4h;
        ssl_handshake_timeout 30s;
        #...
     }
}
```

In this example, the directives in the `server` block instruct NGINX Plus to terminate and decrypt secured TCP traffic from clients and pass it unencrypted to the upstream group `stream_backend` which consists of three servers.

The `ssl` parameter of the [listen](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen) directive instructs NGINX Plus to accept SSL connections. When a clent requests a secure TCP connection, NGINX Plus starts the handshake process, which uses the PEM-format certificate specified by the [ssl_certificate](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate) directive, the certificate’s private key specified by the [ssl_certificate_key](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate_key) directive, and the protocols and cyphers listed by the [ssl_protocols](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_protocols) and [ssl_ciphers](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_ciphers) directives.

As soon as the secure TCP connection is established, NGINX Plus caches the session parameters according to the [ssl_session_cache](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_session_cache) directive. In the example, the session cache is shared between all worker processes (the `shared` parameter), is 20 MB in size (the `20m` parameter), and retains each SSL session for reuse for 4 hours (the [ssl_session_timeout](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_session_timeout) directive).


