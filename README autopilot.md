# Lab Entra ID & Intune : gestion d'un poste Windows 11 de bout en bout

<img width="2752" height="1536" alt="Gemini_Generated_Image_wi2wg3wi2wg3wi2w" src="https://github.com/user-attachments/assets/d2d98aa9-4191-4977-b244-3d1b8ed24aca" />


---

## 1. Présentation

**Objectif du lab** : mettre en place une chaîne de gestion moderne d'un poste Windows 11, depuis l'identité cloud jusqu'à l'accès conditionnel.

**Ce que le lab démontre**
- Identités et groupes dynamiques dans Microsoft Entra ID
- Déploiement Windows Autopilot (piloté par l'utilisateur, jonction Microsoft Entra)
- Conformité des appareils et chiffrement BitLocker via Intune
- Packaging et déploiement d'une application Win32
- Configuration par Settings Catalog et Update Rings
- Accès conditionnel : « Exiger un appareil conforme »

**Contexte** : lab réalisé sur un tenant Microsoft 365 avec un essai E5 d'un mois, dans une série de labs d'infrastructure (AD/GPO, SCCM, packaging) orientée vers un poste d'ingénieur systèmes.

**Source d'inspiration** : ce lab s'inspire du dépôt [`DiCR77/Entra-Intune-Enterprise-Architecture`](https://github.com/DiCR77/Entra-Intune-Enterprise-Architecture). Je n'ai pas modifié sa conception : je l'ai réalisé sur mon propre tenant et documenté étape par étape, avec les problèmes rencontrés.

---

## 2. Architecture et prérequis
┌─────────────────────────────────────────────────────────────────────────────┐
│                         🌐 TENANT MICROSOFT 365                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────┐           ┌─────────────────┐           ┌─────────────────┐
│   🔷 Entra ID  │           │  📦 Intune MDM   │           │  🔄 Autopilot   │
│   Identités    │◄─────────►│   Gestion        │◄─────────►│   Provisioning  │
│   & Groupes    │           │   Appareils      │           │   OOBE          │
└───────────────┘           └─────────────────┘           └─────────────────┘
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ 👥 Groupes    │           │ 📋 Stratégies   │           │ 🖥️ ESP          │
│ Dynamiques    │           │ Conformité      │           │ (Enrollment     │
│ (Départements)│           │ & Configuration │           │  Status Page)   │
└───────────────┘           └─────────────────┘           └─────────────────┘
        │                             │                             │
        └─────────────────────────────┼─────────────────────────────┘
                                      ▼
                    ┌─────────────────────────────────┐
                    │      🛡️ ACCÈS CONDITIONNEL       │
                    │   Zéro Trust - Appareil Conforme  │
                    └─────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ 📱 Applications│           │  🔒 Sécurité    │           │  🔄 Maintenance  │
│ M365 + Win32  │           │  BitLocker      │           │  WUfB Rings     │
│               │           │  Defender       │           │  Feature/QoL    │
└───────────────┘           └─────────────────┘           └─────────────────┘

graph TD
    A[👤 Utilisateur] -->|Authentification| B[🔷 Entra ID]
    B -->|Token JWT| C[🛡️ Accès Conditionnel]
    C -->|Vérification conformité| D[📦 Intune]
    D -->|État appareil| C
    C -->|Autorisation| E[☁️ Microsoft 365]

    F[🖥️ Windows 11 OOBE] -->|Autopilot| G[🔄 ESP]
    G -->|Profils| H[📋 Configuration]
    G -->|Apps| I[📱 Applications]
    G -->|Sécurité| J[🔒 BitLocker/Defender]

    K[⏰ Windows Update] -->|Anneaux| L[🔄 WUfB]
    L -->|Patchs| F

### 2.1 Schéma d'ensemble
📸 *Schéma simple : tenant → groupes (utilisateurs / appareils) → profils Autopilot, conformité, configuration → VM Windows 11 → accès conditionnel*

### 2.2 Prérequis
| Élément | Détail |
|---|---|
| Tenant | Microsoft 365 avec essai **E5** (Intune, Entra ID P1/P2) |
| Hyperviseur | VMware Workstation |
| VM | Windows 11 Pro/Enterprise, **UEFI + Secure Boot + vTPM**, 4 Go RAM, 2 vCPU, 64 Go, réseau **NAT** |
| Outils | PowerShell 7, module `Microsoft.Graph`, `IntuneWinAppUtil.exe` |
| Comptes | Administrateur général, **compte de secours (break-glass)** |

### 2.3 Conventions de nommage
| Objet | Nom |
|---|---|
| Groupe utilisateurs IT | `GRP-IT-Techniciens` |
| Groupes RH / Finance | `RH` et `Finance` |
| Groupe appareils Autopilot | `GRP-Autopilot-Devices` |
| Profil Autopilot | `Autopilot_Entra_user` |
| Conformité | `secuIT` |
| Chiffrement | `BitLocker-OS-Lab` |
| Update Ring | `Update-Ring-IT` |
| Appareil | `LAB-%SERIAL%` |

---

## 3. Étape 1 : Tenant et identités

### 3.1 Licences
- Vérifier la présence des licences E5 (Intune, Entra ID P1/P2).
- 📸 *Page des licences avec l'essai E5*

### 3.2 Création de 10 utilisateurs avec Microsoft Graph
- Connexion : `Connect-MgGraph -Scopes "User.ReadWrite.All"`
- Création en boucle avec `New-MgUser` (liste de 10 utilisateurs de test).
- 📸 *Sortie PowerShell : « Utilisateurs créés : 10 »*

### 3.3 Attributs et licences
- Renseigner **Department**, **JobTitle** et **UsageLocation** (obligatoire pour la licence) avec `Update-MgUser`.
- Attribuer la licence E5 avec `Set-MgUserLicense`.
- Répartition : **4 utilisateurs IT / Technicien**, **3 RH** (Chargé(e) RH), **3 Finance** (Comptable).
- Compte supplémentaire `admintest` : renseigné en IT / Technicien pour entrer dans `GRP-IT-Techniciens` (le groupe étant dynamique, on ne peut pas y ajouter de membre à la main).
- 📸 *Liste des utilisateurs avec service et poste*

### 3.4 Groupes dynamiques
- `GRP-IT-Techniciens` : `(user.department -eq "IT") -and (user.jobTitle -eq "Technicien")`
- Groupes RH et Finance : règles sur `user.department`.
- 📸 *Règle dynamique et liste des membres*

### 3.5 Compte de secours
- Compte cloud uniquement, Administrateur général, mot de passe long, **exclu de toutes les stratégies d'accès conditionnel**.
- Le mot de passe n'est **pas** stocké dans le dépôt.

---

## 4. Étape 2 : Windows Autopilot

### 4.1 Préparation de la VM
- Création sous VMware : UEFI, **Secure Boot activé**, vTPM (chiffrement des fichiers nécessaires au TPM uniquement).
- Arrêt à l'écran de région de l'OOBE, puis **snapshot** `OOBE-propre`.
- 📸 *Paramètres VMware (TPM, UEFI, Secure Boot)*

### 4.2 Portée MDM (point critique)
- *Entra ID → Mobilité (MDM et WIP) → Microsoft Intune* : portée utilisateur MDM sur **Tout**.
- 📸 *Page de la portée MDM*

### 4.3 Groupe d'appareils et profil de déploiement
- Groupe dynamique d'appareils `GRP-Autopilot-Devices` :
  `(device.devicePhysicalIds -any (_ -startsWith "[ZTDId]"))`
- Profil `Autopilot_Entra_user` : piloté par l'utilisateur, **joint à Microsoft Entra**, compte **Standard**, CLUF et confidentialité masqués, modèle de nom `LAB-%SERIAL%`.
- Option « Convertir tous les appareils ciblés en Autopilot » : **Non**.
- Attribution au groupe d'appareils. **Ne pas exclure** un groupe d'utilisateurs dans un profil Autopilot.
- 📸 *Propriétés du profil et attributions*

### 4.4 Page d'état de l'inscription (ESP)
- Afficher la progression, bloquer l'utilisation tant que les profils ne sont pas installés, délai de 60 minutes.
- 📸 *Configuration de l'ESP*

### 4.5 Import du hash matériel
- À l'OOBE : `Shift+F10`, puis :
  ```powershell
  Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
  Install-Script -Name Get-WindowsAutopilotInfo -Force
  Get-WindowsAutopilotInfo -Online
  ```
- 📸 *Message de réussite de l'import*
- 📸 *Appareil dans la liste Autopilot, état du profil : Attribué*

### 4.6 Déploiement et OOBE
- Restauration du snapshot, connexion avec un utilisateur du groupe IT, ESP.
- 📸 *Écran de connexion professionnelle à l'OOBE*
- 📸 *Bureau final et nom de l'appareil `LAB-...`*
- 📸 *Comptes → Accès Professionnel ou scolaire*

### 4.7 Vérification dans Intune
- Appareil visible dans *Appareils → Tous les appareils*, propriété **Entreprise**, géré par Intune.
- Sur la VM : `dsregcmd /status` → `AzureAdJoined : YES` et `MdmUrl` renseigné.
- 📸 *Fiche de l'appareil dans Intune*

---

## 5. Étape 3 : Conformité et BitLocker

### 5.1 Stratégie de conformité `secuIT`
| Paramètre | Valeur |
|---|---|
| BitLocker | Exiger |
| Démarrage sécurisé | Exiger |
| TPM | Exiger |
| Pare-feu, antivirus, anti-espion | Exiger |
| Mot de passe | Longueur minimale : 8 caractères |
- Attribution : `GRP-IT-Techniciens`.
- 📸 *Configuration de la stratégie*

### 5.2 Stratégie de chiffrement `BitLocker-OS-Lab`
(Sécurité des points de terminaison → Chiffrement de disque)
- Exiger le chiffrement des appareils : **Activé**
- Avertissement pour un autre chiffrement : **Désactivé** (chiffrement silencieux)
- Chiffrement utilisateur standard : **Activé**
- Méthode : **XTS-AES 256** pour le lecteur système
- Authentification au démarrage : **TPM uniquement**, PIN et clé de démarrage non autorisés, **BitLocker sans TPM compatible : False**
- Sauvegarde de la clé de récupération dans Entra ID, et **pas de chiffrement tant qu'elle n'est pas stockée**
- 📸 *Récapitulatif de la stratégie*
- 📸 *État des paramètres : Réussite*
- 📸 *Clé de récupération dans la fiche de l'appareil* (**valeur masquée**)

### 5.3 Résultat
- 📸 *Rapport de conformité : 8 paramètres sur 8 conformes*

---

## 6. Étape 4 : Application Win32 (Notepad++)

### 6.1 Packaging
- Dossier source contenant **uniquement** le MSI.
- Génération :
  ```powershell
  .\IntuneWinAppUtil.exe -c C:\Source -s <fichier>.msi -o C:\Output
  ```
- 📸 *Fin d'exécution de la commande et fichier `.intunewin`*

### 6.2 Configuration dans Intune
| Champ | Valeur |
|---|---|
| Installation | `msiexec /i "<fichier>.msi" /qn /norestart` |
| Désinstallation | `msiexec /x {GUID} /qn /norestart` |
| Comportement | Système, aucun redémarrage |
| Exigences | Windows 10 1607+, 64 bits |
| Détection | Fichier `C:\Program Files\Notepad++\notepad++.exe` existe |
| Attribution | Obligatoire → `GRP-IT-Techniciens` |
- 📸 *Programme, règle de détection, attributions*

### 6.3 Résultat
- 📸 *État de l'installation : Installé*
- 📸 *Notepad++ dans le menu Démarrer de la VM*

### 6.4 Microsoft 365 Apps
- Application de type **Microsoft 365 Apps pour Windows 10 et versions ultérieures**.
- Composants : **Teams, Outlook, Word, Excel, PowerPoint**.
- Architecture **64 bits**, canal **Canal actuel**, langue **français**.
- Attribution : **Obligatoire** → `GRP-IT-Techniciens`.
- 📸 *Configuration de l'application*
- 📸 *État de l'installation : Installé*
- 📸 *Applications Office dans le menu Démarrer de la VM*

---

## 7. Étape 5 : Settings Catalog et Update Rings

### 7.1 Settings Catalog
- Profil `Config-Poste-IT`.
- Paramètre : **Removable Disks: Deny write access (User)** → Activé.
- Attribution : `GRP-IT-Techniciens`.
- Test : clé USB connectée à la VM (*VM → Périphériques amovibles*), écriture refusée.
- 📸 *Paramètre dans le profil*
- 📸 *Message de refus d'écriture sur la VM*

### 7.2 Update Rings
- `Update-Ring-IT` : report de **7 jours** pour les mises à jour de qualité, **30 jours** pour les mises à jour de fonctionnalités, heures actives de **8h à 18h**. Échéances : **2 jours** pour les mises à jour de qualité, **7 jours** pour les mises à jour de fonctionnalités.
- 📸 *Configuration de l'anneau*
- 📸 *Rapport : 1 appareil en Réussite*
- 📸 *Windows Update sur la VM : paramètres « gérés par votre organisation »*

---

## 8. Étape 6 : Accès conditionnel

### 8.1 Prérequis
- Désactivation des **paramètres de sécurité par défaut** (incompatibles avec l'accès conditionnel).
- Compte de secours exclu.
- 📸 *Paramètres de sécurité par défaut désactivés*

### 8.2 Stratégie « Exiger un appareil conforme »
| Élément | Valeur |
|---|---|
| Utilisateurs | `GRP-IT-Techniciens` (compte de secours exclu) |
| Ressources | Toutes les ressources, avec exclusion de **Microsoft Intune** et **Microsoft Intune Enrollment** |
| Octroi | Exiger que l'appareil soit marqué comme conforme |
| État | Rapport seul, puis **Activé** |
- 📸 *Détails de la stratégie*

### 8.3 Stratégie « Exiger la MFA pour les administrateurs »
- Stratégie **activée**, compte de secours exclu.
- 📸 *Liste des stratégies*

### 8.4 Tests
| Test | Appareil | Résultat attendu |
|---|---|---|
| 1 | PC personnel non géré | Accès **refusé** |
| 2 | VM conforme | Accès **autorisé** |
- 📸 *Écran de blocage sur le PC*
- 📸 *Accès réussi depuis la VM*
- 📸 *Journaux de connexion : « Exiger un appareil conforme » en échec (PC) et en réussite (VM)* (**adresses IP masquées**)

---

## 9. Problèmes rencontrés et solutions

| # | Symptôme | Cause | Solution |
|---|---|---|---|
| 1 | Test d'Autopilot sur une VM Azure (connexion RDP) | Pas d'OOBE accessible, Autopilot non pris en charge | VM locale VMware |
| 2 | OOBE sur « personnel ou travail » | Profil non attribué à l'appareil | Attendre l'état **Attribué**, puis restaurer le snapshot |
| 3 | Notification « Paramètres Autopilot introuvables » | Message transitoire | Synchroniser, attendre 10 à 15 minutes |
| 4 | Synchronisation refusée | Limite d'une synchronisation toutes les 10 minutes | Attendre |
| 5 | `MdmUrl` vide, appareil absent d'Intune | **Portée MDM sur Aucun** | Portée sur **Tout**, puis refaire l'OOBE |
| 6 | Inventaire à `0.0.0.0` | Premier check-in pas encore remonté | Synchroniser, patienter |
| 7 | Impossible d'ajouter un membre à un groupe | Groupe **dynamique** | Renseigner Department et JobTitle |
| 8 | Erreurs 404 BitLocker / Démarrage sécurisé, échec TPM | Secure Boot désactivé dans VMware, attestation en retard | Activer Secure Boot, redémarrer, resynchroniser |
| 9 | `tpm.msc` en erreur `0x80090029` | Console non fiable sur cette VM | Vérifier avec `Get-Tpm` |
| 10 | Disque chiffré en XTS-AES 128, protection désactivée, aucun protecteur | Chiffrement automatique de Windows avant l'arrivée de la stratégie (probable) | Stratégie `BitLocker-OS-Lab` : elle a suffi, BitLocker est passé en conforme |
| 11 | `IntuneWinAppUtil.exe` introuvable | Exécution sans `.\` | `.\IntuneWinAppUtil.exe` |
| 12 | Activation de l'accès conditionnel refusée | Paramètres de sécurité par défaut actifs | Les désactiver |
| 13 | Bloc « Insights from Copilot » en échec | Service d'aperçu sans lien avec la config | Ignorer |

---

## 10. Limites et pistes d'amélioration

- Essai E5 d'**un mois** : le tenant et ses captures sont perdus à l'expiration.
- Une seule VM de test, utilisateur de test avec droits d'administrateur général (`admintest`) : à remplacer par un utilisateur standard pour un test plus réaliste.
- Pistes : Defender for Endpoint, scripts de remédiation, packaging d'une application plus complexe, Autopilot en mode auto-déploiement.

---

## 11. Lien avec mes autres labs

- Lab AD / GPO → équivalent on-prem de la configuration (Settings Catalog)
- Lab SCCM / MECM → équivalent on-prem du déploiement d'applications
- Lab packaging Intune → complété par la partie Win32 de ce lab
- Dépôts associés :
  - [lab-azure](https://github.com/adjatoto72-cmyk/lab-azure) : création d'utilisateurs et attribution de licences avec PowerShell
  - [Mise-en-place-GPO-avec-powershell](https://github.com/adjatoto72-cmyk/Mise-en-place-GPO-avec-powershell)
  - [SCCM-LAB](https://github.com/adjatoto72-cmyk/SCCM-LAB)
  - [lab-packaging-applications-intune](https://github.com/adjatoto72-cmyk/lab-packaging-applications-intune)
  - [Packaging-et-d-ploiement-d-applications-intune](https://github.com/adjatoto72-cmyk/Packaging-et-d-ploiement-d-applications-intune)
  - [packaging-des-applications](https://github.com/adjatoto72-cmyk/packaging-des-applications)

---

## 12. Sécurité de la publication

**À masquer avant de publier**
- Adresses IP publiques et IPv6
- Identifiant du tenant, identifiants d'appareils et d'utilisateurs (GUID)
- **Clés de récupération BitLocker**
- Mots de passe, compte de secours, codes d'authentification d'appareil (PowerShell)
- Nom du tenant `.onmicrosoft.com` si tu ne veux pas l'exposer

**À vérifier**
- Aucune capture ne montre de jeton ou de secret.
- Les scripts du dépôt utilisent des variables, pas de valeurs réelles.

---

## 13. Structure conseillée du dépôt

```
/
├── README.md
├── 01-identites/
├── 02-autopilot/
├── 03-conformite-bitlocker/
├── 04-application-win32/
├── 05-settings-catalog-update-rings/
├── 06-acces-conditionnel/
├── scripts/
│   ├── create-users.ps1
│   ├── set-attributes-licenses.ps1
│   └── detect-notepadpp.ps1
└── images/
```
