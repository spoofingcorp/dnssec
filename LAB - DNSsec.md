### 📘 LAB Bind9 DNSSEC – Master/Slave + Résolution externe sécurisée

Ce guide vous accompagne dans la mise en place d'un serveur DNS Bind9 Master/Slave avec sécurisation DNSSEC pour votre zone interne et validation des zones externes.

-----

### **1. Pré-requis**

  * **Système** : Ubuntu Server 24.04 (ou une distribution similaire) sur deux machines.
  * **Réseau** :
      * Master DNS1 : `192.168.X.180`
      * Slave DNS2 : `192.168.X.181`
  * **Domaine interne** : `m2.dawan.lab`
  * **Ports ouverts** : TCP/UDP 53 entre les serveurs et vers les clients.


### Diagram

![](/picture/DNSsec.png)




### **2. Installation de Bind9**

Effectuez ces commandes sur les **deux serveurs** (Master et Slave).

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y

# Activer et démarrer le service Bind9
sudo systemctl enable --now named

# Vérifier que le service est bien actif
sudo systemctl status named
```

-----

### **3. Configuration du Master DNS1 (192.168.20.180)**

#### **3.1 Options globales**

Modifiez le fichier `/etc/bind/named.conf.options` pour définir le comportement général de votre serveur DNS.

```conf
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    dnssec-validation auto;

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    listen-on { 192.168.20.180; 127.0.0.1; };
    listen-on-v6 { none; };
};
```

#### **3.2 Création du dossier des zones**

```bash
sudo mkdir -p /etc/bind/zones
sudo chown bind:bind /etc/bind/zones
sudo chmod 750 /etc/bind/zones
```

#### **3.3 Création de la zone interne initiale**

Créez le fichier de zone `/etc/bind/zones/db.m2.dawan.lab`.

```conf
$TTL 604800
@       IN      SOA     ns1.m2.dawan.lab. admin.m2.dawan.lab. (
                              2025092201 ; Serial
                              604800     ; Refresh
                              86400      ; Retry
                              2419200    ; Expire
                              604800 )   ; Negative Cache TTL
;
@       IN      NS      ns1.m2.dawan.lab.
@       IN      NS      ns2.m2.dawan.lab.

ns1     IN      A       192.168.20.180
ns2     IN      A       192.168.20.181
www     IN      A       192.168.20.50
```

#### **3.4 Génération des clés DNSSEC**

Placez-vous dans le répertoire de vos zones pour y générer les clés.

```bash
cd /etc/bind/zones

# Génération de la Zone Signing Key (ZSK)
sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE m2.dawan.lab

# Génération de la Key Signing Key (KSK)
sudo dnssec-keygen -a ECDSAP256SHA256 -b 384 -n ZONE -f KSK m2.dawan.lab
```

#### **3.5 Intégration des clés publiques dans la zone 🔑**

1.  Listez les noms exacts des fichiers de clés publiques :
    ```bash
    ls /etc/bind/zones/K*.key
    ```
2.  Modifiez le fichier `/etc/bind/zones/db.m2.dawan.lab` pour insérer les directives `$INCLUDE` **au tout début du fichier**, juste après le `$TTL`.

✅ Le début de votre fichier de zone doit ressembler à ceci :

```conf
$TTL    604800

; --- Inclusion des clés publiques DNSSEC ---
$INCLUDE Km2.dawan.lab.+013+xxxxx.key 
$INCLUDE Km2.dawan.lab.+013+yyyyy.key  
; -----------------------------------------

@       IN      SOA     ns1.m2.dawan.lab. admin.m2.dawan.lab. (
                        ... (le reste du fichier)
```

**(⚠️ Remplacez `xxxxx` et `yyyyy` par les numéros de vos clés générées.)**

#### **3.6 Signature de la zone**

```bash
sudo dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -d' ' -f1) \
-N INCREMENT -o m2.dawan.lab -t db.m2.dawan.lab
```

Vérifiez que la signature est correcte :

```bash
dnssec-verify -o m2.dawan.lab /etc/bind/zones/db.m2.dawan.lab.signed
```

#### **3.7 Déclaration de la zone (Master)**

Modifiez `/etc/bind/named.conf.local` pour déclarer votre zone signée.

```conf
zone "m2.dawan.lab" {
    type master;
    file "/etc/bind/zones/db.m2.dawan.lab.signed";
    allow-transfer { 192.168.20.181; };
    also-notify { 192.168.20.181; };
};
```

#### **3.8 Redémarrage et validation (Master)**

```bash
sudo named-checkconf
sudo systemctl restart named
```

-----

### **4. Configuration du Slave DNS2 (192.168.20.181)**

La configuration de l'esclave est plus simple. Il doit pouvoir résoudre les requêtes externes (comme le maître) et savoir qu'il est esclave pour la zone `m2.dawan.lab`.

#### **4.1 Options globales (Slave)**

Modifiez le fichier `/etc/bind/named.conf.options` sur le serveur esclave. La configuration est presque identique à celle du maître, seule l'adresse IP dans `listen-on` change.

```conf
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-query { any; };
    dnssec-validation auto;

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // ⚠️ Assurez-vous d'utiliser l'IP du serveur esclave ici
    listen-on { 192.168.20.181; 127.0.0.1; };
    listen-on-v6 { none; };
};
```

#### **4.2 Déclaration de la zone (Slave)**

Modifiez le fichier `/etc/bind/named.conf.local` pour indiquer au serveur son rôle d'esclave pour la zone.

```conf
zone "m2.dawan.lab" {
    type slave;
    file "/var/cache/bind/db.m2.dawan.lab"; // Bind gérera ce fichier automatiquement
    masters { 192.168.20.180; }; // IP du serveur maître
};
```

#### **4.3 Redémarrage et validation (Slave)**

```bash
# Vérifier la syntaxe
sudo named-checkconf

# Appliquer la configuration
sudo systemctl restart named
```

Le serveur esclave va maintenant contacter le maître et télécharger la zone signée. Vous pouvez vérifier que le transfert a eu lieu en listant le contenu du répertoire cache de Bind :

```bash
ls -l /var/cache/bind/
```

➡️ Vous devriez y voir le fichier `db.m2.dawan.lab` apparaître après quelques instants.

-----

### **5. Vérifications finales ✅**

#### **5.1 Résolution interne signée**

```bash
dig +dnssec www.m2.dawan.lab @192.168.20.180
dig +dnssec www.m2.dawan.lab @192.168.20.181
```

➡️ La réponse doit contenir des enregistrements **`RRSIG`**.

#### **5.2 Résolution externe validée**

```bash
dig +dnssec www.cloudflare.com @192.168.20.180
dig +dnssec www.google.com @192.168.20.181
```

➡️ La réponse doit contenir le flag **`ad`** (Authenticated Data) dans les en-têtes.

-----

### **6. Bonnes pratiques et maintenance**

  * **Clés privées** : Les fichiers `.private` ne doivent exister **que sur le serveur maître**.
  * **Rotation des clés** : Prévoyez une rotation régulière de vos clés (ZSK tous les 3-6 mois, KSK tous les ans).
  * **Surveillance** : Gardez un œil sur les logs de Bind9.
    ```bash
    sudo journalctl -u named -f

    ```
