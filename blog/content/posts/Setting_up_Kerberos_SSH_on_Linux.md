---
title: "Setting-up-Kerberos-SSH-on-Linux"
date: 2021-06-27T14:37:03-05:00
draft: false
tags: [
	"Linux",
    "CentOS",
    "Kerberos"
]
---
![kerberos-logo](https://web.mit.edu/kerberos/images/dog-ring.jpg)

[Kerberos](https://web.mit.edu/kerberos/#what_is) is a mature computer network security protocol that authenticates across hosts inside of a untrusted network. It is considered to be quantum-secure and is useful for securing Active Directory, NFS, Samba, email, and ssh.

{{< youtube id="qW361k3-BtU" title="Taming Kerberos - Computerphile" >}}

This post will be covering how to deploy a Kerberos server and client for secure ssh authentication on Fedora/RHEL systems. 

## **Prerequisites**
- 2 Linux machines
    - shared network
    - root access
- DNS server for resolving hostnames (optional) 

## **Server setup**
1. ### Dependencies
    Install kerberos server and workstation packages.
    ```bash {linenos=false}
    yum update -y
    yum install krb5-libs krb5-server krb5-workstation -y
    ```
2. ### KDC configuration
    Edit /var/kerberos/krb5kdc/kdc.conf and set realm.
    ```vim {hl_lines=[5]} 
    [kdcdefaults]
     kdc_ports = 88
     kdc_tcp_ports = 88
    [realms]
     EXAMPLE.COM = {
      master_key_type = aes256-cts
      acl_file = /var/kerberos/krb5kdc/kadm5.acl
      dict_file = /usr/share/dict/words
      admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
      supported_enctypes = aes256-cts:normal aes128-cts:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal
     }
    ```
3. ### KRB5 configuration
    Edit /etc/krb5.conf and set domain-realm mapping. Lines 22-23 should be set to the server FQDN.
    ```vim {hl_lines=[18,"21-23", "27-28"]}
    # Configuration snippets may be placed in this directory as well
    includedir /etc/krb5.conf.d/

    includedir /var/lib/sss/pubconf/krb5.include.d/
    [logging]
     default = FILE:/var/log/krb5libs.log
     kdc = FILE:/var/log/krb5kdc.log
     admin_server = FILE:/var/log/kadmind.log

    [libdefaults]
     dns_lookup_realm = false
     ticket_lifetime = 24h
     renew_lifetime = 7d
     forwardable = true
     rdns = false
     pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
     default_ccache_name = KEYRING:persistent:%{uid}
     default_realm = EXAMPLE.COM

    [realms]
     EXAMPLE.COM = {
      kdc = host.example.com
      admin_server = host.example.com
     }

    [domain_realm]
     .EXAMPLE.COM = EXAMPLE.COM
     EXAMPLE.COM = EXAMPLE.COM
    ```
4. ### Create KRB5 database
    Create KRB5 database and set a strong password. 
    ```bash {linenos=false}
    sudo kdb5_util create -r EXAMPLE.COM -s
    ```
5. ### KRB5 database ACL
    Edit /var/kerberos/krb5kdc/kadm5.acl and change realm.
    ```vim {linenos=false}
    */admin@EXAMPLE.COM     *
    ```
6. ### Create first principal root user.
    Create principal and set password
    ```bash {linenos=false}
    sudo kadmin.local
    addprinc root/admin 
    quit
    ```
7. ### Start KRB5 services and enable on boot
    ```bash {linenos=false}
    sudo systemctl enable --now krb5kdc kadmin
    ```
8. ### Test KRB5 
    ```bash {linenos=false}
    sudo kinit root/admin
    sudo klist 
    ```
9. ### Enable firewall ports
    ```bash {linenos=false}
    sudo firewall-cmd --add-service=kerberos --permanent
    sudo firewall-cmd --add-service=kadmin --permanent
    sudo firewall-cmd --add-service=kpasswd --permanent
    sudo firewall-cmd --reload
    sudo firewall-cmd --list-services 
    ```
10. ### Configure SSH server
    Edit /etc/ssh/sshd_config and disable PasswordAuthentication/PubkeyAuthentication optionally. 
    ```vim
    ...

    KerberosAuthentication yes
    KerberosOrLocalPasswd no
    KerberosTicketCleanup yes

    ...
   
    GSSAPIAuthentication yes
    GSSAPICleanupCredentials no

    ...

    ```
    Restart the SSH server
    ```bash {linenos=false}
    sudo systemctl restart sshd
    sudo systemctl status sshd
    ```
11. ### Add host principal if ssh to KDC is needed
    Generate the principal and write keyentry to use for GSSAPI.
    ```bash {linenos=false}
    sudo kdamin
    addprinc -randkey host/fqdn
    ktadd host/fqdn
    ```

## **Client**

1. ### Dependencies
    ```bash {linenos=false}
    sudo dnf install krb-workstation -y
    ```
2. ### Copy /etc/krb5.conf from KDC
    Edit /etc/krb5.conf and make sure the file matches the kdc server.
    ```vim {hl_lines=[18,"21-23", "27-28"]}
    # Configuration snippets may be placed in this directory as well
    includedir /etc/krb5.conf.d/

    includedir /var/lib/sss/pubconf/krb5.include.d/
    [logging]
     default = FILE:/var/log/krb5libs.log
     kdc = FILE:/var/log/krb5kdc.log
     admin_server = FILE:/var/log/kadmind.log

    [libdefaults]
     dns_lookup_realm = false
     ticket_lifetime = 24h
     renew_lifetime = 7d
     forwardable = true
     rdns = false
     pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
     default_ccache_name = KEYRING:persistent:%{uid}
     default_realm = EXAMPLE.COM

    [realms]
     EXAMPLE.COM = {
      kdc = host.example.com
      admin_server = host.example.com
     }

    [domain_realm]
     .EXAMPLE.COM = EXAMPLE.COM
     EXAMPLE.COM = EXAMPLE.COM
    ```
3. ### Create principal user from either Client/Server kadmin
    Create user either using kadmin from client/server and set password
    ```
    sudo kadmin
    addprinc client
    ```
4. ### Allow GSSAPI for ssh
    Edit /etc/ssh/ssh_config which will change settings for all users or make a new file
    ~/.ssh/config for the current user. Specify in the host section which hosts the rules will apply
    to.
    ```vim
    Host *.fqdn
       GSSAPIAuthentication yes
       GSSAPIDelegateCredentials no  #enable for key forwarding
    ```
5. ### Intialize ticket
    Use the user on the client machine that matches the principal user that was created earlier.
    ```bash {linenos=false}
    kinit 
    klist 
    ```
6. ### Connect using ssh
    Ensuring that the remote destination has a corresponding principal host and key-entry in /etc/krb5.keytab.
    ```bash {linenos=false}
    kinit
    klist
    ssh client@fqdn
    ```

## **Notes**

- Make sure ssh destinations have ssh configured and appropriate principal host/user and key entries configured.
- After changes to entries make sure to request new keys from KDC on the client.
- [Less secure encryption](https://web.mit.edu/~kerberos/krb5-1.19/doc/admin/enctypes.html) in /var/kerberos/krb5kdc/krb5.conf can be disabled
    
## **References**
- <https://web.mit.edu/kerberos/>
- <https://www.dbaplus.ca/2020/10/install-and-configure-kerberos.html>
- <https://www.confluent.io/blog/containerized-testing-with-kerberos-and-ssh/>
- <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/configuring_a_kerberos_5_client>
- <https://web.mit.edu/~kerberos/krb5-1.19/doc/admin/enctypes.html>
