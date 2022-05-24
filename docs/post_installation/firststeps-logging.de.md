Das Logging in mailcow: dockerized besteht aus mehreren Stufen, ist aber immerhin wesentlich flexibler und einfacher in einen Logging-Daemon zu integrieren als bisher.

In Docker schreibt die containerisierte Anwendung (PID 1) ihre Ausgabe auf stdout. Für echte Ein-Anwendungs-Container funktioniert das sehr gut.
Führen Sie `docker-compose logs --help` aus, um mehr zu erfahren. 

Einige Container protokollieren oder streamen an mehrere Ziele.

Kein Container wird persistente Logs in sich behalten. Container sind flüchtige Objekte!

Am Ende wird jede Zeile der Logs den Docker-Daemon erreichen - ungefiltert.

Der **Standard-Logging-Treiber ist "json "**.

### Gefilterte Logs

Einige Logs werden gefiltert und in Redis-Schlüssel geschrieben, aber auch in einen Redis-Kanal gestreamt.

Der Redis-Kanal wird verwendet, um Protokolle mit fehlgeschlagenen Authentifizierungsversuchen zu streamen, die von netfilter-mailcow gelesen werden.

Die Redis-Schlüssel sind persistent und halten 10000 Zeilen von Logs für die Web-UI.

Dieser Mechanismus macht es möglich, jeden beliebigen Docker-Logging-Treiber zu verwenden, ohne die 
ohne die Fähigkeit zu verlieren, Logs von der UI zu lesen oder verdächtige Clients mit netfilter-mailcow zu sperren.

Redis-Schlüssel enthalten nur Logs von Anwendungen und filtern Systemmeldungen heraus (man denke an Cron etc.).

### Logging-Treiber

#### Über docker-compose.override.yml

Hier ist die gute Nachricht: Da Docker einige großartige Logging-Treiber hat, können Sie mailcow: dockerized mit Leichtigkeit in Ihre bestehende Logging-Umgebung integrieren.

Erstellen Sie eine `docker-compose.override.yml` und fügen Sie zum Beispiel diesen Block hinzu, um das "gelf" Logging-Plugin für `postfix-mailcow` zu verwenden:

```
version: '2.1'
services:
  postfix-mailcow: # oder ein anderer
    logging:
      driver: "gelf"
      options:
        gelf-address: "udp://graylog:12201"
```

Ein weiteres Beispiel für **Syslog**:

```
version: '2.1'
services:

  postfix-mailcow: # oder ein anderer
    logging:
      driver: "syslog"
      options:
        syslog-address: "udp://127.0.0.1:514"
        syslog-facility: "local3"

  dovecot-mailcow: # oder ein anderer
    logging:
      driver: "syslog"
      options:
        syslog-address: "udp://127.0.0.1:514"
        syslog-facility: "local3"

  rspamd-mailcow: # oder ein anderer
    logging:
      driver: "syslog"
      options:
        syslog-address: "udp://127.0.0.1:514"
        syslog-facility: "local3"

# Nur für Rsyslog:
# Um die local3-Eingabe nach /var/log/mailcow.log zu verschieben und die Verarbeitung zu beenden, erstellen Sie eine Datei "/etc/rsyslog.d/docker.conf":

local3.* /var/log/mailcow.log
& stop

# Danach rsyslog neu starten.

```

#### Über daemon.json (global)

Wenn Sie den Logging-Treiber **global** ändern wollen, editieren Sie die Konfigurationsdatei des Docker-Daemons `/etc/docker/daemon.json` und starten Sie den Docker-Dienst neu:

```
{
...
  "log-driver": "gelf",
  "log-opts": {
    "gelf-address": "udp://graylog:12201"
  }
...
}
```

Für Syslog:

```
{
...
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "udp://1.2.3.4:514"
  }
...
}
```

Starten Sie den Docker-Daemon neu und führen Sie `docker-compose down && docker-compose up -d` aus, um die Container mit dem neuen Protokollierungstreiber neu zu erstellen.
