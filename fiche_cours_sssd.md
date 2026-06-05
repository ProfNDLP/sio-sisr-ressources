# SSSD — System Security Services Daemon
### Intégration Linux au domaine Active Directory
**BTS SIO SISR** | Debian 13 · Windows Server 2019 · Domaine : `sio.ndlp` · DC : `10.138.100.250`

---

## 1. Rôle et principe de SSSD

SSSD est le démon qui permet à un poste Linux de s'authentifier auprès d'un annuaire distant (ici Active Directory). Sans SSSD, Linux ne connaît que les comptes locaux de `/etc/passwd`. Avec SSSD, il interroge l'AD via LDAP et Kerberos pour valider les identités.

| | Sans SSSD | Avec SSSD |
|---|---|---|
| Authentification | Locale uniquement | Déléguée à l'AD |
| Comptes | Dans `/etc/passwd` et `/etc/shadow` | Comptes AD visibles comme comptes locaux |
| Connexion domaine | Impossible | Login/mdp du domaine sio.ndlp |

---

## 2. Architecture — comment SSSD s'intègre

```
┌─────────────────────────────────────────┐
│           Poste Debian 13               │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  PAM — Pile d'authentification   │   │
│  └──────────────────┬───────────────┘   │
│                     │                   │
│  ┌──────────────────▼───────────────┐   │
│  │  SSSD daemon — Cache + proxy     │   │──── LDAP (TCP 389) ──► ┌─────────────────┐
│  └────────┬─────────────────┬───────┘   │                        │  Active Directory│
│           │                 │           │──── Kerberos (UDP 88) ─►│  10.138.100.250  │
│  ┌────────▼──────┐  ┌───────▼───────┐   │                        │  ┌─────┐ ┌─────┐│
│  │  NSS          │  │  Kerberos     │   │                        │  │LDAP │ │ KDC ││
│  │  Résolution   │  │  Tickets auth │   │                        │  └─────┘ └─────┘│
│  └───────────────┘  └───────────────┘   │         pfSense        └─────────────────┘
└─────────────────────────────────────────┘
```

> **Kerberos** gère l'**authentification** (vérification de l'identité via tickets chiffrés), tandis que **LDAP** gère l'**annuaire** (récupération des informations : groupes, UID, shell, home).

---

## 3. Les composants de SSSD

| Processus | Rôle | Protocole |
|---|---|---|
| `sssd` | Processus principal, orchestre les sous-services | — |
| `sssd_be` | Backend — communique avec l'AD (requêtes LDAP) | LDAP TCP/389 |
| `sssd_nss` | Name Service Switch — résout les noms d'utilisateurs et groupes | Socket Unix |
| `sssd_pam` | Authentification PAM — valide les mots de passe | Socket Unix |
| `sssd_pac` | Kerberos PAC — gère les tickets et attributs | UDP/TCP 88 |
| `krb5_child` | Processus enfant pour Kerberos (pré-authentification) | UDP 88 |

Vérifier les processus actifs :

```bash
sudo systemctl status sssd
# Attendu : 5 processus enfants listés
```

---

## 4. Fichier de configuration — /etc/sssd/sssd.conf

C'est le fichier central. Il doit obligatoirement être en `chmod 600` (propriété de root) sinon SSSD refuse de démarrer.

```ini
# Section globale
[sssd]
domains              = sio.ndlp
config_file_version  = 2
services             = nss, pam

# Section domaine — une section par domaine AD
[domain/sio.ndlp]
id_provider               = ad          # Source : Active Directory
access_provider           = ad          # Contrôle d'accès via l'AD
ad_domain                 = sio.ndlp
krb5_realm                = SIO.NDLP    # Majuscules obligatoires
fallback_homedir          = /home/%u    # %u = nom d'utilisateur
default_shell             = /bin/bash
use_fully_qualified_names = False       # Permet "sio1" au lieu de "sio1@sio.ndlp"
login_formats             = %u
ldap_id_mapping           = True        # Génère UID Linux depuis SID Windows
cache_credentials         = True        # Connexion possible hors ligne
```

> ⚠️ **`fallback_homedir = /home/%u` est critique.** Si on met `/home/%u@%d`, les homes sont créés sous `/home/sio1@sio.ndlp` au lieu de `/home/sio1` et les scripts de montage échouent.

---

## 5. Le mécanisme d'authentification pas à pas

```
Étape 1 ── L'utilisateur saisit login/mdp dans SDDM
             Exemple : login "sio1", mdp "MonMotDePasse"
                │
Étape 2 ── PAM appelle pam_sss.so → transmet les identifiants à SSSD
             Défini dans /etc/pam.d/common-auth
                │
Étape 3 ── SSSD (sssd_pam + krb5_child) envoie une requête Kerberos au KDC
             Port UDP 88 vers 10.138.100.250 — doit être ouvert dans pfSense
                │
Étape 4 ── Le KDC répond avec un TGT si le mot de passe est correct
             En cas d'échec : "Preauthentication failed" dans les logs
                │
Étape 5 ── SSSD récupère les informations de l'utilisateur via LDAP
             UID, groupes, home — Port TCP 389 vers 10.138.100.250
                │
Étape 6 ── PAM ouvre la session
             pam_mkhomedir crée /home/sio1 si absent
             Les scripts pam_exec montent les lecteurs et configurent sudo
                │
Étape 7 ── KDE Plasma démarre dans la session de sio1
             Les scripts autostart (Thunar, premier-login) s'exécutent
```

---

## 6. Le cache SSSD

SSSD met en cache les identités AD dans `/var/lib/sss/db/`. Ce cache permet la connexion même si le DC est hors ligne (`cache_credentials = True`). Mais il peut poser problème pour les nouveaux comptes AD qui ne sont pas encore connus du cache.

| | Avantage | Inconvénient |
|---|---|---|
| Cache actif | Connexion possible si le DC est temporairement inaccessible | Un nouveau compte AD n'est pas visible tant que le cache n'a pas été synchronisé |

Forcer la synchronisation après création d'un compte AD :

```bash
sudo systemctl stop sssd
sudo rm -rf /var/lib/sss/db/*
sudo systemctl start sssd
id sio5   # Vérifier que le compte est visible
```

---

## 7. NSS — résolution des identités

NSS (Name Service Switch) est la couche qui répond aux questions "qui est cet utilisateur ?". Sans SSSD, Linux ne consulte que `/etc/passwd`. Avec SSSD, il consulte également l'AD via `libnss-sss`.

| Commande | Ce qu'elle interroge | Exemple avec sio.ndlp |
|---|---|---|
| `id sio1` | UID, GID, groupes | `uid=550001106(sio1) gid=550000513(...) groupes=...,550001104(groupe_sio1)` |
| `getent passwd sio1` | Entrée complète | `sio1:x:550001106:550000513::/home/sio1:/bin/bash` |
| `groups sio1` | Liste des groupes | `sio1 : utilisateurs du domaine groupe_sio1` |

> ✅ **Minuscules !** SSSD convertit automatiquement les noms de groupes AD en minuscules sous Linux. `GROUPE_SIO1` devient `groupe_sio1`. Tous les scripts (`monter-classe.sh`, `configurer-sudo.sh`) utilisent les noms en minuscules.

---

## 8. Kerberos — l'authentification par tickets

Kerberos est le protocole d'authentification utilisé par Active Directory. Au lieu de transmettre le mot de passe sur le réseau, il utilise des **tickets chiffrés** à durée limitée.

| Terme | Signification | Dans notre infra |
|---|---|---|
| `KDC` | Key Distribution Center — le DC Windows joue ce rôle | 10.138.100.250 |
| `TGT` | Ticket Granting Ticket — preuve d'identité initiale | Durée : 10 heures |
| `Realm` | Nom du domaine Kerberos (toujours en MAJUSCULES) | `SIO.NDLP` |

Tester manuellement un ticket Kerberos :

```bash
kinit administrateur@SIO.NDLP   # Obtenir un ticket
klist                            # Afficher les tickets actifs
# Attendu :
# Default principal: administrateur@SIO.NDLP
# Valid starting: 04/06/2026 18:13  Expires: 05/06/2026 04:13
```

> 🔴 **Synchronisation horaire obligatoire.** Kerberos tolère au maximum 5 minutes d'écart entre le poste et le DC. Si l'horloge dévie, l'authentification échoue silencieusement. NTP doit être configuré (port UDP 123 ouvert dans pfSense).

---

## 9. Points de vérification rapide

| Vérification | Commande | Résultat attendu |
|---|---|---|
| Jonction au domaine | `sudo realm list` | `configured: kerberos-member` |
| SSSD actif | `sudo systemctl status sssd` | `active (running)` |
| Résolution user | `id sio1` | uid + groupe_sio1 visibles |
| Kerberos fonctionnel | `kinit user@SIO.NDLP` | Pas d'erreur, ticket dans klist |
| Partage SMB accessible | `smbclient //10.138.100.250/echange -U svc-montage%...` | Invite `smb: \>` |
| Logs d'erreur SSSD | `sudo journalctl -u sssd -f` | Aucun `Preauthentication failed` |

---

*Lycée Notre-Dame de La Providence — BTS SIO SISR — Avranches*
