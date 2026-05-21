#  LAB 17 — Cracker OWASP UnCrackable Android Level 3

> **Cours : Sécurité des applications mobiles**  
> Outils utilisés : `apktool` · `jadx-gui` · `Ghidra` · `adb` · `Python`

---

##  Objectifs d'apprentissage

<img width="333" height="738" alt="pic1" src="https://github.com/user-attachments/assets/3c32115b-fbde-492b-915c-196379bbd9b7" />


À la fin de ce lab, tu sauras :

-  Décompiler et patcher une APK (smali + Java)
-  Analyser une librairie native `.so` avec un outil gratuit
-  Contourner l'anti-debug, anti-Frida, anti-root et la vérification d'intégrité
-  Déboguer en live avec `gdb` et modifier des registres
-  Comprendre un XOR byte par byte et calculer le mot de passe secret
- Tout ça avec **uniquement des outils gratuits** (Android Studio + Ghidra + apktool)

---

## Prérequis

- Android Studio (avec un émulateur configuré)
- [apktool](https://apktool.org/) ≥ 3.0
- [jadx-gui](https://github.com/skylot/jadx)
- [Ghidra](https://ghidra-sre.org/)
- Java JDK (pour `keytool` et `apksigner`)
- Python 3
- ADB (inclus dans Android Studio)

---

## Étapes du lab

| # | Étape | Description |
|---|-------|-------------|
| 1 | Objectifs | Comprendre les objectifs du codelab |
| 2 | Prérequis | Installer les outils nécessaires |
| 3 | Analyse statique | Analyser l'APK avec jadx-GUI (Java) |
| 4 | Décompilation | Décompiler l'APK avec apktool |
| 5 | Patch smali | Supprimer le message « tampered » / root |
| 6 | Patch natif | Patcher `libfoo.so` avec Ghidra (anti-debug + anti-Frida) |
| 7 | Analyse native | Analyser la logique de vérification dans `libuncrackable3.so` |

---

## Étape 1 — Analyse statique avec jadx-GUI

Ouvre l'APK dans **jadx-gui** et inspecte `MainActivity.java`.

### Vue d'ensemble de `MainActivity` — méthode `verify()`

<img width="1957" height="959" alt="pic2" src="https://github.com/user-attachments/assets/d411e4c6-8fc7-4ef5-8324-2fe86588a1a7" />


**Ce qu'on observe dans `onCreate` :**

```java
// Vérification d'intégrité CRC sur libfoo.so et classes.dex
verifyLibs();

// Lancement d'un thread anti-debug
new AsyncTask<Void, String, String>() {
    public String doInBackground(Void... voidArr) {
        while (!Debug.isDebuggerConnected()) {
            SystemClock.sleep(100L);
        }
        return null;
    }
    public void onPostExecute(String str) {
        MainActivity.this.showDialog("Debugger detected!");
        System.exit(0);
    }
}.execute(null, null, null);

// Vérification root + intégrité
if (RootDetection.checkRoot1() || RootDetection.checkRoot2() || RootDetection.checkRoot3()
    || IntegrityCheck.isDebuggable(getApplicationContext())) {
    showDialog("Rooting or tampering detected.");
}
```

### Méthode `verifyLibs()` — vérification CRC

<img width="1972" height="955" alt="pic3" src="https://github.com/user-attachments/assets/6f5ea1fb-9d92-46ac-b018-a69de6fbb34c" />


```java
this.crc.put("armeabi-v7a", ...);
this.crc.put("arm64-v8a", ...);
// Si le CRC de libfoo.so ou classes.dex ne correspond pas → tampered = 31337
```

### Méthode `onCreate()` complète

<img width="1919" height="954" alt="pic4" src="https://github.com/user-attachments/assets/00773e68-19bf-4788-8b5c-4749bc189519" />

>  La vérification réelle est déléguée à `CodeCheck.check_code()` qui appelle du code natif via `libfoo.so`.

---

## 🛠️ Étape 2 — Décompiler l'APK avec apktool

```powershell
apktool d UnCrackable-Level3.apk -o uncrackable3
```

<img width="2172" height="724" alt="pic5" src="https://github.com/user-attachments/assets/99d91b79-cf94-49f4-9d1f-4a4da798ad9c" />

>  Le dossier `uncrackable3/` contient maintenant les fichiers smali, les ressources XML et les librairies `.so`.

---

## 🔨 Étape 3 — Recompiler l'APK patché

Après avoir modifié les fichiers smali pour neutraliser les détections :

```powershell
cd uncrackable3
cd ..
apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
```

<img width="2172" height="724" alt="pic6" src="https://github.com/user-attachments/assets/7fb2cc1d-7a43-47ad-81e8-cb9b42da0a9d" />

---

## Étape 4 — Signer l'APK

### Générer un keystore

```powershell
keytool -genkey -v -keystore my-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-alias
```

<img width="1341" height="108" alt="pic7" src="https://github.com/user-attachments/assets/2da6cdd6-3da0-458b-a046-e2d998770de8" />


### Signer avec apksigner

```powershell
apksigner sign --ks my-release-key.jks --out UnCrackable-Level3-final.apk UnCrackable-Level3-aligned.apk
```

![Uploading pic8.png…]()


### Installer via ADB

```powershell
adb install UnCrackable-Level3-final.apk
```
<img width="646" height="106" alt="pic9" src="https://github.com/user-attachments/assets/21f96743-a53f-4f3c-81da-3871782aba4d" />

---

## Étape 5 — Comportement avant patch

Sur émulateur sans patch, l'application détecte le root et refuse de démarrer :

<img width="1334" height="623" alt="pic10" src="https://github.com/user-attachments/assets/62a1b757-536a-4022-bfbb-545353ac4486" />


---

##  Étape 6 — Analyser `libfoo.so` avec Ghidra

Ouvre `libfoo.so` dans **Ghidra** et analyse la fonction `FUN_00013080`.

<img width="1036" height="254" alt="pic11" src="https://github.com/user-attachments/assets/b8a436b3-1d3d-4fa8-9a8c-3f38047f0ee8" />


**Ce que la fonction fait :**

```c
// Anti-Frida : boucle sur /proc/self/maps à la recherche de "frida" et "xposed"
__stream = fopen("/proc/self/maps", "r");
while (fgets(local_214, 0x200, __stream)) {
    pcVar1 = strstr(local_214, "frida");
    if (pcVar1 != (char *)0x0) break;
    pcVar1 = strstr(local_214, "xposed");
} while (pcVar1 == (char *)0x0);

// Si détecté → appelle goodbye() et termine le thread
goodbye();
pthread_create(&pStack_354, NULL, FUN_00013080, NULL);
```

>  Pour bypasser : patcher les octets correspondants aux appels `strstr` ou `pthread_create` dans Ghidra (remplacer par des `NOP` ou forcer le retour).

---

##  Étape 7 — Décoder la clé secrète (XOR)

Après analyse de la logique native dans Ghidra, on identifie :
- Une **clé encodée en XOR** stockée en dur
- Un **masque itératif** (`pizzapizzapizzapizzapizzapizza`)

**Script Python de décodage :**

<img width="1568" height="81" alt="pic12" src="https://github.com/user-attachments/assets/34625abd-2e76-4d36-a23f-8471d599de5d" />


```python
# === DÉCODAGE INVERSE DE LA CLÉ ENCODÉE (MODE XOR) ===

# Constante encodée trouvée dans le code C natif
encoded = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")

# Le masque itératif (24 octets)
xor_key = b"pizzapizzapizzapizzapizzapizza"

# XOR byte par byte pour révéler la string claire
secret = bytes(a ^ b for a, b in zip(encoded, xor_key))
print("Clé secrète trouvée :", secret.decode())
```

**Exécution :**

```
python decode_key.py
Clé secrète trouvée : making owasp great again
```

---

##  Résultat final

Entre la clé dans l'application :

```
making owasp great again
```

<img width="679" height="750" alt="pic13" src="https://github.com/user-attachments/assets/0e5512b2-682b-4f73-bc7d-ec4ac104746f" />


**→ Success! This is the correct secret. **

---

## Structure du projet

```
Lab17/
├── README.md
├── screenshots/                    # ← Dossier à créer avec pic1.png à pic13.png
│   ├── pic1.png                    # Objectifs du lab
│   ├── pic2.png                    # jadx — méthode verify()
│   ├── pic3.png                    # jadx — décompilation apktool
│   ├── pic4.png                    # jadx — verifyLibs() + onCreate()
│   ├── pic5.png                    # apktool d (décompilation)
│   ├── pic6.png                    # apktool b (recompilation)
│   ├── pic7.png                    # keytool génération keystore
│   ├── pic8.png                    # apksigner
│   ├── pic9.png                    # adb install
│   ├── pic10.png                   # Rooting detected (avant patch)
│   ├── pic11.png                   # Ghidra — analyse libfoo.so
│   ├── pic12.png                   # Script Python XOR
│   └── pic13.png                   # Success sur l'émulateur
├── UnCrackable-Level3.apk          # APK originale
├── uncrackable3/                   # Décompilé par apktool
│   ├── smali/
│   ├── lib/
│   │   ├── armeabi-v7a/libfoo.so
│   │   └── arm64-v8a/libfoo.so
│   └── AndroidManifest.xml
├── UnCrackable-Level3-patched.apk
├── UnCrackable-Level3-aligned.apk
├── UnCrackable-Level3-final.apk
├── my-release-key.jks
└── decode_key.py
```

---

##  Outils utilisés

| Outil | Usage | Lien |
|-------|-------|------|
| jadx-gui | Décompilation Java de l'APK | [github.com/skylot/jadx](https://github.com/skylot/jadx) |
| apktool | Décompilation/recompilation smali | [apktool.org](https://apktool.org) |
| Ghidra | Analyse de la librairie native `.so` | [ghidra-sre.org](https://ghidra-sre.org) |
| Android Studio | Émulateur Android | [developer.android.com](https://developer.android.com/studio) |
| adb | Installation de l'APK | Inclus dans Android Studio |
| Python 3 | Décodage XOR | [python.org](https://python.org) |

---

##  Disclaimer

> Ce lab est réalisé dans un cadre **pédagogique** sur une application conçue intentionnellement vulnérable ([OWASP MSTG Crackmes](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes)).  
> Ne jamais appliquer ces techniques sur des applications tierces sans autorisation explicite.
