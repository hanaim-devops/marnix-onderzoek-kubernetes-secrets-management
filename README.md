# Kubernetes Secret Management: Het Geheime Ingrediënt voor Sterke Applicatiebeveiliging

*Marnix Wildeman*, Oktober 2023
<br/>
Github URL: https://github.com/MarsTwix/marnix-onderzoek-kubernetes-secrets-management
<hr>

<img align="right" style="width: 140px;" src="https://in4it.com/wp-content/uploads/2020/11/secret.jpg">
<img align="right" style="width: 140px;" src="https://b2750956.smushcdn.com/2750956/wp-content/uploads/2021/01/3-31510_svg-kubernetes-logo-hd-png-download-696x664.png?lossy=1&strip=1&webp=1">

Ik volg op het moment de minor DevOps aan de Hogeschool van Arnhem en Nijmegen. Mij leek het erg interessant om een onderzoek te doen naar Kubernetes Secret Management. Ik heb dit onderwerp gekozen omdat ik het interessant vind om te leren hoe je gevoelige informatie veilig kunt opslaan en beheren in Kubernetes. Ik heb dit onderzoek gedaan om meer te leren over Kubernetes Secret Management en om mijn kennis te delen met anderen.

## Wat is Kubernetes Secret Management?

Een secret is een object dat een kleine hoeveelheid gevoelige gegevens bevat, zoals een wachtwoord, een token of een sleutel. Dergelijke informatie zou anders in een Pod-specificatie of in een containerimage kunnen worden geplaatst. Het gebruik van een secret betekent dat je geen vertrouwelijke gegevens in jouw applicatiecode hoeft op te nemen. (Secrets, 2023)

Secrets worden base64-gecodeerd opgeslagen, maar het is belangrijk op te merken dat dit geen echte beveiliging biedt - het is meer een vorm van "security through obscurity". Daarom is effectief beheer van Secrets essentieel.

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

Deze secrets worden gebruikt om inloggegevens voor Docker-registries op te slaan. Dit stelt pods in staat om Docker-images te halen van prive registry die authenticatie vereisen.

## Aanmaken en gebruik van Secrets

We zullen nu gaan kijken hoe we secrets kunnen aanmaken en gebruiken. Het gebruik van Secrets geeft je meer flexibiliteit bij het definiëren van de Pod Life-cyclus en controle over hoe gevoelige gegevens worden gebruikt. Het vermindert het risico dat de gegevens aan ongeautoriseerde gebruikers worden blootgesteld. (Advocate, 2021)

### Via command line

Maak username.txt en password.txt tekst bestanden.

```bash
echo -n 'root' > ./username.txt
echo -n 'Mq2D#(8gf09' > ./password.txt
```

en

```bash
$ kubectl create secret generic db-cerds \
  --from-file=./username.txt \
  --from-file=./password.txt


Output:

secret/db-cerds created
```

Lijst van secrets:

```bash
$ kubectl get secret/db-cerds


Output:

NAME       TYPE      DATA      AGE
db-cerds   Opaque    2         26s
```

Bekijken van secret:

```bash
$ kubectl describe secret/db-cerds


Output:

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

De Secret bevat twee manieren: data en stringdata. Het data veld wordt gebruikt om willekeurige gegevens op te slaan, gecodeerd met base64.

```bash
$ echo -n 'root' | base64
Output: cm9vdA==

$ echo -n 'Mq2D#(8gf09' | base64
Output: TXEyRCMoOGdmMDk=
```

Aanmaken van een YAML bestand genaamd `creds.yaml` met de secret data.

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

Aanmaken van de secret door `kubectl create`

```bash
$ kubectl create -f creds.yaml

Output: 

secret/database-creds created
```

Lijst van secrets:

```bash
$ kubectl get secret/database-creds

Output:

NAME             TYPE      DATA      AGE
database-creds   Opaque    2         1m
```

Bekijken van secret:

```bash
$ kubectl get secret/database-creds -o yaml

Output:

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
  name: php-mysql-app
spec: 
  containers: 
    - 
      env: 
        - 
          name: MYSQL_USER
          valueFrom: 
            secretKeyRef: 
              key: username
              name: database-creds
        - 
          name: MYSQL_PASSWORD
          valueFrom: 
            secretKeyRef: 
              key: password
              name: database-creds
      image: "php:latest"
      name: php-app
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
    - 
      image: redis
      name: redis-pod
      volumeMounts: 
        - 
          mountPath: /etc/dbcreds
          name: dbcreds
          readOnly: true
  volumes: 
    - 
      name: dbcreds
      secret: 
        secretName: database-creds
```

## Good Practices

De volgende best practices zijn bedoeld voor zowel clusterbeheerders als applicatieontwikkelaars. Gebruik deze richtlijnen om de beveiliging van jouw gevoelige informatie in secret objecten te verbeteren en om jouw secrets effectiever te beheren. (Good practices for kubernetes secrets, 2023)

### Gebruik Namespaces

Gebruik namespaces om jouw secrets te isoleren en de toegang daartoe te beperken. Je kunt voor elk team of elke applicatie een aparte namespaces aanmaken en alleen toegang verlenen aan het noodzakelijke personeel. (Demaku, 2023)

### Beperk geheime toegang tot specifieke containers

Als je meerdere containers in een pod definieert en slechts één van die containers toegang tot een geheim nodig heeft, definieert je de configuratie van het volume of de environment variable zodat de andere containers geen toegang hebben tot dat secret.

### Configureer encryptie in rust

Standaard worden geheime objecten unencrypted opgeslagen in etcd. Je moet de codering van jouw geheime gegevens configureren in etcd. (Good practices for kubernetes secrets, 2023)

### Bescherm geheime gegevens na het lezen

Applicaties moeten nog steeds de waarde van vertrouwelijke informatie beschermen nadat deze uit een environment variable of volume is gelezen. Jouw applicatie moet bijvoorbeeld voorkomen dat de geheime gegevens openbaar worden geregistreerd of naar een niet-vertrouwde partij worden verzonden. (Good practices for kubernetes secrets, 2023)

### Gebruik RBAC

Gebruik Role-Based Access Control (RBAC) om de toegang tot secrets te beheren. Gebruik RBAC om toegang te verlenen tot specifieke secrets of namespaces, en om de toegang tot gevoelige secrets te beperken. (Demaku, 2023)

### Gebruik extern secret management

Gebruik externe oplossingen voor secret management, zoals Hashicorp Vault of AWS Secrets Manager, om jouw secrets extern te beheren. Met deze aanpak kunt je jouw secrets veilig opslaan en kunt je Kubernetes gebruiken om ze op te halen wanneer dat nodig is. (Demaku, 2023)

### Verander secrets regelmatig

Rouleer jouw secrets regelmatig, bijvoorbeeld elke 30 dagen, om ervoor te zorgen dat ze veilig blijven. Gebruik automatiseringstools om dit proces te automatiseren en vermijd handmatige tussenkomst. (Demaku, 2023)

## Populaire Secret Management tools

Nu gaan we kijken naar enkele van de populaire tools voor secret management op de markt, die met Kubernetes kunnen worden gebruikt voor secret management. (Saurabh Gupta, 2022)

### HashiCorp Vault

HashiCorp Vault is een gratis en open-source tool voor het veilig opslaan en beheren van gevoelige gegevens, zoals tokens, certificaten en API-sleutels. Het biedt een uniforme interface voor het beheren van secrets en kan gebruikersaudits genereren. Vault ondersteunt traditionele en moderne methoden voor gebruikersauthenticatie en is een populaire keuze voor secrets management.

### Akeyless

Akeyless Vault is een centrale oplossing voor secret management, inclusief integratie met Kubernetes. Het gebruikt de Kubernetes Mutating Admission Webhook om secrets in pods te injecteren en biedt functies zoals segregatie tussen Kubernetes-clusters, SSO-ondersteuning en verschillende authenticatiemogelijkheden.

### CyberArk Conjur

CyberArk Secrets Manager is ontworpen voor Kubernetes en biedt secrets- en referentiebeheer voor cloud-native en containeromgevingen. Het verwijdert hardgecodeerde secrets uit broncode en biedt centraal beheer voor verschillende toepassingsworkloads.

### AWS Secrets Manager

AWS Secrets Manager is een AWS-service waarmee je snel database-inloggegevens, API-sleutels en andere wachtwoorden kunt roteren, beheren en ophalen. Met Secrets Manager kun je secrets beveiligen, analyseren en beheren die nodig zijn om toegang te krijgen tot AWS Cloud-mogelijkheden, externe services en on-premises.

### Google Cloud Secrets Manager

Secret Manager is een dienst van Google Cloud voor het beheren van secrets op clusters. Het biedt een veilig opslagsysteem voor API-sleutels, certificaten, wachtwoorden en andere gevoelige gegevens. Het biedt een centrale plek en een enkele bron om secrets over Google Cloud te beheren, met functies zoals versiebeheer van secrets, integratie met cloud IAM en audit logging.

### Azure Key Vault

Azure Key Vault is een Microsoft Azure-service waarmee gebruikers cryptografische sleutels kunnen opslaan en gebruiken. Voor klantspecifieke sleutels biedt Azure Key Vault verschillende sleuteltypen en algoritmen, evenals het gebruik van Hardware Security Modules (HSM).

## Conclusie

Kubernetes Secret Management is de praktijk van het veilig opslaan en beheren van gevoelige informatie binnen Kubernetes-clusters. Dit omvat wachtwoorden, tokens en sleutels die anders in applicatiecode zouden worden opgenomen. Er zijn verschillende soorten Secrets die gebruikt kunnen worden, zoals Generic/Opaque Secrets, SSH Secrets, TLS Secrets, Service Account Tokens en Docker Registry Secrets.

Secrets worden aangemaakt en gebruikt via of de command line of YAML-bestanden. Ze kunnen als environment variables aan pods worden toegevoegd of als volumes worden gemount.

Best Practices voor de kubernetes secret management omvatten het gebruik van namespaces voor isolatie, encryptie voor beveiliging, RBAC voor toegangsbeheer, externe secret management oplossingen en regelmatige verandering van secrets.

Best practices voor Kubernetes Secret Management omvatten het gebruik van namespaces voor isolatie, encryptie voor beveiliging, RBAC voor toegangsbeheer, het beperken van secret toegang tot specifieke containers, het gebruik van externe secret management-oplossingen zoals HashiCorp Vault, regelmatige verandering van secrets en bescherming van geheime gegevens na lezing.

Populaire tools voor Secret Management in Kubernetes zijn HashiCorp Vault, Akeyless, CyberArk Conjur, AWS Secrets Manager, Google Cloud Secrets Manager en Azure Key Vault. Effectief Secret Management is essentieel om de beveiliging en betrouwbaarheid van Kubernetes-toepassingen te waarborgen.

## Bronnen

Secrets (2023) Kubernetes. Beschikbaar op: https://kubernetes.io/docs/concepts/configuration/secret/ (geraadpleegd: 05 oktober 2023).

Advocate, K. (2021) Secrets management in kubernetes, Medium. Beschikbaar op: https://medium.com/avmconsulting-blog/secrets-management-in-kubernetes-378cbf8171d0 (geraadpleegd op 05 oktober 2023).

Good practices for kubernetes secrets (2023) Kubernetes. Beschikbaar op: https://kubernetes.io/docs/concepts/security/secrets-good-practices/ (geraadpleegd: 05 oktober 2023).

Demaku, A. (2023) Managing kubernetes secrets: Best practices and examples, Medium. Beschikbaar op: https://medium.com/@arton.demaku/managing-kubernetes-secrets-best-practices-and-examples-85eaeb6a6835 (geraadpleegd: 05 oktober 2023).

(Saurabh Gupta), S.G. (2022) Popular Secret Management Tools for Kubernetes workloads in 2022, Medium. Beschikbaar op: https://medium.com/@DevopsFollower/popular-secret-management-tools-for-kubernetes-workloads-in-2022-6d9027accf7c (geraadpleegd: 05 oktober 2023).
