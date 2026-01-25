# Dedicated User Pattern für Docker Container

## Übersicht

Das `compose_hull` Framework unterstützt die Erstellung und Verwaltung dedizierter System-User für Docker Container. Dies verbessert die Sicherheit, indem Container nicht mehr unter dem admin-User (z.B. `daniel`) laufen, sondern mit eigenen, unprivilegierten System-Usern.

## Vorteile

1. **Security**: Jeder Service läuft mit minimalen Berechtigungen
2. **Isolation**: Container-Prozesse sind von Host-User-Accounts isoliert
3. **Least Privilege**: Services haben nur Zugriff auf ihre eigenen Daten
4. **Audit Trail**: Klare Trennung von Service-Aktivitäten auf dem Host

## Konfiguration

### Basis-Konfiguration

In der Service-Rolle (`roles/<service>/tasks/main.yml`):

```yaml
- ansible.builtin.import_role:
    name: compose_hull
  vars:
    service_defaults:
      # ... normale Service-Konfiguration ...
      dedicated_user:
        enabled: true
        name: <service-name>        # z.B. "n8n", "loki", "grafana"
        uid: <unique-uid>            # z.B. 13091 (5-stellig, zufällig)
        add_to_docker_group: false   # Optional, Standard: false
```

### Parameter

#### `dedicated_user.enabled` (Boolean, Standard: `false`)
Aktiviert das Dedicated User Pattern für diesen Service.

#### `dedicated_user.name` (String, Pflicht wenn enabled)
Name des System-Users, der erstellt wird. Sollte identisch mit dem Service-Namen sein.

#### `dedicated_user.uid` (Integer, Pflicht wenn enabled)
Eindeutige User-ID (UID) für den System-User.

**Empfohlene UIDs:**
- Bereich: 10000-65535 (um Konflikte mit normalen Usern zu vermeiden)
- Verwende 5-stellige zufällige UIDs für mehr Sicherheit
- Dokumentiere verwendete UIDs zur Vermeidung von Duplikaten

**Bereits verwendete UIDs:**
- 10001: loki
- 472: grafana
- 13091: n8n

#### `dedicated_user.add_to_docker_group` (Boolean, Standard: `false`)
Fügt den dedizierten User zur `docker`-Gruppe hinzu.

**Wichtig:** Nur aktivieren, wenn der Container Zugriff auf den Docker Socket benötigt!

- `false` (Standard): Container läuft mit `uid:uid` (z.B. `13091:13091`)
- `true`: Container läuft mit `uid:docker-gid` (z.B. `13091:999`)

## Docker Compose Integration

Im Service-Template (`roles/<service>/templates/docker-compose.yml.j2`):

```yaml
services:
  myservice:
    container_name: "{{ service_cfg.name }}"
    image: myimage:latest
    restart: unless-stopped
{% if service_cfg.dedicated_user.enabled | default(false) %}
    user: "{{ service_cfg.dedicated_user.uid }}:{% if service_cfg.dedicated_user.add_to_docker_group | default(false) %}{{ PGID }}{% else %}{{ service_cfg.dedicated_user.uid }}{% endif %}"
{% endif %}
    security_opt: *base_security_opt
    volumes:
      - "{{ service_cfg.directory }}/data:/app/data"
```

Die `user:`-Direktive wird automatisch gesetzt wenn `dedicated_user.enabled = true`.

## Ablauf (Internal)

Wenn ein Service mit `dedicated_user.enabled: true` deployed wird:

1. **user_info.yml**: Holt PUID/PGID vom Admin-User und der Docker-Gruppe
2. **facts.yml**:
   - Merged alle Service-Konfigurationen
   - Setzt `service_cfg.owner` automatisch auf `dedicated_user.name`
3. **create_dedicated_user.yml**:
   - Erstellt System-User mit der konfigurierten UID
   - Fügt User optional zur docker-Gruppe hinzu
4. **create_directories.yml**:
   - Erstellt Service-Verzeichnisse mit dem dedizierten User als Owner
5. **deploy_docker.yml**:
   - Deployed Container mit der `user:`-Direktive

## Beispiele

### Beispiel 1: n8n (ohne Docker-Socket-Zugriff)

```yaml
# roles/n8n/tasks/main.yml
- ansible.builtin.import_role:
    name: compose_hull
  vars:
    service_defaults:
      directory: "{{ docker_dir }}/n8n"
      name: n8n
      subdirs: ["n8n_data", "local-files"]
      dedicated_user:
        enabled: true
        name: n8n
        uid: 13091
        add_to_docker_group: false  # Kein Docker-Socket-Zugriff nötig
```

**Resultat:**
- Host-System: User `n8n` (UID 13091), Gruppe `n8n` (GID 13091)
- Verzeichnisse: `/docker/n8n/*` gehören `n8n:docker` (13091:999)
- Container: Läuft als `13091:13091` (user:group)

### Beispiel 2: Loki (mit Docker-Gruppe für Volumes)

```yaml
# roles/loki/tasks/main.yml
- ansible.builtin.import_role:
    name: compose_hull
  vars:
    service_defaults:
      directory: "{{ docker_dir }}/loki"
      name: loki
      subdirs: ["data", "conf"]
      dedicated_user:
        enabled: true
        name: loki
        uid: 10001
        add_to_docker_group: false
```

**Resultat:**
- Host-System: User `loki` (UID 10001)
- Container: Läuft als `10001:10001`

## Migration bestehender Services

### Schritt 1: Backup
```bash
ssh -p 2612 daniel@89.58.48.243
sudo tar czf /tmp/n8n-backup-$(date +%Y%m%d).tar.gz /docker/n8n
```

### Schritt 2: Service-Konfiguration aktualisieren
Füge `dedicated_user` zur Service-Konfiguration hinzu (siehe Beispiele oben).

### Schritt 3: Docker Compose Template aktualisieren
Füge die `user:`-Direktive zum Template hinzu (siehe "Docker Compose Integration").

### Schritt 4: Deploy
```bash
pipenv run ansible-playbook n8n.yml -i hosts
```

### Schritt 5: Verifizierung
```bash
ssh -p 2612 daniel@89.58.48.243
docker inspect n8n | grep -A 2 '"User"'
# Sollte zeigen: "User": "13091:13091"

ls -la /docker/n8n
# Sollte zeigen: drwxrwxr-x n8n docker
```

## Best Practices

1. **Eindeutige UIDs**: Verwende 5-stellige zufällige UIDs (10000-65535)
2. **Dokumentation**: Notiere verwendete UIDs im Projekt
3. **Minimale Berechtigungen**: Setze `add_to_docker_group: false` wenn möglich
4. **Testen**: Teste Service-Funktionalität nach Migration gründlich
5. **Monitoring**: Prüfe Container-Logs nach Deployment auf Permission-Fehler

## Verwandte Dokumentation

- [CONTAINER_USER_SECURITY_ANALYSIS.md](../../docs/CONTAINER_USER_SECURITY_ANALYSIS.md) - Security-Analyse
