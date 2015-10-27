# Security Checks

### Verify SSL certificate chain

```sh
openssl s_client -connect www.yahoo.com:443 -CAfile /usr/local/etc/openssl/cert.pem
```

You should see `verify return:1` for each certificate in the chain.

### Host header injection

[Read about it here](http://carlos.bueno.org/2008/06/host-header-injection.html).

```sh
curl -i --header "Host: evilsite.com" https://www.yahoo.com
```

Your site is vulnerable if `evilsite.com` appears in the results.

### SPF

Check if your SPF record is valid. [Enter your domain here](http://www.kitterman.com/spf/validate.html).

### DNSSEC

Very few sites have this right now.

```sh
dig pir.org +dnssec
```

[See how to interpret the results](http://docs.menandmice.com/display/MM/How+to+test+DNSSEC+validation).
