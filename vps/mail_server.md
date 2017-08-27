# sakura vps mail sever memo
## spec
### os
CentOS7 x86_64 ( sakura vps standart os plan)

### package
postfix
dovecot
Let's encrypt

### port

SMPT  : 25
SMTPS : 465
IMAPS : 993
POP3S : 995

## Let's encrypt

```
# cd /usr/local/
# git clone https://github.com/letsencrypt/letsencrypt 
# cd letsencrypt/
# ./letsencrypt-auto --help 
# ./letsencrypt-auto certonly --standalone \
    -d mail.example.com \
    -m sample@example.com \
    --agree-tos 
```

crontab
```
00 05 01 * * /usr/local/letsencrypt/letsencrypt-auto certonly --standalone -d mail.ekoh.jp --renew-by-default && /bin/systemctl reload postfix && /bin/systemctl reload dovecot 
```

## postfix
### /etc/postfix/main.cf
```
queue_directory = /var/spool/postfix
command_directory = /usr/sbin
daemon_directory = /usr/libexec/postfix
data_directory = /var/lib/postfix
mail_owner = postfix
myhostname = mail.ekoh.jp
mydomain = ekoh.jp
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
local_recipient_maps = proxy:unix:passwd.byname $alias_maps
unknown_local_recipient_reject_code = 550
relay_domains = $mydestination
relayhost =
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
home_mailbox = Maildir/
smtpd_banner = $myhostname ESMTP
debug_peer_level = 2
debugger_command =
	 PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
	 ddd $daemon_directory/$process_name $process_id & sleep 5
sendmail_path = /usr/sbin/sendmail.postfix
newaliases_path = /usr/bin/newaliases.postfix
mailq_path = /usr/bin/mailq.postfix
setgid_group = postdrop
html_directory = no
manpage_directory = /usr/share/man
sample_directory = /usr/share/doc/postfix-2.10.1/samples
readme_directory = /usr/share/doc/postfix-2.10.1/README_FILES
mynetworks_style = host
disable_vrfy_command = yes
mailbox_size_limit = 204800000
message_size_limit = 5120000
smtpd_sender_restrictions =
    reject_rhsbl_sender zen.spamhaus.org
    reject_unknown_sender_domain
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
broken_sasl_auth_clients = yes
smtpd_use_tls = yes
smtp_tls_security_level = may
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.ekoh.jp/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.ekoh.jp/privkey.pem
smtpd_tls_loglevel = 1
smtpd_tls_received_header = yes
smtpd_tls_session_cache_database = btree:/var/lib/postfix/smtpd_scache
smtpd_tls_session_cache_timeout = 3600s
smtpd_recipient_restrictions =
    permit_mynetworks
    permit_sasl_authenticated
    reject_unauth_destination
```
### /etc/postfix/master.cf

```
smtp      inet  n       -       n       -       -       smtpd
smtps     inet  n       -       n       -       -       smtpd
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
bounce    unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       bounce
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       n       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
```

## dovecot

### /etc/dovecot/dovecot.conf
```
protocols = imap pop3
dict {
  #quota = mysql:/etc/dovecot/dovecot-dict-sql.conf.ext
  #expire = sqlite:/etc/dovecot/dovecot-dict-sql.conf.ext
}
!include conf.d/*.conf
!include_try local.conf
```

### /etc/dovecot/conf.d/10-master.conf
```
service imap-login {
  inet_listener imap {
    port = 0
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}
service pop3-login {
  inet_listener pop3 {
    port = 0
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}
service lmtp {
  unix_listener lmtp {
    #mode = 0666
  }
    # Avoid making LMTP visible for the entire internet
    #address =
    #port =
}
service imap {
}
service pop3 {
}
service auth {
  unix_listener auth-userdb {
    #mode = 0666
    #user =
    #group =
  }
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
service auth-worker {
}
service dict {
  unix_listener dict {
    #mode = 0600
    #user =
    #group =
  }
}
```

### /etc/dovecot/conf.d/10-auth.conf
```
disable_plaintext_auth = no
auth_mechanisms = plain login
!include auth-system.conf.ext
```

### /etc/dovecot/conf.d/10-ssl.conf
```
ssl = yes
ssl_cert = </etc/letsencrypt/live/mail.ekoh.jp/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.ekoh.jp/privkey.pem
```


### /etc/dovecot/conf.d/10-mail.conf

```
 mail_location = maildir:~/Maildir
namespace inbox {
  inbox = yes
}
first_valid_uid = 1000
mbox_write_locks = fcntl
```


### etc/dovecot/conf.d/10-logging.conf
```
log_path = /var/log/dovecot/dovecot.log
plugin {
}
```

## log
```
# mkdir /var/log/dovecot
# mkdir /var/log/mail
# vi /etc/rsyslog.conf
mail.*                                                  -/var/log/mail/maillog
# systemctl restart rsyslog
# rm /var/log/maillog*

# vi /etc/logrotate.d/syslog
- /var/log/maillog

# vi /etc/logrotate.d/maillog
/var/log/mail/maillog {
    daily
    missingok
    dateext
    rotate 60
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}

# vi /etc/logrotate.d/dovecot
/var/log/dovecot/dovecot.log {
    daily
    missingok
    dateext
    rotate 60
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /var/run/dovecot/master.pid 2>/dev/null` 2> /dev/null || true
    endscript
}
```


## create mail account

```
# useradd -s /sbin/nologin in4
# passwd in4

```
