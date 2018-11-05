# Installation von GitLab 8.12 #

1. [Abhängigkeiten](#abhängigkeiten)
2. [System User](#system-user)
3. [GitLab Shell](#gitlab-shell)
4. [GitLab Workhorse](#gitlab-workhorse)
5. [GitLab](#gitlab)
6. [Install Bundle Gems](#install-bundle-gems)
7. [Init Database](#init-database)
8. [Precompile assets](#precompile-assets)
9. [Apache Redirect](#apache-redirect)
10. [Check Status](#check-status)
11. [Fertig](#fertig)
12. [GitLab-Shell und die SSH-keys](#gitlab-shell-und-die-ssh-keys)

13. [Upgraden](#upgraden)
14. [Upgraden von 7.x auf 8.x](#upgraden-von-7x-auf-8x)
15. [Upgraden von 8.x auf 8.17](#upgraden-von-8x-auf-817)

16. [Impressum](#impressum)


Diese Anleitung bezieht sich direkt auf die offiziellen Installationsanleitung [hier](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/install/installation.md). Für Uberspace sind jedoch einige Dinge unwichtig, andere zusätzlich nötig. Genauere Beschreibungen sind in der offiziellen Anleitung zu finden. Viele der Befehle aus der offiziellen Anleitung laufen jedoch auch ohne das sudo.


## Abhängigkeiten ##

### Python ###

Python wird in einer Version 2.5+ (nicht 3.0+) benötigt.

Python ist auf den Uberspace-Servern bereits aktiviert. Jedoch manchmal noch in der Version 2.4. Prüft das mit dem Befehl `python -V`. Falls noch die alte Version aktiv ist, könnt ihr nach der Anleitung [hier](https://wiki.uberspace.de/development:python) eine neuere aktivieren.


### Git ###

Git wird in der Version 1.7.10+ benötigt. *Nicht zu verwechseln mit 1.7.1!*

Git ist auch bereits auf den Servern installiert. Prüft mit `git --version` eure Version. Falls sie zu alt ist könnt ihr über Toast eine neuere Version installieren. Sucht dazu [hier](https://www.kernel.org/pub/software/scm/git/) eine Version und kopiert den Link zum Tarball. Mit `toast arm [URL zum Tarball]` wird diese installiert und eingerichtet.

```bash
#Git 2.10.0:
toast arm https://www.kernel.org/pub/software/scm/git/git-2.10.0.tar.gz
# Dies kann einige Minuten dauern...
```


### Redis ###

Installiere Redis wie [hier](https://wiki.uberspace.de/database:redis) beschrieben. Redis akzeptiert auf Uberspace nur Verbindungen zu seinem Socket, was in allen Konfigurationsfiles von GitLab zu beachten ist.


### cmake ###

Ab Version 7.2 benötigt Gitlab zudem cmake. Dies ist aber auf Uberspace nicht standardmäßig vorinstalliert!

Abhilfe schaffen wir uns wieder mittels toast und passen die Paths fuer curl, gcc und openssl an:

```bash
 LD_LIBRARY_PATH=/package/host/localhost/gcc-5.2/lib64:/package/host/localhost/curl-7.44.0/lib:/package/host/localhost/openssl-1.0.2h/lib toast arm https://cmake.org/files/v3.0/cmake-3.0.2.tar.gz
```

### Ruby ###

Ruby wird in der Version 2.0+ benötigt.

Auf den Uberspace Servern wird standardmäßig eine ältere Version genutzt. [Hier](https://wiki.uberspace.de/development:ruby) wird erklärt wie die neueren zur Verfügung stehenden Versionen aktiviert werden.

```bash
cat \<\<'__EOF__' >> ~/.bashrc
export PATH=/package/host/localhost/ruby-2.2.3/bin:$PATH
export PATH=$HOME/.gem/ruby/2.2.0/bin:$PATH
__EOF__
```

#### .bashrc vs. .bash_profile ####
SSH Keys werden innerhalb GitLab über die GitLab Shell verwaltet. Da diese SSH Keys direkt auf das GL Shell Script verweisen wird `.bash_profile` nicht geladen.
Seid ihr der Anleitung auf [Uberspace](https://wiki.uberspace.de/development:ruby) gefolgt, müssen daher die `$PATH` Angaben aus der `.bash_profile` in `.bashrc` verschoben (oder kopiert) werden.


#### Bundler Gem ####

```bash
gem install bundler --user-install --no-ri --no-rdoc
```

`--user-install` sorgt dafür, dass der Gem im Nutzerverzeichnis statt global installiert wird.

**.gemrc**

Alternativ lässt sich diese Option auch dauerhaft aktivieren. Dafür einfach `gem: --user-install --no-rdoc --no-ri` in die ~/.gemrc eintragen. Falls die Datei noch nicht existiert erstellen.

```bash
touch ~/.gemrc
echo "gem: --user-install --no-rdoc --no-ri" > ~/.gemrc
```


## System User ##

Auf den Uberspace Servern gibt es *nicht* die Möglichkeit einen extra User `git` anzulegen. Der Normale Nutzer geht allerdings auch. Jedoch muss das in **fast allen** Konfigurationsfiles beachtet werden.


## GitLab Shell ##

Unten die Shell-Befehle nach Anleitung.

```bash
cd ~
git clone https://gitlab.com/gitlab-org/gitlab-shell.git -b v2.6.8
cd gitlab-shell
cp config.yml.example config.yml
nano config.yml
```
Wichtig in der `config.yml`:

Änderung aller Pfade von `/home/git/...` zu `/home/[Nutzername]/...`

Außerdem sind folgende Änderungen durchzuführen:

```ruby
user: [Nutzername]
gitlab_url: "https://[Nutzername].[Host].uberspace.de"

#[...]Redis Einstellungen

bin: /usr/local/bin/redis-cli
# Der Pfad sollte identisch mit der Ausgabe von 'which redis-cli' sein

# auskommentieren:
# host: ...
# port: ...

socket: /home/[Nutzername]/.redis/sock
# der Pfad findet sich auch in der Datei '~/.redis/conf' um sicherzugehen.
```

**Für die gitlab_url mit https muss der komplette Pfad inklusive Server und .uberspace.de angegeben werden, damit das Zertifikat auch passt. Auch wenn ihr eine eigene Domain haben solltet!**

Nachdem die Konfigurationdatei geändert wurde.

```bash
./bin/install
```

## Gitlab-Workhorse ##

Neu hinzugekommen ist seit Gitlab 8.0 der so genannte ~~*git-http-server*~~. Dieser wird seit Gitlab 8.2 *gitlab-workhorse* genannt. Ein kleiner Prozess, der beim Pushen und Pullen von Git-Repositories als HTTP-Server einspringt, und so den Unicorn-Server entlastet (damit das Frontend weiterhin flüssig läuft). Workhorse benötigt **GO*lang***, um zu kompilieren.

> Leider ist GO unter CentOS 5 nicht lauffähig! Siehe dazu: [Go auf deinem Uberspace](https://wiki.uberspace.de/development:golang).


```bash
cd ~
git clone https://gitlab.com/gitlab-org/gitlab-workhorse.git
cd gitlab-workhorse
```
Im `Makefile` muss die Zeile `GOBUILD := go build -ldflags "-X main.Version=$(VERSION)"`
mit `GOBUILD := go build -ldflags "-X main.Version=$(VERSION)" -p 1` ersetzt werden. Das `-p 1` sorgt dafuer, dass wir nicht zu viele Dateien waehrend des Kompilierens oeffnen, indem wir die Anzahl der verwendeten Prozessoren auf eins reduzieren.


```bash
make
```

## GitLab ##

```bash
cd ~
git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 8-3-stable gitlab
cd gitlab

# init some configs
cp config/gitlab.yml.example config/gitlab.yml
cp config/unicorn.rb.example config/unicorn.rb
cp config/resque.yml.example config/resque.yml
cp config/database.yml.mysql config/database.yml

cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb # No need to edit this later

# create some directories and make sure the chmod is correct
mkdir $HOME/gitlab-satellites
mkdir $HOME/repositories
mkdir tmp/pids/
mkdir tmp/sockets/
mkdir public/uploads

chmod g+ws $HOME/repositories && chmod o-rwx $HOME/repositories
chmod o-rwx config/database.yml

# Muss nicht aber ist nützlich
git config --global user.name "GitLab"
git config --global user.email "gitlab@localhost"
git config --global core.autocrlf input
```


### gitlab.yml Konfiguration ###

`nano config/gitlab.yml`

```ruby
host: [Nutzername].[Host].uberspace.de
https: true
[...]
user: [Nutzername] # Auskommentierung muss entfernt werden ("#" am Anfang der Zeile entfernen)!
```

**Interessant** für alle, die Gitlab in einer Subdomain (zB. für die [SSH-Keys](#eindeutige-logins-durch-subdomains)) oder in einem Unterordner verwenden wollen:
Mit dem Eintrag `ssh_host` lässt sich für SSH ein anderer Host angeben, als für http (`host`).

Unter 3: Advanced Settings:

alle `/home/git/...` ändern in `/home/[Nutzername]/...`

```ruby
git:
bin_path: /home/[Nutzername]/.toast/armed/bin/git
```

**Der `bin_path` sollte stimmen, falls git per Toast installiert wurde. Trotzdem sicherheitshalber per `which git` den Pfad auf seine Richtigkeit überprüfen!**


### unicorn.rb Konfiguration

`nano config/unicorn.rb`

alle `/home/git/...` ändern in `/home/[Nutzername]/...`
`listen "127.0.0.1:8080"...` *port* in einen noch freien Port ändern. z.B. für den Port 9765 in: `listen "127.0.0.1:9765"...`

Um zu überprüfen ob der gewählte Port noch frei ist, führt folgenden Befehl durch. Ist die Rückgabe leer, so ist der Port noch nicht belegt.

```bash
netstat -tulpen | grep [port]
```

**Diesen Port am besten merken oder irgendwo notieren. Wir brauche ihn später nochmal! (unter dem Namen `[your unicorn port]`)**

> **Um Verwirrungen vorzubeugen:**

> Laut [Uberspace-Wiki](https://wiki.uberspace.de/system:ports) sind Ports nur im Bereich von 61000 bis 65535 erlaubt. Dies bezieht sich aber nur auf Ports, die wir später nach Außen auf dem Server öffnen wollen!
> Wir hingegen wollen den Port aber nur intern nutzen, um [später](#apache-redirect) den Webserver per .htaccess vom externen Port 80 auf unseren lokalen Port weiterzuleiten.
> Es empfiehlt sich also vermutlich ein Port irgendwo zwischen 1024 und 61000 zu nehmen. Eventuell aufpassen, dass man nicht gerade einen von den [well-known Ports](https://de.wikipedia.org/wiki/Liste_der_standardisierten_Ports) erwischt.


### resque.yml Konfiguration

Den richtigen Redis Zugang einfügen
`production: 'unix:/home/[Nutzername]/.redis/sock'`
Socket ändern, falls er bei der GitLab Shell schon anders war.


### database.yml Konfiguration

Unter `production: ` die MySQL Nutzerdaten eintragen:

```ruby
database: [Nutzername]_[Datenbankname] # Datenbankname nach uberspace-Namenskonvention
username: [Nutzername]
password: [MySQL Passwort] #Wenn es nicht geändert wurde, dann unter ~/.my.cnf zu finden
```


### statische Files ausliefern ###

`nano config/environments/production.rb`

```ruby
config.serve_static_files = true
```


## Install Bundle Gems ##

**Achtung:** [Gabriel Bretschner][1] Hat darauf hingewiesen, dass es auf Servern unter CentOS 5 zu Problemen mit *Charlock Holmes* kommen kann. Die Lösung ist recht einfach und stammt aus dem [Uberspace-Wiki](https://wiki.uberspace.de/development:ruby#charlock_holmes):

```bash
bundle config build.charlock_holmes --with-icu-dir=/package/host/localhost/icu4c
```

**Install:**

```bash
bundle install --deployment --without development test postgres aws
```


## Init Database ##

```bash
bundle exec rake gitlab:setup RAILS_ENV=production
```

## Precompile assets ##

```bash
bundle exec rake gitlab:assets:clean gitlab:assets:compile cache:clear RAILS_ENV=production
```

Dieser Vorgang kann eine Weile dauern...

### Tippt 'yes' zum erstellen der Datenbank ###

**Wenn ihr fertig seid, sollte so etwas kommen:**

```
Administrator account created:

login.........root
password......5iveL!fe
```

**Den Benutzernamen und das Passwort brauchen wir später für den erstmaligen Login noch!**


### GitLab als Uberspace-Service verwalten ###

Gabriel Bretschner hat die ultimative Lösung für Uberspace parat!
In seinem [Blogeintrag][1] erklärt er, wie sich GitLab als Service verwalten lässt.

Eine kurze Anleitung und die Service-Skripte findet ihr in seiner eigenen [GitLab-Installation](https://git.kanedo.net/kanedo/gitlab-uberspace/tree/master/services)

... und als Kopie auch nochmal bei mir:
- [Anleitung](services/Readme.md)
- [sidekiq-Service](services/sidekiq) *man beachte den neuen Parameter `-q archive_repo` um Downloads von Repositories als gepacktes Archiv zu ermöglichen*
- [gitlab-Service](services/gitlab) (Unicorn)

**Und für das neue gitlab-workhorse:**
- [gitlab-workhorse-Service](services/gitlab-workhorse)

In dem Script sind am Anfang zwei Ports anzugeben.
1. Für `[your unicorn port]` nehmen wir den unter [unicorn.rb Konfiguration](#unicornrb-konfiguration) ausgewählten Port für den Unicorn-Webserver.
2. Für `[your gitlab-workhorse port]` suchen wir uns einen neuen freien Port nach dem selben Schema aus (zwischen 1024 und 61000) und merken uns diesen nun auch noch.

## Apache Redirect ##

In `~/html` oder einem Subdomain-Ordner eine `.htaccess` erstellen und damit füllen

```htaccess
<IfModule mod_rewrite.c>
    RewriteEngine On

    RewriteCond %{HTTPS} !=on

    # redirect all trafic to https
    RewriteCond %{ENV:HTTPS} !=on
    RewriteRule .* https://%{SERVER_NAME}%{REQUEST_URI} [R=301,L]

    # don't escape encoded characters in api requests
    RewriteCond %{REQUEST_URI} ^/api/v3/.*
    RewriteRule .* http://127.0.0.1:[your gitlab-workhorse port]%{REQUEST_URI} [P,QSA,NE]

    # redirect file download requests to gitlab-workhorse
    RewriteCond %{REQUEST_URI} .*\.(git|zip) [OR]
    RewriteCond %{REQUEST_URI} .*/raw/
    RewriteRule .* http://127.0.0.1:[your gitlab-workhorse port]%{REQUEST_URI} [P,QSA]

    # redirect any other traffic to unicorn
    RewriteBase /
    RewriteCond %{DOCUMENT_ROOT}/%{REQUEST_FILENAME} !-f
    RewriteRule .* http://127.0.0.1:[your unicorn port]%{REQUEST_URI} [P,QSA]
</IfModule>

RequestHeader set X-Forwarded-Proto https
RequestHeader set X-Forwarded-Ssl on
```

> siehe Beispiel-.htaccess: [.htaccess](_.htaccess)

Hier dürfen nun die beiden gemerkten Ports eingesetzt und anschließend endlich vergessen werden!

> Die Zeile mit `RequestHeader` behebt den Fehler "Can't verify CSRF token authenticity" beim Login mit https.

## Check Status ##

```bash
bundle exec rake gitlab:env:info RAILS_ENV=production
bundle exec rake gitlab:check RAILS_ENV=production
```

## Fertig ##

Jetzt sollte erst mal alles funktionieren.


## GitLab-Shell und die SSH-keys ##

Ein großes Problem bei GitLab und Uberspace ist das fehlen eines separaten Users. Loggt ihr euch für gewöhnlich per Key über SSH ein, wird dies nach der Installation der GitLab-Shell nicht mehr möglich sein. Diese blockt nämlich den Shell-Zugriff für alle auf GitLab registrierten Keys! Trotzdem wollen wir aber gerne die GitLab-Pfade und Nutzerrechte zum clonen, pushen etc. benutzen.
Im Grunde genommen gibt es dafür zwei Mögliche Lösungen. Beide haben den Vorteil, dass durch die Nutzung der ssh-config die angelegten Host-Aliases Systemweit zu Verfügung stehen (inklusive SFTP)!

> Zur Nutzung unter Windows kann ich leider keine klare Aussage treffen. Wenn hier jemand Erfahrung hat würde ich mich sehr über Hinweise freuen.


### Separates Keypaar ###

Ihr legt euch ein separates Key-Paar für den Shellzugriff an.

```bash
ssh-keygen -f ~/.ssh/shellAccess
# optional auch mit custom-Mail/Kommentar:
ssh-keygen -f ~/.ssh/shellAccess -C [aussagekräftiger-Name]@[server]
```

und kopiert den Inhalt des Public-Keys (`.pub`) in die `~/.ssh/authorized_keys` eures Servers.

Anschließend müsst ihr allerdings beim Login noch deutlich machen, mit welchem Keypaar ihr euch einloggen wollt. Das geht am Besten, indem ihr euch in eure `~/.ssh/config` einen Host-*Alias* anlegt, der dann in etwa wie folgt aussehen sollte:

```
Host Servername.ShellKey
HostName [Host]
User [Nutzername]
IdentityFile ~/.ssh/shellAccess
IdentitiesOnly yes
```

Nun sollten wir uns direkt mit `ssh Servername.ShellKey` einloggen können!

Der `Host` Eintrag ist dabei als Alias zu verstehen. Ihr könnt im Prinzip benutzen, was euch gefällt.

Alternativ lässt sich ssh auch zur einmaligen Nutzung ohne ssh-config überreden. Das sieht dann in etwa wie folgt aus:

```bash
ssh -i ~/.ssh/shellAccess [Nutzername]@[Host]
```


### per Passwort ###

Alternative Zwei funktioniert so ähnlich, verzichtet aber auf ein weiteres Keypaar. Stattdessen loggen wir uns old-school mäßig via Passwort ein.
Hierfür muss allerdings ssh konkret der Login mit einem Key verboten werden.

Das geht einmalig mit einem Konstrukt wie:

```bash
ssh -o PreferredAuthentications=keyboard-interactive -o PubkeyAuthentication=no [Nutzername]@[Host]
```

Zum Dauerhaften deaktivieren erstellen wir uns wieder einen Eintrag in die ssh-config `~/.ssh/config`.

```
Host Servername.NoKey
HostName [Nutzername].[Host].uberspace.de
User [Nutzername]
PubkeyAuthentication no
```

Der Befehl zum Verbinden lautet nun `ssh Servername.NoKey`.

### Eindeutige Logins durch Subdomains ###

Gabriel Bretschner hat vor kurzem eine super Ergänzung [veröffentlicht][1].
Er erklärt darin wie sich das Problem mit den SSH-Keys durch eine separate Subdomain für GitLab lösen lässt.

Im Grunde genommen ganz einfach. Ihr lasst GitLab und die GitLab-Shell in einer Subdomain laufen (zB.: git.[Nutzername].[Host].uberspace.de) und erstellt euch einen Host-*Alias* ähnlich wie im obigem [Beispiel](#separates-keypaar).
Natürlich müssen auch die Pfade in den *Configs* angepasst werden.

Besonders wichtig sind hier `gitlab_url` in der *gitlab-shell/config.yml*, sowie `host` in der *gitlab/config/gitlab.yml*.
Damit anschließend die Pfade für SSH noch stimmen sollte auch der Eintrag `ssh_host` in der *gitlab.yml* ergänzt werden ([siehe](#gitlab-yml-konfiguration)).

Dann erstellt ihr euch ein neues Keypaar und den passenden Eintrag in die ssh-config.

```bash
ssh-keygen -f ~/.ssh/shellAccess
```

```
Host git.[Nutzername].[Host].uberspace.de
User [Nutzername]
IdentityFile ~/.ssh/shellAccess
IdentitiesOnly yes
```

> ~~**Einziges Problem** an dieser Lösung sind die SSL-Zertifikate. Uberspace biete selber zwar Wildcard-Zertifikate an, diese sind aber natürlich nicht für eigene Domains oder Sub-Subdomains der User gültig. Im Allgemeinen lässt Uberspace zwar [eigene Zertifikate](https://wiki.uberspace.de/webserver:https#nutzung_eigener_ssl-zertifikate) zu. Anbieter wie [StartCom](https://www.startssl.com/) bieten sogar einfache *Class 1* Zertifikate gratis an! Subdomains decken diese jedoch nicht ab (**Ausnahme**: StartSSL Class 1 beinhaltet eine Subdomain!) . Entsprechende *Class 2* Zertifikate kosten bei allen Stellen etwas.~~

> Dank Let's Encrypt und dem hervorragend Support bei [Uberspace](https://wiki.uberspace.de/webserver:https#let_s-encrypt-zertifikate) sollte es für niemanden mehr ein Hinderniss sein, die eigene Seite mit einem entsprechenden Zertifikat abzusichern. Auch Subdomains sind kein Problem. Wer noch ein Zertifikat bei [StartCom](https://www.startssl.com/) hat, sollte demnächst wechseln, da deren root-Zertifikat in nächster Zeit von Firefox und Chrome nicht mehr getrusted wird: [Googles Certificate-Transparency-Projekt](https://docs.google.com/document/d/1C6BlmbeQfn4a9zydVi2UvjBGv6szuSB4sMYUcVrR8vQ/edit) && [Golem](http://www.golem.de/news/zertifikate-mozilla-will-startcom-und-wosign-das-vertrauen-entziehen-1609-123462.html).

### ControlMaster ###

Falls Ihr für SSH einen [ControlMaster](https://wiki.uberspace.de/faq?s[]=controlmaster#ich_baue_viele_ssh-verbindungen_auf_und_komm_ploetzlich_nicht_mehr_rein) verwendet, solltet ihr diesen für die entsprechenden Einträge deaktivieren, um eine Verwirrung des Shell-Logins und des GitLab-Logins zu vermeiden!

Dazu einfach `ControlMaster no` noch zum Host in die ssh-config hinzufügen. Fertig!

### Shared-Keys ###

Eine weitere Lösung kann durch die manuelle Bearbeitung der Gitlab-Shell Einträge liefern. Weiter Details können [hier als Gist](https://gist.github.com/hanseartic/368a63933afb7c9f7e6b) nachgelesen werden.

## Upgraden ##

### Gitlab-Shell ###

Manche Gitlab-Upgrades benötigen auch eine aktuellere Version von Gitlab-Shell. Keine Panik, das ist ganz einfach - z.B.: auf 2.6.8:

```bash
cd ~/gitlab-shell
git fetch
git checkout v2.6.8
```

### GitLab ###

Zuerst sicherheitshalber ein Backup erstellen. Anschließend einfach den Prozess stoppen, den aktuellen Stand pullen und neu builden lassen.

```bash
cd ~/gitlab
bundle exec rake gitlab:backup:create RAILS_ENV=production # kann eine Weile dauern
svc -d ~/service/gitlab && svc -d ~/service/sidekiq && svc -d ~/service/gitlab-workhorse

git fetch --all
git checkout 8-3-stable

bundle install --without development test postgres aws --deployment
bundle exec rake db:migrate RAILS_ENV=production
bundle exec rake assets:clean assets:precompile cache:clear RAILS_ENV=production
```

> Fall es beim `git fetch` zu Konflikten kommt, reicht in der Regel ein einfaches `git checkout filename` für die entsprechenden Dateien. Danach kann `git fetch` erneut ausgeführt, und der Update-Prozess fortgeführt werden.

Änderungen in der production.rb müssen gegebenenfalls erneut gesetzt werden.

`nano config/environments/production.rb`

```ruby
config.serve_static_files = true
```

Falls alles erfolgreich verlief kann GitLab nun wieder gestartet werden.

```bash
svc -u ~/service/sidekiq && svc -u ~/service/gitlab-workhorse && svc -u ~/service/gitlab
```

Nach dem Start schadet ein erneuter Check nicht:

```bash
bundle exec rake gitlab:env:info RAILS_ENV=production
bundle exec rake gitlab:check RAILS_ENV=production
```


## Upgraden von 7.x auf 8.x ##

Bei einem Upgrade auf Gitlab 8.x ist der neu hinzugekommene git-http-server zu beachten.
Dieser kann wie in [Gitlab-Git-HTTP-Server](#gitlab-git-http-server) erwähnt installiert werden.

Eine Konfiguration des Tools selber ist nicht notwendig. Dafür müssen entsprechende Ports im neuen Service-Script angeben werden (siehe: [GitLab als Uberspace-Service verwalten](#gitlab-als-uberspace-service-verwalten)).

Um die Zuordnung zum jeweiligen WebServer unserem Apache mitzuteilen, müssen folgende zwei Zeilen in der *.htaccess* vor dem allgemeinen Routing zu Unicorn ergänzt werden (siehe: [.htaccess](_.htaccess)):

```htaccess
RewriteCond %{REQUEST_URI} .*\.(git)
RewriteRule .* http://127.0.0.1:[your gitlab-workhorse port]%{REQUEST_URI} [P,QSA]
```

## Upgraden von 8.x auf 8.17 ##

Seit Version 8.17 verwendet Gitlab die node.js Bibliotek [webpack](https://webpack.js.org) um seine frontend assets zu kompilieren. Webpack setzt eine node Version von `> v4.3.0` voraus.

Auf dem Uberspace lässt sich node wie in der Anleitung [hier](https://wiki.uberspace.de/development:nodejs) beschrieben installieren.


## Impressum ##

Nach einem Tutorial von: [Benjamin Milde](http://kobralab.centaurus.uberspace.de/benni/uberspace/blob/master/install.md)

Aktualisierungen, Fehlerbehebungen und Ergänzungen: [Richard Weinhold](https://ricwein.com/u/ricwein), [Andrej Schoeke](https://github.com/schoeke)(WIP)

Support und Feedback von: [Gabriel Bretschner](http://kanedo.net), [Christian Raunitschka](http://ch.rauni.me), [Jan Beck](http://jancbeck.com)

[1]: https://blog.kanedo.net/1925,gitlab-7-0-auf-einem-uberspace-installieren.html
