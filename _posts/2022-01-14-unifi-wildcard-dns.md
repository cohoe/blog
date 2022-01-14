---
title: "Wildcard DNS Record with UniFi Network"
date: 2022-01-14
published: true
---

I use a [UniFi Security Gateway](https://www.ui.com/unifi-routing/usg/) as the core of my home network, which also acts as my local DNS resolver. I also use [Traefik](https://traefik.io/) as a convenient application router for any application stacks (services) I happen to be playing with. If I wanted to deploy some service with [Docker Compose](https://docs.docker.com/compose/) such as Grafana or even a dummy webapp I would have to:

1.  Annotate my containers in a `docker-compose.yaml` appropriately to setup the HTTP listener.
    ```
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host('grafana.example.grantcohoe.com')"
    ```
    Note: The backticks in this example have been replaced with single quotes due to a Github Pages Jekyll issue.

2.  Edit my UniFi `config.gateway.json` to add a static DNS record associating that name with the host IP.
    ```json
    {
      "service": {
        "dns": {
          "forwarding": {
            "options": [
              "host-record=grafana.example.grantcohoe.com,169.254.169.254"
            ]
          }
        }
      }
    }
    ```
    Note: This example has been simplied for brevity.

3.  Trigger a Force Reprovision of my USG via the UniFi Controller to pick up the change and pray I didn't make a syntax error.

This is two steps too many for my liking. I shouldn't have to edit a file and do a manual action just to tinker around with something. Fortunately I have a wildcard SSL certificate installed on Traefik from my local CA so I don't need to add a Step 4 of issueing a certificate. So lets fix this nonsense! Enter the Wildcard DNS record concept. You can go [read about it on Wikipedia](https://en.wikipedia.org/wiki/Wildcard_DNS_record). tldr:
```
# The following record resolves example.com and any host
# within example.com to the IP address of 127.0.0.1.
*.example.com. IN A 127.0.0.1
```

This is commonly done as a pattern with Kubernetes ingress controllers. Anywho... how do we get the UniFi to service a wildcard record? UniFi runs [Dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) under the hood. And [someone on the internet](https://qiita.com/bmj0114/items/9c24d863bcab1a634503) had a similar-enough question. Note the `address=/` portion of the configuration directive. So combine that with the `config.gateway.json` from UniFi and we should be in business.

One line change and one final force reprovision were needed to make this work.
```json
{
  "service": {
    "dns": {
      "forwarding": {
        "options": [
          "address=/apps.example.grantcohoe.com/169.254.169.254"
        ]
      }
    }
  }
}
```

Now I get a DNS record for anything under `apps.example.grantcohoe.com` domain.
```
~ # dig foo.apps.example.grantcohoe.com

; <<>> DiG 9.16.24-RH <<>> foo.apps.example.grantcohoe.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31613
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;foo.apps.example.grantcohoe.com.	IN	A

;; ANSWER SECTION:
foo.apps.example.grantcohoe.com.	0 IN	A	169.254.169.254

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Fri Jan 14 15:57:15 EST 2022
;; MSG SIZE  rcvd: 75
```
Note: The astute will have noticed that `SERVER: 127.0.0.53` means I'm using [Systemd-Resolved](https://fedoramagazine.org/systemd-resolved-introduction-to-split-dns/). Get used to it.

The last piece needed was to issue a new wildcard SSL certificate from my internal CA for the wildcard domain for Traefik to serve. Now deploying new apps and having them available is a breeze!
