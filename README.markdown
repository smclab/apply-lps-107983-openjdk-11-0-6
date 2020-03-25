# Risolvere il problema di compatibilità OpenJDK 11.0.6 su Liferay CE 7.2 GA2

Qualche giorno addietro ho avuto la felice idea di aggiornare la versione di
OpenJDK 11.0.5 alla 11.0.6. Sembrava che tutto filasse per il verso giusto ma
sono stato colpito dal proverbio *"Non lasciare la vecchia strada per quella*
*nuova"* 😔

All’avvio di Liferay Portal 7.2 GA2 CE eccoti apparire l'errore molto criptico
indicato a seguire.

```
org.apache.jasper.JasperException: PWC6033: Error in Javac compilation for JSP
```

Questo errore è presente anche su Liferay DXP 7.2 Fix Pack 4 (la versione che ho provato personalmente). 

Dopo qualche ricerca ho scoperto che l'errore è dovuto ad un bug (aperto) sulla 
[OpenJDK 11.0.6 JDK-8237875 (zipfs) DirectoryStream entries changed behavior 
between JDK 11.0.5 and JDK 11.0.6](https://bugs.openjdk.java.net/browse/JDK-8237875)

Liferay di suo ha provveduto a risolvere tramite [LPS-107983 - 
OpenJDK 11.0.6 compatibility problem](https://issues.liferay.com/browse/LPS-107983). 
Questa LPS risulta chiusa e risolve il problema su 7.x e master.

L'ultima GA di Liferay CE non contiene ancora la fix, allora, come possiamo risolvere? Le possibili soluzioni sono:
   1. Downgrade della versione di OpenJDK a 11.0.5;
   2. Se disponiamo della DXP possiamo richiedere tramite il Customer Portal la
      Hot Fix specifica, ricordandoci di indicare nella descrizione del ticket
      l’LPS che in questo caso è LPS-107983;
   3. Se disponiamo della CE possiamo applicare la patch in modo autonomo.

La prima strada non è sempre percorribile a causa del fatto che il più delle
volte l’aggiornamento della JDK rientra tra uno di quei componenti il cui
aggiornamento è centralizzato e abbiamo per cui poco spazio di manovra (come per esempio un downgrade).

La seconda strada è semplice da percorre; molti di noi hanno già avuto questo
tipo di esperienza. Solitamente in pochi giorni si ottiene la Hot Fix da Liferay.

Per non farci mancare nulla, direi di scegliere la terza strada, che poi è
quella più interessante.


## 1 - Come applicare la patch a Liferay Portal CE
Vediamo come sia possibile e semplice applicare la patch a Liferay Portal CE, 
in particolare la nostra versione di riferimento è la 7.2.1 GA2.

Come possiamo applicare la nostra Hot Fix anche sulla CE? La risposta è:
[Overriding lpkg files](https://portal.liferay.dev/docs/7-1/tutorials/-/knowledge_base/t/overriding-lpkg-files)

Il bundle a cui dobbiamo applicare la [patch 84189](https://patch-diff.githubusercontent.com/raw/brianchandotcom/liferay-portal/pull/84189.patch) è **Liferay Portal OSGi Web Servlet JSP Compiler** 
([com.liferay.portal.osgi.web.servlet.jsp.compiler](https://repo1.maven.org/maven2/com/liferay/com.liferay.portal.osgi.web.servlet.jsp.compiler/4.0.14/)) che fa
parte dell’LPKG **Liferay CE Static - Impl.lpkg**. In questo caso possiamo quindi
applicare l’Overriding lpkg files, anche a quelli Static.

La procedura è quella indicata:
   1. Creare il bundle in formato jar contenente la patch;
   2. Il nome del bundle deve essere uguale all'originale, meno le informazioni
      sulla versione (`com.liferay.portal.osgi.web.servlet.jsp.compiler.jar`);
   3. Copiare il file .jar del bundle nella cartella `$LIFERAY_HOME/osgi/static`;
   4. Rimuovere il contenuto della directory `$LIFERAY_HOME/osgi/state/`;
   5. Avviare Liferay Portal. Notare che ogni volta che si aggiungono e
      rimuovono .jars in questo modo, è necessario chiudere e riavviare Liferay
      Portal per rendere effettive le modifiche.

Se è necessario ripristinare le personalizzazioni, eliminare i file .jar di
sostituzione: Liferay Portal utilizzerà il file .jar originale al successivo
avvio.

Sembra tutto molto chiaro! Come facciamo a creare il nuovo bundle che conterrà
la patch dell’LPS-107983? Lo vediamo nel prossimo capitolo.

## 2 - Come creare il bundle con la patch
Per creare il nuovo bundle sono necessari i sorgenti e per ottenerli esistono
essenzialmente due modi che sono:
   1. Clone dei sorgenti di Liferay Portal 7.2.1 GA2. Per vedere come fare
      vedere il documento [Sorgenti Liferay con Scripts](https://git.smc.it/guidelines/development/blob/master/sorgenti-liferay-con-scripts.md#sorgenti-liferay-con-scripts);
   2. Ottenere i sorgenti dal repository
      Maven [com.liferay.portal.osgi.web.servlet.jsp.compiler](https://repo1.maven.org/maven2/com/liferay/com.liferay.portal.osgi.web.servlet.jsp.compiler/4.0.14/) versione 4.0.14.

La scelta della prima strada potrebbe richiedere parecchio tempo per via del
clone del repository che anche se eseguito con l’opzione `--depth 1` richiederà del
tempo non trascurabile.

Personalmente ho scelto la seconda strada, che in pochi minuti ci consentirà di
ottenere il bundle con la patch. A seguire tutti gli step necessari (e non
serve nessun IDE). In elenco i macro task:

   1. Creazione di un Liferay Workspace via blade;
   2. Creazione della struttura di directory per il modulo **portal-osgi-web-**
      **servlet-jsp-compiler**;
   3. Copiare i sorgenti dal bundle jar scaricato dal repository Maven
      all’interno del modulo su Liferay Workspace;
   4. Scaricare il bnd.bnd dal repository GitHub di Liferay e posizionarlo
      sulla root del modulo. Attività necessaria perchè sui sorgenti scaricati da Maven il file non c'è;
   5. Scaricare il build.gradle dal repository GitHub di Liferay e posizionarlo
      sulla root del modulo. Attività necessaria perchè sui sorgenti scaricati da Maven il file non c'è;
   6. Modificare il build.gradle per aggiungere la visibilità di FileUtil
      (com.liferay.gradle.util.FileUtil);
   7. Applicare la patch 84189;
   8. Eseguire la build del modulo;
   9. Accertarsi che il portale sia spento;
  10. Installare il modulo in `$LIFERAY_HOME/osgi/static`;
  11. Avviare il portale



Il set di comandi mostrati a seguire realizzano i punti da 1 a 5. Se volete
risparmiare anche questo tempo, allora potreste attingere direttamente al
repository Git che ho predisposto ad hoc sul nostro GitLab 
[https://git.smc.it/antonio.musarra/apply-lps-107983-openjdk-11-0-6](https://git.smc.it/antonio.musarra/apply-lps-107983-openjdk-11-0-6)



```shell script
$ mkdir ~/apply-lps-107983-openjdk-11-0-6
$ cd ~/apply-lps-107983-openjdk-11-0-6
$ blade init -v 7.2
$ mkdir modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler
$ cd modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler
$ mkdir -p src/main/java
$ mkdir -p src/main/resources
$ cd src/main/java
$ curl https://repo1.maven.org/maven2/com/liferay/com.liferay.portal.osgi.web.servlet.jsp.compiler/4.0.14/com.liferay.portal.osgi.web.servlet.jsp.compiler-4.0.14-sources.jar | jar xv
$ rm -rf META-INF/
$ rm -rf org/
$ mv javax ../resources
$ cd ../../../
$ curl -L -O https://github.com/liferay/liferay-portal/raw/7.2.1-ga2/modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler/bnd.bnd
$ curl -L -O https://github.com/liferay/liferay-portal/raw/7.2.1-ga2/modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler/build.gradle
```



Il punto 6 richiede la modifica del file build.gradle, questo è necessario
affinché non otteniate l’errore mostrato a seguire.



```shell script
Could not compile build file '/Users/antoniomusarra/Progetti/MyProjects/
Liferay/sources/apply-lps-107983-openjdk-11-0-6/modules/apps/static/portal-
osgi-web/portal-osgi-web-servlet-jsp-compiler/build.gradle'.
> startup failed:
  build file '/Users/antoniomusarra/Progetti/MyProjects/Liferay/sources/apply-
lps-107983-openjdk-11-0-6/modules/apps/static/portal-osgi-web/portal-osgi-web-
servlet-jsp-compiler/build.gradle': 1: unable to resolve class
com.liferay.gradle.util.FileUtil
   @ line 1, column 1.
     import com.liferay.gradle.util.FileUtil
     ^
```



Per evitare l’errore occorre aggiungere sul file build.gradle il contenuto
mostrato a seguire.



```
buildscript {
   dependencies {
      classpath group: "com.liferay", name: "com.liferay.gradle.util", version:
"1.0.36"
   }

   repositories {
      maven {
         url "https://repository-cdn.liferay.com/nexus/content/groups/public"
      }
   }
}
```



A questo punto siamo nelle condizioni di poter applicare la patch nel seguente
modo.



```shell script
$ cd ~/apply-lps-107983-openjdk-11-0-6
$ curl -O https://patch-diff.githubusercontent.com/raw/brianchandotcom/liferay-portal/pull/84189.patch
$ git apply --verbose 84189.patch
```



Se tutto va come deve, dovreste vedere un output simile a quello mostrato nella
figura seguente.



![ApplyPatch](docs/images/ApplyPatch.jpg)



Una volta applicata la patch possiamo procedere con la build del bundle utilizzando il
classico comando build.



```shell script
$ ./gradlew build
```



Dopo la build che dovrebbe concludersi con successo, dovreste trovare il bundle
all’interno della directory `$LIFERAY_WORKSPACE_HOME/modules/apps/static/portal-`
`osgi-web/portal-osgi-web-servlet-jsp-compiler/build/libs`



```shell script
$ ls -l modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler/build/libs
```


A portale spento, copiamo il nuovo bundle (senza la versione) all’interno della
directory `$LIFERAY_HOME/osgi/static/`



```shell script
$ cp modules/apps/static/portal-osgi-web/portal-osgi-web-servlet-jsp-compiler/build/libs/com.liferay.portal.osgi.web.servlet.jsp.compiler-4.0.14.jar /Users/antoniomusarra/dev/liferay/liferay-ce-portal-7.2.1-ga2-wildfly-16/osgi/static/com.liferay.portal.osgi.web.servlet.jsp.compiler.jar
```



A questo punto, dopo aver eliminato il contenuto della directory `$LIFERAY_HOME/
osgi/state/`, avviamo il portale. Fate attenzione ai log di Liferay perchè
dovreste vedere le seguenti righe di log che indicano l’avvenuta installazione
del nuovo bundle che va in override all’esistente.



```shell script
2020-03-24 22:17:48.836 INFO  [ServerService Thread Pool -- 73]
[ModuleFrameworkImpl:1468] Starting initial bundles
2020-03-24 22:17:49.234 INFO  [ServerService Thread Pool -- 73]
[ModuleFrameworkImpl:952] /Users/antoniomusarra/dev/liferay/liferay-ce-portal-
7.2.1-ga2-wildfly-16/osgi/marketplace/Liferay CE Static - Impl.lpkg:
com.liferay.portal.osgi.web.servlet.jsp.compiler-4.0.13.jar is overridden by /
Users/antoniomusarra/dev/liferay/liferay-ce-portal-7.2.1-ga2-wildfly-16/osgi/
static/com.liferay.portal.osgi.web.servlet.jsp.compiler.jar
```



Appena il portale è in piedi, eseguite il deploy di una qualunque portlet MVC e
verificate di non ottenere più errori di compilazione sulle JSP. Le figure a
seguire mostrano la home page del portale Liferay prima e dopo l’installazione
del bundle con la patch.




![LiferayPortalBeforPatch](docs/images/LiferayPortalBeforPatch.png)

![LiferayPortalAfterPatch](docs/images/LiferayPortalAfterPatch.png)



In questi casi, quando andiamo a fare il patch di componenti Liferay, è
consigliabile metterlo in evidenza, per esempio con opportuna indicazione sul
Bundle Name.



```shell script
g! lb com.liferay.portal.osgi.web.servlet.jsp.compiler
START LEVEL 20
   ID|State      |Level|Name
    1|Active     |    6|Liferay Portal OSGi Web Servlet JSP Compiler (SMC -
PATCHED-1) (4.0.14)|4.0.14
g!
```



Benissimo abbiamo finito e il Portale Liferay 7.2 GA2  funziona con
OpenJDK 11.0.6 🤓