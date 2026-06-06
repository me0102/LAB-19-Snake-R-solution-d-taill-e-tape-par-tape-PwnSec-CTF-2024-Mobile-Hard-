<div align="center">

# LAB 19 

[![CTF](https://img.shields.io/badge/CTF-PwnSec%202024-red?style=for-the-badge&logo=hackthebox&logoColor=white)](https://pwnsec.com)
[![Catégorie](https://img.shields.io/badge/Catégorie-Mobile-blue?style=for-the-badge&logo=android&logoColor=white)]()
[![Difficulté](https://img.shields.io/badge/Difficulté-Hard-critical?style=for-the-badge)]()
[![Statut](https://img.shields.io/badge/Statut-Résolu%20✓-success?style=for-the-badge)]()

**Bypass anti-root Android + exploitation de désérialisation SnakeYAML**

</div>

---

## 📖 Vue d'ensemble

Ce writeup couvre la résolution complète du challenge **Snake** du PwnSec CTF 2024. L'épreuve fournit un fichier `snake.apk` — une application Android qui refuse de s'exécuter sur tout environnement rooté ou émulé. La solution se déroule en deux actes :

1. **Phase 1 — Bypass** : contourner le mécanisme anti-root en modifiant directement le bytecode Dalvik
2. **Phase 2 — Exploitation** : abuser d'une désérialisation non sécurisée via SnakeYAML pour exécuter du code arbitraire

> [!NOTE]
> **Easter egg thématique** : Le challenge joue sur plusieurs couches de références croisées. « Snake » renvoie à la fois à *Solid Snake* (*Metal Gear Solid*), à la bibliothèque Java **SnakeYAML**, et à la classe interne `BigBoss` (antagoniste principal de la saga). Le flag lui-même cite une réplique culte du jeu : *"We're not tools of the government, or anyone else."*

---

## 🎯 Objectif

Récupérer le flag au format :

```
PWNSEC{...}
```

---

## 🛠️ Prérequis

| Outil | Version | Utilisation |
|---|---|---|
| `adb` | any | Déploiement APK et transfert de fichiers |
| `jadx` | any | Décompilation vers Java lisible |
| `apktool` | 3.0+ | Désassemblage / réassemblage smali |
| `apksigner` | — | Re-signature de l'APK modifié |
| `Java JDK` | ≥ 21 | Prérequis d'apksigner |
| Éditeur de texte | any | Édition des fichiers `.smali` |
| `PowerShell` | any | Génération du payload YAML |

---

## 🔗 Chaîne d'exploitation

```
snake.apk
    │
    ├─[1]─ Installation → CRASH (root détecté)
    │
    ├─[2]─ JADX → lecture du code Java → isDeviceRooted() identifié
    │
    ├─[3]─ apktool → désassemblage smali
    │
    ├─[4]─ Patch : isDeviceRooted() → retour forcé à false
    │
    ├─[5]─ apktool → recompilation → snake_patched.apk
    │
    ├─[6]─ apksigner → signature → snake_signed.apk
    │
    ├─[7]─ adb install → l'app tourne normalement ✓
    │
    ├─[8]─ SnakeYAML `!!` → instanciation de BigBoss via réflexion
    │
    └─[9]─ adb push Skull_Face.yml → FLAG 🎉
```

---

## 📋 Résolution pas à pas

### Étape 1 — Installation initiale

On déploie l'APK sur un émulateur rooté :

```powershell
adb install snake.apk
```

```
Performing Streamed Install
Success
```

L'installation passe sans erreur. Mais au premier lancement, l'application s'arrête immédiatement : une protection anti-root est en place.

---

### Étape 2 — Analyse statique (JADX)

La décompilation avec **jadx** expose la logique d'initialisation dans `MainActivity`.

#### `onCreate()` — Point d'entrée de l'activité

```java
@Override
public final void onCreate(Bundle bundle) {
    super.onCreate(bundle);
    setContentView(R.layout.activity_main);

    if (isDeviceRooted(getApplicationContext())) {
        Log.w("Rooted", "Root detected! Exiting application.");
        finish();
        System.exit(0);  // ← verrou #1 à neutraliser
    }
    // ...
}
```

> [!WARNING]
> `isDeviceRooted()` est appelée au démarrage. En cas de détection, `finish()` + `System.exit(0)` terminent brutalement le processus. C'est la première protection à contourner.

#### Indice sur les permissions

```java
String[] strArr = {"android.permission.READ_EXTERNAL_STORAGE"};
```

L'app réclame un accès au stockage externe — signal fort qu'elle consomme un fichier depuis `/sdcard/` au démarrage (probablement un fichier YAML).

---

### Étape 3 — Désassemblage smali (Apktool)

On extrait le bytecode Dalvik en fichiers `.smali` éditables :

```powershell
apktool d snake.apk -o snake_smali
```

<details>
<summary>📄 Sortie complète</summary>

```
I: Using Apktool 3.0.2 on snake.apk with 4 threads
I: Loading resource table...
I: Baksmaling classes.dex...
I: Decoding value resources...
I: Decoding file resources...
I: Generating values XMLs...
I: Decoding AndroidManifest.xml with resources...
I: Copying original files...
```

</details>

Navigation vers le package cible :

```powershell
cd snake_smali/smali/com/pwnsec/snake/
```

---

### Étape 4 — Patch smali : neutralisation du root check

On ouvre `MainActivity.smali` et on repère `isDeviceRooted`.

#### ❌ Version originale (détection active)

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

    # Vérifications : binaire su, tags de build, chemins système...

    const/4 v0, 0x1    # → true : root détecté, app tuée
    return v0
.end method
```

#### ✅ Version patchée (bypass complet)

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

    const/4 p0, 0x0    # → false : l'appareil n'est jamais considéré rooté

    return p0
.end method
```

> [!TIP]
> Le principe est radical : on vide entièrement la méthode et on la remplace par un retour hardcodé à `false` (`0x0`). Quelle que soit la réalité du device, le garde-fou est désactivé de façon permanente.

---

### Étape 5 — Recompilation

```powershell
apktool b snake_smali -o snake_patched.apk
```

```
I: Using Apktool 3.0.2
I: Smaling smali folder into classes.dex...
I: Building resources with aapt2...
I: Building apk file...
I: Built apk into: snake_patched.apk  ✓
```

---

### Étape 6 — Signature de l'APK

Android rejette tout APK non signé. On utilise le **debug keystore** standard :

```powershell
# Définir JAVA_HOME
$env:JAVA_HOME = "C:\Program Files\Microsoft\jdk-21.0.8.9-hotspot"

# Signer
& "C:\...\Android\Sdk\build-tools\36.1.0\apksigner.bat" sign `
    --ks debug.keystore `
    --ks-pass pass:android `
    --key-pass pass:android `
    --out snake_signed.apk `
    snake_patched.apk
```

Vérification :

```powershell
& "C:\...\apksigner.bat" verify snake_signed.apk
echo $?
# → True ✓
```

---

### Étape 7 — Déploiement du binaire patché

```powershell
adb install snake_signed.apk
```

```
Performing Streamed Install
Success
```

✅ L'application démarre normalement sur l'émulateur rooté. Bypass anti-root confirmé.

---

### Étape 8 — Exploitation SnakeYAML

L'analyse du code révèle que l'app utilise **SnakeYAML** pour parser un fichier YAML depuis `/sdcard/`. Cette bibliothèque supporte la syntaxe `!!` (type tag explicite), permettant de forcer l'instanciation d'une classe Java arbitraire par réflexion.

**Sans validation d'entrée → surface d'attaque par désérialisation non contrôlée.**

#### Classe cible identifiée

```
com.pwnsec.snake.BigBoss
```

#### Construction du payload

```powershell
$payload = '!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaake"]'

[System.IO.File]::WriteAllText(
    "$env:USERPROFILE\Desktop\snake\snake\Skull_Face.yml",
    $payload,
    (New-Object System.Text.UTF8Encoding $false)  # UTF-8 sans BOM
)
```

Contenu généré :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaake"]
```

#### Anatomie du payload

| Composant | Rôle |
|---|---|
| `!!com.pwnsec.snake.BigBoss` | Ordonne à SnakeYAML d'instancier la classe via réflexion Java |
| `["Snaaaaaaaaaaaaake"]` | Argument transmis au constructeur |
| **Résultat** | Le constructeur de `BigBoss` s'exécute et retourne/affiche le flag |

> [!IMPORTANT]
> **Pourquoi ça marche ?** Quand SnakeYAML rencontre un tag `!!`, il résout la classe indiquée par réflexion et appelle son constructeur avec les arguments fournis. Si ce constructeur déclenche un effet de bord observable (affichage du flag), l'attaquant peut le provoquer sans accès au code source complet — c'est la définition d'une gadget chain de désérialisation.

#### Injection sur l'appareil

```powershell
adb push Skull_Face.yml /sdcard/Skull_Face.yml
```

L'app consomme ce fichier au démarrage ou via son interface, déclenchant le payload.

---

### Étape 9 — Flag 🏁

```
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

> *"We're not tools of the government, or anyone else. Fighting was the only thing… the only thing I was good at. But at least I got to fight for what I believed in."* — Solid Snake, Metal Gear Solid

---

## 🧠 Notions techniques couvertes

| Domaine | Technique |
|---|---|
| Reverse engineering Android | Lecture du bytecode Dalvik (smali) + Java reconstitué (jadx) |
| Contournement anti-root | Modification directe d'une méthode `.smali` pour neutraliser la détection |
| Repackaging APK | Reconstruction + re-signature d'un binaire Android altéré |
| Désérialisation SnakeYAML | Abus du type tag `!!` pour instancier des classes arbitraires |
| ADB | Déploiement, transfert de fichiers, interaction avec l'émulateur |

---

## 📚 Références

- [SnakeYAML Deserialization — HackTricks](https://book.hacktricks.xyz/pentesting-web/deserialization#snakeyaml)
- [Apktool — Documentation officielle](https://apktool.org/)
- [apksigner — Android Developers](https://developer.android.com/tools/apksigner)
- [Smali / Baksmali — JesusFreke](https://github.com/JesusFreke/smali)

---

<div align="center">

*Writeup rédigé dans le cadre du PwnSec CTF 2024*

</div>
