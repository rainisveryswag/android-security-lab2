# Rapport d'Expertise : Analyse de l'Intégrité Système et Audit Privilégié (Lab 2)

**Analyste :** Yousra Zarri  
**Date :** 01/05/2026  
**Établissement :** ENSA Marrakech  
**Environnement :** Android Studio (AVD) — API 33

---

## 1. Contexte et Périmètre

L'objectif de ce lab est d'observer concrètement l'impact d'un accès superutilisateur sur les mécanismes de protection d'Android. En simulant un attaquant disposant de privilèges root, on évalue dans quelle mesure le sandboxing applicatif et la chaîne de confiance au démarrage peuvent être contournés.

### Fiche de Périmètre

| Champ | Détail |
|---|---|
| **Cible** | `FireStorm.apk` v1.0 (APK de test) |
| **Environnement** | AVD isolé — aucun compte Google, réseau cloisonné |
| **Vecteurs testés** | ADB (Android Debug Bridge), shell root, inspection système |
| **Données** | 100% fictives — aucune donnée réelle impliquée |
| **Clôture** | Réinitialisation complète de l'AVD après session |

---

## 2. Élévation de Privilèges

### Procédure appliquée

L'émulateur Android autorise, sur les images de type *Google APIs*, le basculement du démon ADB en mode root. La séquence suivante permet d'obtenir un accès superutilisateur et de remonter la partition système en lecture/écriture :

```bash
# Démarrage de l'AVD avec partition système modifiable
emulator -avd FireStorm_Lab_API33 -writable-system

# Activation du mode root sur le démon ADB
adb root

# Remontage de /system en lecture/écriture
adb remount
```

### Validation de l'état root

```bash
adb shell id
# Attendu : uid=0(root) gid=0(root)

adb shell getprop ro.boot.verifiedbootstate
# Attendu : orange (verity désactivé, signature non verrouillée)

adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```

Un retour `uid=0(root)` et un état `verifiedbootstate` à `orange` confirment que les protections système ne sont plus actives — l'environnement est prêt pour l'audit.

---

## 3. Chaîne de Confiance — Verified Boot & AVB

Android garantit l'intégrité du système via une validation en cascade au démarrage : chaque composant vérifie le suivant avant de lui céder le contrôle.

```
ROM Boot → Bootloader → Kernel → System → Application
   └── rupture à n'importe quel niveau compromet toute la chaîne
```

| État | Interprétation |
|---|---|
| 🟢 `GREEN` | Système intact, conforme à l'image constructeur |
| 🟡 `YELLOW / ORANGE` | Image modifiée — rooting ou ROM custom détecté |
| 🔴 `RED` | Intégrité sévèrement compromise |

La désactivation de `dm-verity` via `adb disable-verity` neutralise les contrôles d'intégrité sur les partitions. Cela ouvre la voie à l'injection de binaires arbitraires (ex. `su`) et à des modifications persistantes que le système d'exploitation ne peut plus détecter.

**AVB 2.0** renforce ce mécanisme avec une protection anti-rollback (blocage des downgrades vers des versions vulnérables) et des métadonnées cryptographiques par partition consolidées dans la structure `vbmeta`.

---

## 4. Scénarios Applicatifs de Référence

Trois scénarios fonctionnels ont été définis avant tout test intrusif, afin d'établir un comportement de référence reproductible pour `FireStorm.apk` :

| ID | Scénario | Action | Résultat Attendu |
|---|---|---|---|
| S1 | Démarrage | Lancement de FireStorm | Écran principal chargé, aucune erreur |
| S2 | Navigation | Parcours des sections de l'app | Transitions fluides, données fictives affichées |
| S3 | Interaction | Déclenchement d'une action métier | Réponse correcte de l'application |

---

## 5. Audit des Données Applicatives (OWASP)

L'accès root donne accès au répertoire privé de chaque application, normalement protégé par le sandbox Android. Cela permet d'appliquer directement les méthodologies du **MASTG**.

### MASVS STORAGE-1 — Inspection du stockage local

```bash
adb shell ls -la /data/data/com.lab.firestorm/
adb shell cat /data/data/com.lab.firestorm/shared_prefs/*.xml
```

On vérifie qu'aucune donnée sensible (token, clé, identifiant) n'est stockée en clair dans les fichiers de préférences partagées. Une application conforme à STORAGE-1 doit chiffrer ces valeurs via l'**Android Keystore**.

### MASVS NETWORK-1 — Détection de fuites via Logcat

```bash
adb logcat -d | grep -iE "token|auth|password|key|secret" > firestorm_leaks.txt
```

L'analyse des journaux pendant l'exécution de FireStorm permet d'identifier d'éventuelles fuites d'informations sensibles dans les logs système — une erreur courante dans les builds de debug.

---

## 6. Matrice des Risques

| # | Risque | Impact | Mesure Appliquée |
|---|---|---|---|
| 1 | Sandbox contourné | Accès direct aux fichiers privés de FireStorm | Chiffrement Android Keystore |
| 2 | dm-verity désactivé | Modifications système indétectables | AVD jetable, jamais réutilisé |
| 3 | Données persistantes post-audit | Fuite d'informations résiduelles | Wipe Data systématique en fin de session |
| 4 | Instabilité de l'émulateur | Tests non reproductibles | Snapshots AVD avant chaque test |
| 5 | Réseau non cloisonné | Communications non maîtrisées | Mode Host-only, pas d'accès Internet |
| 6 | APK non vérifiée | Introduction de comportements inattendus | Seule FireStorm.apk installée |
| 7 | Absence de traçabilité | Audit non reproductible | Logcat archivé + captures horodatées |
| 8 | Compte personnel lié | Fuite de données personnelles | AVD vierge, aucun compte configuré |

---

## 7. Clôture et Restauration de l'Environnement

### Fiche environnement

| Champ | Valeur |
|---|---|
| Date | 01/05/2026 |
| Analyste | Yousra Zarri |
| Support | AVD Google APIs — API 33 |
| Application | `FireStorm.apk` v1.0 |
| Réseau | Isolé — Host-only |
| Reset effectué | ✅ Oui |

### Procédure de réinitialisation

```bash
# Android Studio → Device Manager → Wipe Data
# ou via ligne de commande :
adb emu avd wipe-data
```

L'AVD est considéré comme réinitialisé lorsque l'assistant de configuration Android s'affiche au redémarrage.

### Checklist de clôture

| Tâche | Statut |
|---|---|
| Périmètre documenté avant les tests | ✅ |
| AVD propre, sans données résiduelles | ✅ |
| FireStorm.apk installée uniquement | ✅ |
| 3 scénarios exécutés et documentés | ✅ |
| Données 100% fictives | ✅ |
| Logcat archivé | ✅ |
| Wipe Data effectué | ✅ |
| Aucun compte personnel configuré | ✅ |

---

> Ce rapport a été produit dans un cadre strictement pédagogique.  
> Toutes les manipulations ont été réalisées sur un AVD dédié. Aucune action n'a été menée sur un appareil de production ou un environnement réel.

<div align="center">
  <sub>ENSA Marrakech · GCDSTE · Sécurité des Applications Mobiles</sub>
</div>
