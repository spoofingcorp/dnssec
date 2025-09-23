# Cours sur DNSSEC : Sécuriser le carnet d'adresses d'Internet

DNSSEC (Domain Name System Security Extensions) est un ensemble d'extensions de sécurité qui renforce l'authenticité des réponses DNS. Pour comprendre son importance, il faut d'abord se souvenir de la faiblesse fondamentale du DNS : il a été conçu dans les années 80 en partant du principe que les échanges sur Internet étaient basés sur la confiance.

Le DNS est comme un immense annuaire téléphonique. Quand vous tapez `www.google.fr`, votre ordinateur demande à un serveur DNS l'adresse IP correspondante. Le problème est que, sans DNSSEC, un pirate peut intercepter cette demande et vous donner une fausse adresse IP, vous redirigeant vers un site malveillant. C'est ce qu'on appelle **l'empoisonnement du cache DNS (DNS Cache Poisoning)**.

DNSSEC ne chiffre pas les données, mais il agit comme un **sceau d'inviolabilité numérique**. Il garantit que la réponse que vous recevez provient bien du bon expéditeur (**authenticité**) et qu'elle n'a pas été modifiée en chemin (**intégrité**).

-----

### Les Piliers de DNSSEC

DNSSEC repose sur les principes de la cryptographie asymétrique (clés publiques/privées) et introduit de nouveaux types d'enregistrements DNS.

#### **1. Les Clés Cryptographiques 🔑**

Chaque zone DNS sécurisée utilise deux paires de clés pour séparer les responsabilités et améliorer la sécurité.

  * **KSK (Key Signing Key - Clé de Signature de Clé)** : C'est la "clé du coffre-fort". La clé privée de la KSK est très sensible et est utilisée rarement, uniquement pour signer la clé publique de la ZSK. Sa clé publique est l'élément qui est validé par la zone parente.

  * **ZSK (Zone Signing Key - Clé de Signature de Zone)** : C'est la "clé des documents". La clé privée de la ZSK est utilisée beaucoup plus fréquemment pour signer tous les enregistrements de la zone (A, MX, CNAME, etc.). Comme elle est plus exposée, on la change plus souvent.

-----

#### **2. Les Signatures Numériques (RRSIG)**

Pour chaque enregistrement dans une zone DNSSEC, une signature numérique est créée en utilisant la clé privée de la ZSK. Cette signature est stockée dans un nouvel enregistrement appelé **RRSIG (Resource Record Signature)**.

Quand un résolveur DNS demande l'adresse IP de `www.exemple.fr`, le serveur DNS lui renvoie non seulement l'enregistrement `A` mais aussi l'enregistrement `RRSIG` associé. Le résolveur peut alors utiliser la clé publique de la ZSK pour vérifier que la signature correspond bien à l'enregistrement `A`.

-----

#### **3. La Chaîne de Confiance (DS Record)**

C'est le concept le plus important. Comment faire confiance à la clé KSK d'un domaine ? En faisant en sorte que son **parent** la valide.

L'enregistrement **DS (Delegation Signer)** est une empreinte numérique (un *hash*) de la clé KSK publique d'une zone enfant, qui est stockée et signée dans la **zone parente**. Cette relation crée une chaîne de confiance ininterrompue, de la zone racine (`.`) jusqu'au domaine final.

-----

### Le Fonctionnement : La Validation étape par étape 🔎

Voici comment un résolveur DNS (comme celui de votre FAI ou `8.8.8.8`) valide une requête pour `www.exemple.fr` :

1.  **Question** : Votre ordinateur demande l'adresse de `www.exemple.fr` au résolveur.
2.  **Racine de la Confiance** : Le résolveur, qui connaît la clé publique de la zone racine (`.`), interroge les serveurs racine.
3.  **Validation du TLD (.fr)** : Il obtient l'enregistrement DS de la zone `.fr` auprès de la racine, ce qui lui permet de valider la KSK de `.fr`.
4.  **Validation du Domaine (exemple.fr)** : Il obtient l'enregistrement DS de `exemple.fr` auprès de `.fr`, ce qui lui permet de valider la KSK de `exemple.fr`.
5.  **Validation des Clés du Domaine** : La KSK validée de `exemple.fr` lui permet de vérifier la signature de la ZSK du domaine.
6.  **Validation de l'Enregistrement Final** : La ZSK validée lui permet de vérifier la signature `RRSIG` de l'enregistrement `A` pour `www.exemple.fr`.
7.  **Réponse Sécurisée** : Si tout est valide, le résolveur renvoie la réponse avec le drapeau **AD (Authenticated Data)**. Sinon, il retourne une erreur (`SERVFAIL`).

-----

### DNSSEC en Pratique avec BIND9 🛠️

BIND9 est le logiciel serveur DNS le plus utilisé et il intègre une suite complète d'outils pour gérer le cycle de vie de DNSSEC.

#### **1. Activation et Configuration**

La mise en œuvre de DNSSEC dans BIND9 se fait à deux niveaux :

  * **Pour le résolveur (côté client)** : Pour que votre serveur BIND9 valide les réponses DNS pour vos clients, il suffit d'activer la validation dans `/etc/bind/named.conf.options` :

    ```conf
    options {
        dnssec-validation auto;
    };
    ```

    Le mode `auto` s'appuie sur une clé racine de confiance (`trust anchor`) gérée automatiquement par BIND.

  * **Pour le serveur autoritaire (côté zone)** : Pour sécuriser votre propre zone, vous devez indiquer à BIND d'utiliser le fichier de zone signé. Dans `/etc/bind/named.conf.local`, la directive `file` pointera vers le fichier `.signed` :

    ```conf
    zone "exemple.fr" {
        type master;
        file "/etc/bind/zones/db.exemple.fr.signed";
    };
    ```

-----

#### **2. La Boîte à Outils DNSSEC de BIND**

BIND fournit des commandes spécifiques pour chaque étape de la sécurisation d'une zone.

  * **`dnssec-keygen`** : C'est l'utilitaire qui **génère vos paires de clés** (KSK et ZSK). Pour chaque clé, il crée deux fichiers : un fichier `.key` contenant la clé publique et un fichier `.private` contenant la clé privée. **La clé privée ne doit jamais être partagée.**

  * **`dnssec-signzone`** : C'est le **cœur du processus de signature**. Cet outil prend votre fichier de zone standard, lit les clés publiques que vous y avez incluses via `$INCLUDE`, utilise les clés privées correspondantes pour signer chaque enregistrement, et génère un nouveau fichier de zone complet (`.signed`) contenant les enregistrements **RRSIG** et **NSEC/NSEC3**.

  * **Utilitaires de maintenance** : BIND inclut d'autres outils pour faciliter la gestion, comme `dnssec-dsfromkey` pour générer l'enregistrement DS à transmettre à votre registrar, ou `dnssec-settime` pour gérer les métadonnées temporelles des clés.

En résumé, le processus avec BIND est le suivant : **générer les clés** (`dnssec-keygen`), **les inclure** dans le fichier de zone, **signer la zone** (`dnssec-signzone`), **charger la zone signée** dans BIND, et enfin **publier l'enregistrement DS** chez le parent.

-----

### En Résumé : Que Fait (et ne Fait Pas) DNSSEC ?

#### **Ce que DNSSEC garantit :**

  * ✅ **Authenticité de l'origine** : Vous êtes sûr que la réponse provient du serveur qui fait autorité.
  * ✅ **Intégrité des données** : La réponse n'a pas été altérée durant le transport.
  * ✅ **Déni d'existence authentifié** : DNSSEC peut prouver de manière cryptographique qu'un nom de domaine n'existe pas.

#### **Ce que DNSSEC ne garantit PAS :**

  * ❌ **Confidentialité** : Les requêtes et les réponses DNS restent en clair. Pour le chiffrement, il faut utiliser d'autres protocoles comme DNS over TLS (DoT) ou DNS over HTTPS (DoH).
  * ❌ **Protection contre les attaques DDoS** : Il ne protège pas les serveurs DNS contre les attaques par déni de service.



# Comment reconnaître une réponse DNSSEC ? 🕵️

Analyser une réponse DNS pour y déceler les preuves de la validation DNSSEC est simple une fois que l'on sait où regarder. L'outil `dig`, avec l'option `+dnssec`, est parfait pour cela. Cette option demande au serveur DNS de nous renvoyer les enregistrements liés à la sécurité.

Prenons l'exemple de votre requête vers `www.cloudflare.com`.

```bash
dig +dnssec www.cloudflare.com @192.168.20.180

; <<>> DiG 9.18.39-0ubuntu0.24.04.1-Ubuntu <<>> +dnssec www.cloudflare.com @192.168.20.180
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44055
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
...
;; ANSWER SECTION:
www.cloudflare.com.     300     IN      A       104.16.124.96
www.cloudflare.com.     300     IN      A       104.16.123.96
www.cloudflare.com.     300     IN      RRSIG   A 13 3 300 20250924075954...
```

Il y a deux indicateurs clés à observer.

-----

### **7. Le flag `ad` dans l'en-tête (HEADER)**

C'est la preuve la plus importante. Regardez la ligne `flags` dans la section `HEADER` :

```
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
```

Le flag **`ad`** signifie **Authenticated Data** (Données Authentifiées).

Sa présence indique que **votre résolveur DNS** (ici, `192.168.20.180`) a effectué avec succès toute la validation cryptographique de la chaîne de confiance pour `www.cloudflare.com`. Il vous certifie que les enregistrements de la section `ANSWER` sont authentiques et n'ont pas été modifiés.

**Analogie :** Le flag `ad` est comme le tampon "VÉRIFIÉ" apposé par un notaire sur un document. Ce n'est pas le document lui-même, mais la confirmation par un tiers de confiance que tout est en ordre.

-----

### **7.2. L'enregistrement `RRSIG` dans la réponse (ANSWER)**

Dans la section `ANSWER`, en plus des enregistrements `A` (les adresses IP), vous voyez un enregistrement **`RRSIG`**.

```
www.cloudflare.com.     300     IN      RRSIG   A 13 3 300 ...
```

**RRSIG** signifie **Resource Record Signature**. C'est la **signature numérique** des enregistrements `A`.

Sa présence montre que la zone `cloudflare.com` est **signée par son propriétaire**. C'est cette signature que votre résolveur a vérifiée pour pouvoir ensuite positionner le flag `ad`.

**Analogie :** Si les enregistrements `A` sont le contenu d'une lettre, l'enregistrement `RRSIG` est le sceau de cire apposé par l'expéditeur.

-----

### 7.3 Conclusion

Pour confirmer qu'une réponse est bien sécurisée par DNSSEC, vous devez chercher ces deux éléments :

  * **L'enregistrement `RRSIG`** : prouve que la zone d'origine est signée.
  * **Le flag `ad`** : prouve que **votre résolveur** a validé cette signature et vous garantit son authenticité.

Obtenir les deux est le signe d'un fonctionnement parfait de la chaîne DNSSEC.
