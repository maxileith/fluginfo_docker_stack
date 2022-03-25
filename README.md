# Installationsanleitung

Diese Anleitung erklärt Ihnen Schritt für Schritt, wie Sie die Fluginfo Anwendung auf einem Linux-Server installieren.

## Voraussetzungen

-   Ubuntu Server 20.04 LTS\* ([Download](https://ubuntu.com/download/server))
-   Docker Engine ([Installationsanleitung](https://docs.docker.com/engine/install/ubuntu/))
-   Docker-Compose ([Installationsanleitung](https://docs.docker.com/compose/install/#install-compose))

<small>\* Empfehlung, andere Linux-Distributionen sind ebenfalls kompatibel.</small>

## Laden der Container

Da die Container des Front- und Backends nicht in öffentlichen Docker-Registries verfügbar sind, müssen diese erst manuell geladen werden. Im Git-Repository werden zu jedem Release die Container als Tar-Ball (_.tar.gz_ Datei) hinterlegt:

-   [Frontend v1.0](https://git.leith.de/fluginfo/frontend/releases/tag/v1.0)
-   [Backend v1.0](https://git.leith.de/fluginfo/backend/releases/tag/v1.0)

Die Tar-Balls lädt man entweder direkt auf dem Server herunter, oder auf dem lokalen PC und überträgt diese mit SFTP. Eine Anleitung dazu finden Sie [hier](https://www.ionos.com/help/hosting/setting-up-and-managing-ftp-access/transferring-files-with-filezilla-using-sftp/).

Sobald die Dateien auf dem Server liegen, können die Container geladen werden:

```bash
sudo docker load -i /path/to/maxileith_fluginfo-frontend_v1.0.tar.gz
sudo docker load -i /path/to/maxileith_fluginfo-backend_v1.0.tar.gz
```

## Docker Stack aufsetzen

Die Definition des Docker-Stacks befindet sich in diesem Repository, weshalb dieses zuerst geklont wird:

```bash
sudo git clone https://git.leith.de/fluginfo/docker-stack.git /opt/fluginfo && cd /opt/fluginfo
```

Bevor die Beispielkonfiguration angepasst wird, wird diese kopiert:

```bash
sudo cp docker-compose.example.yaml docker-compose.yaml
sudo cp data/traefik.example.yml data/traefik.yml
```

## Docker-Compose

### Umgebungsvariablen

Mit `sudo nano docker-compose.yaml` können Sie nun die Konfiguration anpassen:

**Frontend:**

| Variable                  | Bedeutung                                | Wertebereich |
| ------------------------- | ---------------------------------------- | ------------ |
| FLUGINFO_BACKEND_BASE_URL | URL unter der das Backend erreichbar ist | Valide URLs  |

**Backend:**

| Variable                            | Bedeutung                                                                                                                                                            | Wertebereich                        |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| FLUGINFO_BACKEND_AMADEUS_API_SECRET | Wird im [Amadeus Self-Service Workspace](https://developers.amadeus.com/my-apps) ausgestellt.                                                                        | `string`                            |
| FLUGINFO_BACKEND_AMADEUS_API_KEY    | Wird im [Amadeus Self-Service Workspace](https://developers.amadeus.com/my-apps) ausgestellt.                                                                        | `string`                            |
| FLUGINFO_BACKEND_AMADEUS_PROD       | Definiert, ob die Produktivversion von Amadeus verwendet wird. Anmerkung: Credentials der Testumgebung funktionieren nicht für die Produktivumgebung und vice versa. | `true` \| `false`                   |
| FLUGINFO_BACKEND_AIRHEX_API_KEY     | Siehe [hier](https://airhex.com/pricing/).                                                                                                                           | `string` leer falls nicht verfügbar |
| FLUGINFO_BACKEND_DEBUG              | Debug-Modus An / Aus. Im Debug-Modus werden Stack-Traces in der Konsole ausgegeben sowie die meisten Cache-Funktionalitäten deaktiviert.                             | `true` \| `false`                   |
| FLUGINFO_BACKEND_HOSTNAME           | Domain des Backends                                                                                                                                                  | valide Domain                       |
| FLUGINFO_BACKEND_FRONTEND_HOSTNAME  | Domain des Frontends                                                                                                                                                 | valide Domain                       |

### Labels

Außerdem müssen die Labels angepasst werden, damit Traefik den Traffic korrekt weiterleitet.

**Traefik:**

| Label                             | Bedeutung                                                   |
| --------------------------------- | ----------------------------------------------------------- |
| traefik.http.routers.traefik.rule | Domain, unter der das Dashboard von Traefik erreichbar ist. |

**Frontend:**

| Label                                       | Bedeutung                                                                                                                          |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| traefik.http.routers.fluginfo-frontend.rule | Domain, unter der das Frontend erreichbar ist. Sollte identisch zu der Umgebungsvariable `FLUGINFO_BACKEND_FRONTEND_HOSTNAME` sein |

**Backend:**

| Label                                      | Bedeutung                                                                                                                |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| traefik.http.routers.fluginfo-backend.rule | Domain, unter der das Backend erreichbar ist. Sollte identisch zu der Umgebungsvariable `FLUGINFO_BACKEND_HOSTNAME` sein |

## Traefik

**traefik.yml:**

Mit `sudo nano data/traefik.yml` können Sie die E-Mail, die zum automatisierten Ausstellen von SSL-Zertifikaten mittels Let's Encrypt verwendet wird, unter _certificatesResolvers -> le -> acme -> email_ ändern.

**acme.json:**

In der Datei `data/acme.json` werden SSL-Zertifikate gespeichert. Aus Sicherheitsgründen werden die Berechtigungen angepasst:

```bash
sudo chmod 600 data/acme.json
```

**.htpasswd:**

In der Datei `data/.htpasswd` wird das Passwort zum Sichern des Traefik-Dashboards gespeichert. Um das Passwort zu setzen installieren Sie zunächst die Apache2 Utils:

```bash
sudo apt install apache2-utils
```

Anschließend können Sie Credentials anlegen, beispielsweise für einen Nutzer `admin`:

```bash
sudo htpasswd -cB data/.htpasswd admin
```

## DNS

Nun müssen nur noch die korrekten DNS-Einträge gesetzt werden.

**Annahmen:**

-   Die IPv4 des Servers ist `123.123.123.123`
-   Die IPv6 des Servers ist `2001::1`
-   Die Frontend-Domain ist `domain.tld`
-   Die Backend-Domain ist `api.domain.tld`
-   Die Traefik-Dashboard-Domain ist `traefik.domain.tld`
-   `mail@example.org` ist eine Kontaktmöglichkeit bei unbefugten Versuchen ein SSL-Zertifikat auszustellen

**Zonendatei:**

Es kann bis zu 48 Stunden dauern, bis die Records über alle DNS-Server propagiert sind.

```
@                   1800  IN  A      123.123.123.123
@                   1800  IN  AAAA   2001::1
@                   1800  IN  CAA    128  issuewild  ";"
@                   1800  IN  CAA    128  issue      "letsencrypt.org"
@                   1800  IN  CAA    128  iodef      "mailto:mail@example.org"
api.domain.tld      1800  IN  CNAME  domain.tld.
traefik.domain.tld  1800  IN  CNAME  domain.tld.
```

Anmerkung: Die CAA-Records sind optional und dienen dazu die Zertifizierungsstelle von Lets's Encrypt als einzige zur Ausstellung von SSL-Zertifikaten zu befugen. Wildcard-Zertifikate werden komplett untersagt. Bei Verstößen, bei denen sich ein Angreifer versucht ein Zertifikat einer unbefugten Zertifizierungsstelle ausstellen zu lassen, kann die Zertifizierungsstelle die E-Mail `mail@example.org` kontaktieren.

## Starten

Nun kann Fluginfo gestartet werden:

```bash
sudo docker-compose up
```
