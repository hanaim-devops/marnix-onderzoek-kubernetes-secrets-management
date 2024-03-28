# Kubernetes Secret Management: Het Geheime Ingrediënt voor Sterke Applicatiebeveiliging

*Marnix Wildeman*, Maart 2024
<br/>
Github URL: https://github.com/MarsTwix/marnix-onderzoek-kubernetes-secrets-management
<hr>

<img align="right" style="width: 140px;" src="https://in4it.com/wp-content/uploads/2020/11/secret.jpg">
<img align="right" style="width: 140px;" src="https://b2750956.smushcdn.com/2750956/wp-content/uploads/2021/01/3-31510_svg-kubernetes-logo-hd-png-download-696x664.png?lossy=1&strip=1&webp=1">

Ik volg op het moment de minor DevOps aan de Hogeschool van Arnhem en Nijmegen. Mij leek het erg interessant om een onderzoek te doen naar Kubernetes Secret Management. Ik heb dit onderwerp gekozen omdat ik het interessant vind om te leren hoe je gevoelige informatie veilig kunt opslaan en beheren in Kubernetes. Ik heb dit onderzoek gedaan om meer te leren over Kubernetes Secret Management en om mijn kennis te delen met anderen.

## Wat is Kubernetes Secret Management?

Eerst zullen we kijken naar wat Kubernetes precies is. Kubernetes is een draagbaar, uitbreidbaar, open source platform voor het beheren van gecontaineriseerde workloads en services, dat zowel declaratieve configuratie als automatisering mogelijk maakt. (Overview, 2024)

Het beheren van gecontaineriseerde workloads en services gaat als volgt in een Kubernetes cluster. Een Kubernetes-cluster bestaat uit ten minste één master node en meerdere working nodes. De nodes zijn machines (VM's of fysieke servers) waarop applicaties en workloads als containers worden uitgevoerd. Een cluster is een reeks van nodes die samenwerken om containerapplicaties uit te voeren. Binnen een cluster en een worker node worden containers geplaatst in pods, die de kleinste eenheid van Kubernetes zijn. (Ken Lee, 2024)

Om deze containers te laten communiceren met andere services, databases of API's, is het vaak nodig om gevoelige informatie zoals wachtwoorden, tokens of sleutels op te slaan. Kubernetes Secret Management is de praktijk van het veilig opslaan en beheren van gevoelige informatie binnen Kubernetes-clusters. Naar gevoelige informatie wordt verwezen als "secrets" in Kubernetes en hier zullen we verder op ingaan.

Een secret is een object dat een kleine hoeveelheid gevoelige gegevens bevat, zoals een wachtwoord, een token of een sleutel. Dergelijke informatie zou anders in een Pod-specificatie of in een containerimage kunnen worden geplaatst. Het gebruik van een secret betekent dat je geen vertrouwelijke gegevens in jouw applicatiecode hoeft op te nemen. (Secrets, 2023)

Omdat secrets onafhankelijk kunnen worden gemaakt van de pods die ze gebruiken, is er minder risico dat de secret (en de gegevens ervan) zichtbaar worden tijdens de workflow van het maken, bekijken en bewerken van pods. (Secrets, 2023)

Secrets worden base64-gecodeerd opgeslagen, maar het is belangrijk op te merken dat dit geen echte beveiliging biedt - het is meer een vorm van "security through obscurity". Daarom is effectief beheer van Secrets essentieel en zal later in deze blog nog besproken worden, wat de best practices zijn voor het beheren van Secrets.

### Soorten Secrets

#### Generic/Opaque Secrets

Dit zijn de meest voorkomende secrets en de standaard secret in Kubernetes. Ze kunnen willekeurige tekstgegevens opslaan, zoals gebruikersnamen en wachtwoorden, en worden vaak gebruikt om toegangsgegevens voor databases, API's en andere services op te slaan.

#### SSH Secrets

Deze secrets worden gebruikt om SSH-sleutels op te slaan die nodig zijn voor toegang tot andere servers. Ze kunnen worden gebruikt in pods om beveiligde verbindingen met externe servers tot stand te brengen.

#### TLS Secrets

TLS (Transport Layer Security) Secrets worden gebruikt om SSL/TLS-certificaten op te slaan. Deze certificaten worden vaak gebruikt voor het beveiligen van communicatie met services, zoals HTTPS-verbindingen.

#### Service Account Tokens

Elke pod in een Kubernetes-cluster heeft standaard toegang tot een serviceaccount-token. Deze tokens kunnen worden gebruikt om toegang te krijgen tot de Kubernetes API-server en andere Kubernetes-resources, zoals Secrets. Ze zijn handig voor het automatiseren van taken binnen een cluster.

#### Docker Registry Secrets

Deze secrets worden gebruikt om inloggegevens voor Docker-registries op te slaan. Dit stelt pods in staat om Docker-images te halen van privé registry die authenticatie vereisen.

## Aanmaken en gebruik van Secrets

We zullen nu gaan kijken hoe we secrets kunnen aanmaken en gebruiken. Het gebruik van Secrets geeft je meer flexibiliteit bij het definiëren van de Pod Life-cyclus en controle over hoe gevoelige gegevens worden gebruikt. Het vermindert het risico dat de gegevens aan ongeautoriseerde gebruikers worden blootgesteld. (Advocate, 2021)

Voordat we beginnen, moet `kubectl` zijn geïnstalleerd en geconfigureerd om te communiceren met jouw Kubernetes-cluster.

Voor MacOS kan dat als volgt in de terminal:

```bash
$ brew install kubectl
```

Voor Windows is het verstandig om deze [handleiding](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) te volgen.

### Via command line

Als eerste zal de methode worden uitgelegd om via de terminal of command line een secret aan te maken.

Als eerste stap zullen we twee bestanden aanmaken voor met gebruikersnaam en wachtwoord, waar we later naar verwijzen. Dit kan als volgt in de terminal:

```bash
echo -n 'root' > ./username.txt
echo -n 'Mq2D#(8gf09' > ./password.txt
```

Hierna kunnen we de secret aanmaken voor bijv. een database en verwijzen we naar de bestanden die we net hebben aangemaakt, die we graag in onze secret willen hebben. Deze secret zullen we `db-creds` noemen. Dit kan als volgt in de terminal:

```bash
$ kubectl create secret generic db-cerds \
  --from-file=./username.txt \
  --from-file=./password.txt
```

Het resultaat zal zijn:

```bash
secret/db-cerds created
```

Om te controleren of de "db-creds" secret is aangemaakt, zullen we een commando uitvoeren die alle secrets in de namespace laat zien. Dit kan als volgt in de terminal:

```bash
$ kubectl get secret/db-cerds
```

De db-cerds secret zal nu worden teruggegeven in de lijst:

```bash
NAME       TYPE      DATA      AGE
db-cerds   Opaque    2         26s
```

Om de details van de secret te bekijken, kunnen we het volgende commando uitvoeren:

```bash
$ kubectl describe secret/db-cerds
```

Zoals hieronder te zien is, bevat de secret twee bestanden: `username.txt` en `password.txt` en is van het type `Opaque`.

```bash

Name:         db-cerds
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  11 bytes
username.txt:  4 bytes
```

### Via YAML bestand

Via een YAML-bestand kunnen we ook een secret aanmaken. Dit kan handig zijn als je meerdere secrets wilt aanmaken bij het deployen van een applicatie. Deze YAML bestanden kunnen worden opgeslagen ergens waar alleen de DevOps team bij kan.

In dit scenario zullen we een secret aanmaken voor een database omgeving met een gebruikersnaam en wachtwoord.

Omdat een secret base64-gecodeerd is, moeten we de waarden van de secret eerst base64-encoderen.

Als eerste willen we de gebruikersnaam `root` base64-encoderen. Dit kan als volgt in de terminal:

```bash
$ echo -n 'root' | base64
```
Met als resultaat:

```bash
cm9vdA==
```

Vervolgens willen we het wachtwoord `Mq2D#(8gf09` base64-encoderen. Dit kan als volgt in de terminal:

```bash
$ echo -n 'Mq2D#(8gf09' | base64
```

Met als resultaat:

```bash
TXEyRCMoOGdmMDk=
```

Nu kunnen we een YAML-bestand aanmaken met bijv. de naam `creds.yml` en de secret zal de naam `database-creds` krijgen. Hierin komen de base64-gecodeerde waarden van de gebruikersnaam en het wachtwoord te staan en dit ziet er als volgt uit:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-creds
type: Opaque
data:
  username: cm9vdA==
  password: TXEyRCMojGdmMDk=
```

Nu we het YAML-bestand hebben aangemaakt, staat de secret nog niet in de Kubernetes cluster. Om de secret aan te maken, kunnen we het volgende commando uitvoeren:

```bash
$ kubectl create -f creds.yaml
```

Het resultaat zal zijn:

```bash
secret/database-creds created
```

Om te controleren of de "database-creds" secret is aangemaakt, zullen we een commando uitvoeren die alle secrets in de namespace laat zien. Dit kan als volgt in de terminal:

```bash
$ kubectl get secret/database-creds
```

De database-creds secret zal nu worden teruggegeven in de lijst:

```bash
NAME             TYPE      DATA      AGE
database-creds   Opaque    2         1m
```

Om de details van de secret te bekijken met als output in YAML, kunnen we het volgende commando uitvoeren:

```bash
$ kubectl get secret/database-creds -o yaml
```
Het volgende yaml format zal worden teruggegeven:

```bash
apiVersion: v1
data:
  password: TXEyRCMojGdmMDk=
  username: cm9vdA==
kind: Secret
metadata:
  creationTimestamp: "2023-10-05T17:45:17Z"
  name: database-creds
  namespace: default
  resourceVersion: "105750"
  uid: 26d540ba-6e44-43a0-ab3a-397f0aac46ff
type: Opaque
```

### Gebruik van secrets

Een secret kan op twee manieren worden gebruikt:

1. specificeer environment variables die verwijzen naar de waarden van het secret
2. mount een volume dat het secret bevat.

#### Environment variables

```yaml
--- 
apiVersion: v1
kind: Pod
metadata: 
  name: mysql-app
spec: 
  containers:
    - name: mysql
      image: mysql:5.7
      env:
        - name: MYSQL_USER
          valueFrom: 
            secretKeyRef: 
              key: username
              name: database-creds
        - name: MYSQL_PASSWORD
          valueFrom: 
            secretKeyRef: 
              key: password
              name: database-creds
      
```

#### Secret als een volume

```yaml
--- 
apiVersion: v1
kind: Pod
metadata: 
  name: redis-pod
spec: 
  containers: 
    - image: redis
      name: redis-pod
      volumeMounts: 
        - mountPath: /etc/dbcreds
          name: dbcreds
          readOnly: true
  volumes: 
    - name: dbcreds
      secret: 
        secretName: database-creds
```

## Good Practices

De volgende best practices zijn bedoeld voor zowel clusterbeheerders als applicatieontwikkelaars. Gebruik deze richtlijnen om de beveiliging van jouw gevoelige informatie in secret objecten te verbeteren en om jouw secrets effectiever te beheren. (Good practices for kubernetes secrets, 2023)

### Gebruik Namespaces

Gebruik namespaces om jouw secrets te isoleren en de toegang daartoe te beperken. Je kunt voor elk team of elke applicatie een aparte namespaces aanmaken en alleen toegang verlenen aan het noodzakelijke personeel. (Demaku, 2023)

### Beperk geheime toegang tot specifieke containers

Als je meerdere containers in een pod definieert en slechts één van die containers toegang tot een geheim nodig heeft, definieert je de configuratie van het volume of de environment variable zodat de andere containers geen toegang hebben tot dat secret.

### Bescherm geheime gegevens na het lezen

Applicaties moeten nog steeds de waarde van vertrouwelijke informatie beschermen nadat deze uit een environment variable of volume is gelezen. Jouw applicatie moet bijvoorbeeld voorkomen dat de geheime gegevens openbaar worden geregistreerd of naar een niet-vertrouwde partij worden verzonden. (Good practices for kubernetes secrets, 2023)

### Gebruik RBAC

Gebruik Role-Based Access Control (RBAC) om de toegang tot secrets te beheren. Gebruik RBAC om toegang te verlenen tot specifieke secrets of namespaces, en om de toegang tot gevoelige secrets te beperken. (Demaku, 2023)

### Gebruik extern secret management

Gebruik externe oplossingen voor secret management, zoals Hashicorp Vault of AWS Secrets Manager, om jouw secrets extern te beheren. Met deze aanpak kun je jouw secrets veilig opslaan en kan je Kubernetes gebruiken om ze op te halen wanneer dat nodig is. (Demaku, 2023)

### Verander secrets regelmatig

Rouleer jouw secrets regelmatig, bijvoorbeeld elke 30 dagen, om ervoor te zorgen dat ze veilig blijven. Gebruik automatiseringstools om dit proces te automatiseren en vermijd handmatige tussenkomst. (Demaku, 2023)

## Populaire Secret Management tools

Nu gaan we kijken naar enkele van de populaire tools voor secret management op de markt, die met Kubernetes kunnen worden gebruikt voor secret management. (Saurabh Gupta, 2022)

### HashiCorp Vault

HashiCorp Vault is een gratis en open-source tool voor het veilig opslaan en beheren van gevoelige gegevens, zoals tokens, certificaten en API-sleutels. Het biedt een uniforme interface voor het beheren van secrets. Vault ondersteunt traditionele en moderne methoden voor gebruikersauthenticatie en is een populaire keuze voor secrets management.

### Akeyless

Akeyless Vault is een centrale oplossing voor secret management, inclusief integratie met Kubernetes. Het gebruikt de Kubernetes Mutating Admission Webhook om secrets in pods te injecteren en biedt functies zoals Kubernetes-clusters kunnen gescheiden worden in verschillende omgevingen (denken aan test-, ontwikkelings- en productieomgevingen), SSO-ondersteuning en verschillende authenticatiemogelijkheden.

### CyberArk Conjur

CyberArk Secrets Manager is ontworpen voor Kubernetes en biedt secrets- en referentiebeheer voor cloud-native en containeromgevingen. Het verwijdert hardgecodeerde secrets uit broncode en biedt centraal beheer voor verschillende toepassingsworkloads.

### AWS Secrets Manager

AWS Secrets Manager is een AWS-service waarmee je snel database-inloggegevens, API-sleutels en andere wachtwoorden kunt veranderen, beheren en ophalen. Met deze Secrets Manager kun je secrets beveiligen, analyseren en beheren die nodig zijn om toegang te krijgen tot AWS Cloud-mogelijkheden, externe services en on-premise oplossingen.

### Google Cloud Secrets Manager

Secret Manager is een dienst van Google Cloud voor het beheren van secrets in kubernetes clusters. Het biedt een veilig opslagsysteem voor API-sleutels, certificaten, wachtwoorden en andere gevoelige gegevens. Het biedt een centrale plek en eén bron om secrets over Google Cloud te beheren, met functies zoals versiebeheer van secrets, integratie met cloud identity & access management en audit logging.

### Azure Key Vault

Azure Key Vault is een Microsoft Azure-service waarmee gebruikers cryptografische sleutels kunnen opslaan en gebruiken. Voor klantspecifieke sleutels biedt Azure Key Vault verschillende sleuteltypen en algoritmen, evenals het gebruik van Hardware Security Modules (HSM).

## Conclusie

Kubernetes Secret Management is de praktijk van het veilig opslaan en beheren van gevoelige informatie binnen Kubernetes-clusters. Dit omvat wachtwoorden, tokens en sleutels die anders in applicatiecode zouden worden opgenomen. Er zijn verschillende soorten Secrets die gebruikt kunnen worden, zoals Generic/Opaque Secrets, SSH Secrets, TLS Secrets, Service Account Tokens en Docker Registry Secrets.

Secrets worden aangemaakt en gebruikt via of de command line of YAML-bestanden. Ze kunnen als environment variables aan pods worden toegevoegd of als volumes worden gemount.

Best practices voor Kubernetes Secret Management omvatten het gebruik van namespaces voor isolatie, encryptie voor beveiliging, RBAC voor toegangsbeheer, het beperken van secret toegang tot specifieke containers, het gebruik van externe secret management-oplossingen zoals HashiCorp Vault, regelmatige verandering van secrets en bescherming van geheime gegevens na lezing.

Populaire tools voor Secret Management in Kubernetes zijn HashiCorp Vault, Akeyless, CyberArk Conjur, AWS Secrets Manager, Google Cloud Secrets Manager en Azure Key Vault. Effectief Secret Management is essentieel om de beveiliging en betrouwbaarheid van Kubernetes-toepassingen te waarborgen.

## Bronnen

Overview (2024) Kubernetes. Beschikbaar op: https://kubernetes.io/docs/concepts/overview/ (geraadpleegd: 28 maart 2024).

Ken Lee, K. (2024) How Does Kubernetes Work? Beschikbaar op: https://www.suse.com/c/how-does-kubernetes-work/ (geraadpleegd: 28 maart 2024).

Secrets (2023) Kubernetes. Beschikbaar op: https://kubernetes.io/docs/concepts/configuration/secret/ (geraadpleegd: 05 oktober 2023).

Advocate, K. (2021) Secrets management in kubernetes, Medium. Beschikbaar op: https://medium.com/avmconsulting-blog/secrets-management-in-kubernetes-378cbf8171d0 (geraadpleegd op 05 oktober 2023).

Good practices for kubernetes secrets (2023) Kubernetes. Beschikbaar op: https://kubernetes.io/docs/concepts/security/secrets-good-practices/ (geraadpleegd: 05 oktober 2023).

Demaku, A. (2023) Managing kubernetes secrets: Best practices and examples, Medium. Beschikbaar op: https://medium.com/@arton.demaku/managing-kubernetes-secrets-best-practices-and-examples-85eaeb6a6835 (geraadpleegd: 05 oktober 2023).

(Saurabh Gupta), S.G. (2022) Popular Secret Management Tools for Kubernetes workloads in 2022, Medium. Beschikbaar op: https://medium.com/@DevopsFollower/popular-secret-management-tools-for-kubernetes-workloads-in-2022-6d9027accf7c (geraadpleegd: 05 oktober 2023).
