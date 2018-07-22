---
layout: post
title: "Creating a private docker registry without a DNS domain (Ip address only)"
author: "Zakaria"
comments: true
---


# Background

If you are developping applications with multiple teams involved or using tools like Kubernetes, then making use of a docker registry is a must. If you have enough resources, you can look into solutions like [Quay.io](https://quay.io/) or [Docker Enterprise](https://www.docker.com/pricing), otherwise you will need set up your own registry. The minimum requirements are a host (1GB of memory should be enough) with a public ip address. The proper way of doing it would be to register a domain and to obtain a signed certificate from a certificate authority (CA) for allowing access over https (http is not allowed). If you want to save more bucks, then it is possible to use the ip address only with a self-signed certificate.  

# Steps

The set up steps are already described in docker documentation, so no need reinvent the wheel. 

 1. Follow the steps described [here](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04) to create a self-signed certificate 
   
 2. Follow the steps described in docker documentation to [create](https://docs.docker.com/registry/deploying/) and [deploy](https://docs.docker.com/registry/insecure/#use-self-signed-certificates) the registry using the created certificate. 
   
 3. **Important**: To avoid x509 issues, you need to edit the `/etc/ssl/openssl.cnf` on the registry host (not inside the container), and alter the [v3_ca] section to add your ip address (instead of `1.2.3.4`):
```
[ v3_ca ]
subjectAltName = IP:1.2.3.4
```
Reference: [https://github.com/docker/distribution/issues/948](https://github.com/docker/distribution/issues/948)

1. Finally, distribute the certificate (ca.crt file) to all the developers and machines that need to access the registry. The certificate need to be copied to `/etc/docker/certs.d/1.2.3.4:5000/ca.crt` ( with `1.2.3.4` being your host address and 5000 being the registry port). The folder usually does not exist, so a `mkdir -p /etc/docker/certs.d/1.2.3.4:5000/` can do the trick. 

# Take away

Setting up a docker registry using an ip address and a self-signed certificate can be seen as a quick and dirty way of doing things, but it can help you get going with minimal effort and budget. 