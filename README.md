# <img src=".gitlab/ansible.png" alt="ansible" height="30"/> haproxy

Ansible Role instalująca i konfigurująca **HAProxy** jako load balancer HTTP/HTTPS.

---

## Wspierane platformy

- Ubuntu / Debian
- RedHat / CentOS / AlmaLinux / Rocky
- Alpine Linux

---

## Zmienne

Cała konfiguracja przekazywana jest przez słownik `in_haproxy`. Poniżej kluczowe sekcje:

| Sekcja | Opis |
|---|---|
| `global` | Tuning globalny: `maxconn`, `log`, `extra_lines` |
| `defaults` | Domyślne timeouty, tryb, opcje |
| `stats` | Panel statystyk (domyślnie `127.0.0.1:8404/stats`) |
| `frontends` | Lista frontendów — `name`, `bind`, `mode`, `default_backend`, `acls`, `use_backends`, `options`, `extra_lines` |
| `backends` | Lista backendów — `name`, `mode`, `balance`, `options`, `http_check`, `tcp_check`, `servers`, `extra_lines` |

Każdy serwer w backendzie: `name`, `address`, `port`, `params` (np. `"check"`).

---

## Przykład użycia

Typowy scenariusz — usługi z localhost wystawione na VIP (eth0):

```yaml
- hosts: haproxy_nodes
  roles:
    - role: haproxy
      vars:
        in_haproxy:
          frontends:
            - name: http-in
              bind: "{{ ansible_facts['default_ipv4']['address'] }}:80"
              mode: http
              default_backend: app_servers

            - name: https-in
              bind: "{{ ansible_facts['default_ipv4']['address'] }}:443 ssl crt /etc/haproxy/certs/site.pem"
              mode: http
              default_backend: app_servers
              extra_lines:
                - "http-request set-header X-Forwarded-Proto https"

          backends:
            - name: app_servers
              mode: http
              balance: roundrobin
              http_check:
                send: "meth GET uri /health"
                expect: "status 200"
              servers:
                - name: app1
                  address: 127.0.0.1
                  port: 8080
                  params: "check"
```

Pełny przykład: [examples/localhost-to-vip.yml](examples/localhost-to-vip.yml)

## Architektura roli

### Flow wykonania tasków

```mermaid
flowchart TD
    START([Start]) --> INSTALL["Install haproxy<br/><i>ansible.builtin.package</i>"]
    INSTALL --> CONFDIR["Ensure config directory<br/>/etc/haproxy/"]
    CONFDIR --> SSL_CHECK{{"Frontend<br/>używa SSL?"}}
    SSL_CHECK -- Tak --> CERTSDIR["Ensure certs directory<br/>/etc/haproxy/certs/"]
    SSL_CHECK -- Nie --> TEMPLATE
    CERTSDIR --> TEMPLATE["Render haproxy.cfg<br/><i>ansible.builtin.template</i><br/>+ validate"]
    TEMPLATE -- "notify" --> RELOAD[/"Reload haproxy"/]
    TEMPLATE --> SYSTEMD_CHECK{{"systemd?"}}
    SYSTEMD_CHECK -- Tak --> ENABLE["Enable & start<br/><i>ansible.builtin.systemd</i>"]
    SYSTEMD_CHECK -- Nie --> SKIP["Skip service mgmt<br/><i>debug msg</i>"]
    ENABLE --> DONE([Done])
    SKIP --> DONE

    style SSL_CHECK fill:#f9f,stroke:#333
    style SYSTEMD_CHECK fill:#f9f,stroke:#333
    style RELOAD fill:#ff9,stroke:#333
```

### Struktura generowanej konfiguracji

```mermaid
flowchart LR
    subgraph haproxy.cfg
        direction TB
        G["<b>global</b><br/>maxconn, log, daemon"]
        D["<b>defaults</b><br/>mode, timeouty, opcje"]
        S["<b>listen stats</b><br/>bind, uri, auth<br/><i>(opcjonalnie)</i>"]
        F1["<b>frontend</b> http-in<br/>bind, mode, ACL,<br/>default_backend"]
        F2["<b>frontend</b> https-in<br/>bind ssl, mode,<br/>extra_lines"]
        B1["<b>backend</b> app_servers<br/>balance, http_check,<br/>servers"]
        G --> D --> S --> F1 --> F2 --> B1
    end
```

### Schemat sieciowy (typowy use-case)

```mermaid
flowchart LR
    CLIENT(("Klient<br/>Internet")) -->|":80 / :443"| FE["<b>HAProxy</b><br/>Frontend<br/>VIP (eth0)"]
    FE -->|"roundrobin"| BE1["Backend<br/>127.0.0.1:8080<br/>app1"]
    FE -->|"roundrobin"| BE2["Backend<br/>127.0.0.1:8081<br/>app2"]
    FE -.->|":8404/stats"| STATS["Stats panel<br/>127.0.0.1:8404"]

    style FE fill:#4a9,stroke:#333,color:#fff
    style CLIENT fill:#69c,stroke:#333,color:#fff
```

---

## Contributions

Jeśli masz pomysły na ulepszenia, zgłoś problemy, rozwidl repozytorium lub utwórz Merge Request. Wszystkie wkłady są mile widziane!
[Contributions](CONTRIBUTING.md)

---

## License

[Licencja](LICENSE) oparta na zasadach Creative Commons BY-NC-SA 4.0, dostosowana do potrzeb projektu.

---

## Author Information

| ![Maciej Rachuna](https://gitlab.com/uploads/-/system/user/avatar/8161705/avatar.png?width=120px) |
|---------------------------------------------------------------------------------------------------|
| [Maciej Rachuna](https://gitlab.commrachuna)                                                      |
