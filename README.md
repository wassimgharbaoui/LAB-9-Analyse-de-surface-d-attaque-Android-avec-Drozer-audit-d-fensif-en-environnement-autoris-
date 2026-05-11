# Analyse de Sécurité – Application Android "KeyStorage"

## Fiche d'identité

| Champ | Valeur |
|-------|--------|
| **Application** | KeyStorage |
| **Paquet** | com.keystorage.securebox |
| **Version** | 2.1.0 |
| **Date de l’analyse** | 2025-11-03 |
| **Instrument principal** | Drozer 3.2.1 |
| **Environnement de test** | Émulateur Android arm64 – API 31 (Android 12) |

---

## Résumé des failles détectées

L’application KeyStorage présente plusieurs configurations dangereuses au niveau de ses composants Android. Sept vulnérabilités ont été recensées, dont deux de niveau « critique » permettant l’extraction des mots de passe utilisateur sans aucune authentification préalable.

---

## Méthodologie appliquée

- Vérification de la connectivité ADB et de l’état de l’émulateur
- Installation de l’agent Drozer et de l’APK cible
- Énumération des composants exportés (activities, services, receivers, providers)
- Inspection des permissions et des filtres d’intent
- Détection des URIs non protégées
- Proposition de correctifs techniques

---

## Étape 1 – Préparation du poste de travail

<img width="327" height="641" alt="Screenshot 2026-05-11 190722" src="https://github.com/user-attachments/assets/db138d17-350e-49df-a39f-e466482d87d6" />


    adb install drozer-agent-3.2.1.apk   → OK
    adb install keystorage.apk           → OK
    adb forward tcp:31415 tcp:31415      → Tunnel actif

---

## Étape 2 – Connexion à la console Drozer

<img width="805" height="402" alt="Screenshot 2026-05-11 190729" src="https://github.com/user-attachments/assets/3d5a2b67-39bc-40ba-9877-6ff56e567482" />


    drozer console connect
    Appareil sélectionné : emu-5556 (Android 12 - API 31)
    Console Drozer (v3.2.1)
    dz>

---

## Étape 3 – Inventaire des composants exposés

<img width="816" height="331" alt="Screenshot 2026-05-11 190735" src="https://github.com/user-attachments/assets/70f69936-75ee-4816-aaa3-05adbeaeb55c" />


### Activités exportées

    dz> run app.activity.info -a com.keystorage.securebox

<img width="635" height="252" alt="Screenshot 2026-05-11 190739" src="https://github.com/user-attachments/assets/b12d5173-c817-4e78-ae4a-84b2462f96ba" />


| Activité | Protection |
|----------|------------|
| MainLogin | ❌ aucune |
| FileSelector | ❌ aucune |
| SecretList | ❌ aucune |

### Services accessibles

    dz> run app.service.info -a com.keystorage.securebox

<img width="605" height="148" alt="Screenshot 2026-05-11 190744" src="https://github.com/user-attachments/assets/0a49d9b5-71c6-48d2-aa21-a0303d04694d" />


| Service | Permission requise |
|---------|--------------------|
| GateService | ❌ aucune |
| CodecService | ❌ aucune |

### Récepteurs broadcast

    dz> run app.broadcast.info -a com.keystorage.securebox

<img width="691" height="115" alt="Screenshot 2026-05-11 190747" src="https://github.com/user-attachments/assets/5ee971ff-c75c-4f0c-b051-1c69bb6eeeb2" />


| Récepteur | Niveau de permission |
|-----------|----------------------|
| UpdateReceiver | ✅ android.permission.DUMP |

### Fournisseurs de contenu

    dz> run app.provider.info -a com.keystorage.securebox

<img width="865" height="543" alt="Screenshot 2026-05-11 190753" src="https://github.com/user-attachments/assets/07b2074c-1b8e-45ed-b703-c506132ede15" />


| Fournisseur | Lecture | Écriture |
|-------------|---------|----------|
| InternalProvider | ❌ libre (sauf /keys) | ❌ libre (sauf /keys) |
| BackupAgentProvider | ❌ libre | ❌ libre |

---

## Étape 4 – Vérification des garde-fous

### Extraction du manifeste

    dz> run app.package.manifest com.keystorage.securebox
<img width="887" height="781" alt="Screenshot 2026-05-11 190808" src="https://github.com/user-attachments/assets/6794b2f6-c257-4b96-a773-41e15e120b44" />


**Anomalies relevées :**
- `debuggable="true"` → non conforme pour une release
- `allowBackup="true"` → risque d’exfiltration des données
- `protectionLevel="dangerous"` sur READ_KEYS/WRITE_KEYS → insuffisant

### URIs exposées

    dz> run scanner.provider.finduris -a com.keystorage.securebox

<img width="881" height="322" alt="Screenshot 2026-05-11 190813" src="https://github.com/user-attachments/assets/674349d4-e1c5-46a3-a8da-e326dc0d3760" />


**Chemins accessibles sans permission :**

    content://com.keystorage.securebox.provider.InternalProvider/Secrets
    content://com.keystorage.securebox.provider.InternalProvider/Secrets/
    content://com.keystorage.securebox.provider.InternalProvider/keys/

---

## Étape 5 – Classification des risques

| ID | Composant | Vulnérabilité | Sévérité | Impact |
|----|-----------|---------------|----------|--------|
| F1 | SecretList | Activité exportée sans restriction | 🔴 Critique | Accès direct à la liste des secrets |
| F2 | InternalProvider/Secrets | URI accessible sans permission | 🔴 Critique | Fuite de tous les mots de passe |
| F3 | FileSelector | Activité exportée inutilement | 🔴 Élevé | Lecture de fichiers arbitraires |
| F4 | GateService | Service exporté sans permission | 🔴 Élevé | Contournement de l’authentification |
| F5 | CodecService | Service exporté sans permission | ⚠️ Moyen | Opérations crypto non autorisées |
| F6 | Application | debuggable=true | ⚠️ Moyen | Exposition en environnement de production |
| F7 | Application | allowBackup=true | ⚠️ Moyen | Extraction des données via sauvegarde |

---

## Correspondance avec OWASP MASVS

| ID | Faiblesse | Référence MASVS | Justification |
|----|-----------|----------------|----------------|
| F1 | SecretList exportée sans protection | MSTG-PLATFORM-1 | Limiter les composants exposés au strict nécessaire |
| F2 | InternalProvider mal sécurisé | MSTG-STORAGE-2 | Les données sensibles exigent des contrôles d'accès |
| F4 | GateService sans validation | MSTG-PLATFORM-2 | Valider toute entrée externe |
| F3 | Récepteurs sans vérification | MSTG-PLATFORM-3 | Vérifier les intents entrants |
| F5 | Permissions trop permissives | MSTG-AUTH-1 | Renforcer les mécanismes d'authentification |

---

## Recommandations correctives

### 1. Activités SecretList et FileSelector

Code initial :"activity android:name=".SecretList" android:exported="true" /"

Code corrigé :
    "activity android:name=".SecretList" android:exported="false" /"

### 2. Fournisseur InternalProvider

Code initial :
    "provider android:name=".InternalProvider" android:exported="true" "

Code corrigé :
    provider
        android:name=".InternalProvider"
        android:exported="true"
        android:readPermission="com.keystorage.securebox.READ_KEYS"
        android:writePermission="com.keystorage.securebox.WRITE_KEYS" "

### 3. Services GateService et CodecService

Code initial :
    "service android:name=".GateService" android:exported="true" /"

Code corrigé :
    "service
        android:name=".GateService"
        android:exported="false" "

### 4. Balise application

Code initial :
    "application android:debuggable="true" android:allowBackup="true" /"

Code corrigé :
    application android:debuggable="false" android:allowBackup="false" "

### 5. Permissions – passage au niveau signature

    <permission
        android:name="com.keystorage.securebox.READ_KEYS"
        android:protectionLevel="signature" />

---

## Organisation des éléments de preuve

    audit/
    ├── rapport_final.md
    ├── triage.csv
    ├── checklist_fin.md
    ├── activities/
    │   └── exported_activities.txt
    ├── services/
    │   └── exported_services.txt
    ├── receivers/
    │   └── exported_receivers.txt
    └── providers/
        └── exported_providers.txt

---

## ✅ Points de contrôle – fin d'audit

- [x] Toutes les étapes du laboratoire ont été respectées
- [x] L'ensemble des composants Android a été examiné
- [x] La matrice de criticité est complète
- [x] Chaque correction proposée est précise et opérationnelle
- [x] La correspondance avec OWASP MASVS est exacte
- [x] Aucune donnée personnelle réelle n'apparaît dans ce document
- [x] La structure du rapport est claire et homogène


