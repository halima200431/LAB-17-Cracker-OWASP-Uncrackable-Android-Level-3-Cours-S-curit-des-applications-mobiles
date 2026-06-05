# Rapport LAB 17 — Cracker OWASP UnCrackable Android Level 3

## 1. Introduction

Ce laboratoire a pour objectif d’analyser et de contourner les protections de l’application **OWASP UnCrackable Android Level 3** dans un environnement contrôlé. L’application contient plusieurs mécanismes de protection : vérification anti-root, vérification d’intégrité, détection anti-debug, détection anti-Frida/anti-Xposed et vérification native du mot de passe.

Le but final du lab est de comprendre le fonctionnement de ces protections, de les neutraliser proprement, puis d’extraire le secret attendu par l’application.

> Ce travail est réalisé uniquement dans un cadre pédagogique sur une application volontairement conçue pour l’apprentissage du reverse engineering Android.

---

## 2. Objectifs du lab

À la fin de ce lab, les objectifs suivants ont été atteints :

- Installer et exécuter l’APK OWASP UnCrackable Level 3.
- Analyser le code Java avec JADX.
- Décompiler l’APK avec apktool.
- Modifier le code smali pour contourner les vérifications Java.
- Reconstruire, signer et réinstaller une APK patchée.
- Analyser la librairie native `libfoo.so` avec Ghidra.
- Identifier et neutraliser la fonction anti-Frida / anti-Xposed.
- Comprendre la logique de vérification native.
- Décoder le secret avec une opération XOR.
- Valider le secret dans l’application.

---

## 3. Environnement de travail

| Élément | Valeur utilisée |
|---|---|
| Système hôte | Windows |
| Dossier du lab | `C:\APK-Analysis\Lab17-Uncrackable3` |
| Émulateur Android | Android Emulator |
| Architecture ABI | `x86_64` |
| APK analysée | `UnCrackable-Level3.apk` |
| Outils utilisés | Android Studio, ADB, JADX, apktool, apksigner, Ghidra, Python |
| Librairie native analysée | `libfoo.so` |

---

## 4. Installation de l’APK originale

L’APK officielle a été téléchargée puis installée sur l’émulateur Android.

Commande de téléchargement :

```powershell
Invoke-WebRequest -Uri "https://github.com/OWASP/owasp-mstg/raw/master/Crackmes/Android/Level_03/UnCrackable-Level3.apk" -OutFile "UnCrackable-Level3.apk"
```

Comme la commande `adb` n’était pas reconnue directement dans PowerShell, le chemin complet de l’outil ADB a été utilisé :

```powershell
& "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe" install .\UnCrackable-Level3.apk
```

Résultat obtenu :

```text
Performing Streamed Install
Success
```

Après installation, l’application s’est ouverte correctement et a affiché l’écran principal avec le champ **Enter the Secret String** et le bouton **VERIFY**.

<img width="350" height="677" alt="Capture d&#39;écran 2026-06-05 163647" src="https://github.com/user-attachments/assets/960cf1ed-f90e-49bc-a361-ee37c4add4e1" />
-

## 5. Analyse statique avec JADX

L’APK a été ouverte avec **JADX-GUI** afin d’analyser le code Java décompilé.

Le package principal est :

```text
sg.vantagepoint.uncrackable3
```

La classe principale étudiée est :

```text
MainActivity
```

Dans cette classe, plusieurs éléments importants ont été identifiés.

### 5.1 Chargement de la librairie native

Le code Java charge une librairie native nommée `foo` :

```java
System.loadLibrary("foo");
```

Cela signifie que la logique importante de l’application n’est pas entièrement écrite en Java. Une partie du contrôle est réalisée dans le fichier natif :

```text
libfoo.so
```

### 5.2 Vérification d’intégrité

La méthode `verifyLibs()` est utilisée pour vérifier l’intégrité des fichiers de l’application. Si une modification est détectée, la variable `tampered` peut indiquer que l’application a été altérée.

### 5.3 Détection root et debug

Dans la méthode `onCreate()`, l’application effectue plusieurs vérifications :

```java
RootDetection.checkRoot1()
RootDetection.checkRoot2()
RootDetection.checkRoot3()
IntegrityCheck.isDebuggable(...)
tampered
```

Si une condition suspecte est détectée, l’application affiche le message :

```text
Rooting or tampering detected.
```

<img width="945" height="311" alt="Capture d&#39;écran 2026-06-05 165725" src="https://github.com/user-attachments/assets/65f1ffdd-301e-444b-9b4f-e39d6f888864" />


## 6. Décompilation de l’APK avec apktool

L’APK a ensuite été décompilée avec apktool afin d’obtenir le code smali et les ressources modifiables.

Commande utilisée :

```powershell
java -jar .\apktool.jar d .\UnCrackable-Level3.apk -o .\uncrackable3
```
![Uploading Capture d'écran 2026-06-05 165725.png…]()

Le dossier généré contient notamment :

```text
uncrackable3/
├── smali/
├── lib/
├── res/
└── AndroidManifest.xml
```

Le fichier smali principal se trouve dans :

```text
uncrackable3/smali/sg/vantagepoint/uncrackable3/MainActivity.smali
```



## 7. Patch smali anti-root / anti-tampering

Le fichier suivant a été ouvert dans VS Code :

```text
C:\APK-Analysis\Lab17-Uncrackable3\uncrackable3\smali\sg\vantagepoint\uncrackable3\MainActivity.smali
```

Le bloc responsable de l’affichage du message de détection a été localisé :

```smali
:cond_0
const-string v0, "Rooting or tampering detected."

.line 127
invoke-direct {p0, v0}, Lsg/vantagepoint/uncrackable3/MainActivity;->showDialog(Ljava/lang/String;)V
```

Ce bloc a été remplacé par un saut direct vers la suite normale de l’exécution :

```smali
:cond_0
goto :cond_1
```

Ainsi, même si une détection root, debug ou tampering est déclenchée, l’application ne lance plus la boîte de dialogue d’erreur et continue son initialisation.


## 8. Reconstruction et signature de l’APK patchée

Après modification du fichier smali, l’APK a été reconstruite avec apktool.

Commande utilisée :

```powershell
java -jar .\apktool.jar b .\uncrackable3 -o .\UnCrackable-Level3-patched-unsigned.apk
```
<img width="958" height="162" alt="Capture d&#39;écran 2026-06-05 165733" src="https://github.com/user-attachments/assets/7c8999c4-ca62-4278-85fb-61ee3fcf5125" />

Résultat obtenu :

```text
I: Using Apktool 3.0.2 on UnCrackable-Level3.apk with 8 threads
I: Smaling smali folder into classes.dex...
I: Building resources with aapt2...
I: Building apk file...
I: Importing lib...
I: Built apk into: .\UnCrackable-Level3-patched-unsigned.apk
```

Ensuite, l’APK a été signée avec `apksigner` :

```powershell
$apksigner="C:\Users\halim\AppData\Local\Android\Sdk\build-tools\37.0.0\apksigner.bat"

& $apksigner sign --ks "$env:USERPROFILE\.android\debug.keystore" --ks-pass pass:android --key-pass pass:android --out .\UnCrackable-Level3-patched.apk .\UnCrackable-Level3-patched-unsigned.apk
```

L’ancienne version a ensuite été remplacée par la version patchée :

```powershell
$adb="$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
& $adb install .\UnCrackable-Level3-patched.apk
```

Résultat obtenu :

```text
Performing Incremental Install
Performing Streamed Install
Success
```
<img width="942" height="71" alt="image" src="https://github.com/user-attachments/assets/b02d5340-2f1a-4432-8350-2e6bae5d5eeb" />



## 9. Vérification de l’architecture de l’émulateur

Avant d’analyser la librairie native, l’architecture de l’émulateur a été vérifiée avec la commande suivante :

```powershell
& $adb shell getprop ro.product.cpu.abi
```

Résultat obtenu :

```text
x86_64
```

La librairie native à analyser est donc :

```text
C:\APK-Analysis\Lab17-Uncrackable3\uncrackable3\lib\x86_64\libfoo.so
```

---

## 10. Analyse native avec Ghidra

La librairie `libfoo.so` a été importée dans Ghidra.

Fichier importé :

```text
C:\APK-Analysis\Lab17-Uncrackable3\uncrackable3\lib\x86_64\libfoo.so
```

Après l’analyse automatique, une recherche de chaînes a été réalisée avec **Search > For Strings**.

Les chaînes suivantes ont été trouvées :

```text
/proc/self/maps
frida
xposed
```

Ces chaînes indiquent que la librairie native vérifie la mémoire du processus afin de détecter la présence d’outils d’instrumentation comme Frida ou Xposed.
<img width="953" height="193" alt="Capture d&#39;écran 2026-06-05 175520" src="https://github.com/user-attachments/assets/63df3072-58df-4f83-b396-91e0d2492f91" />
<img width="952" height="231" alt="Capture d&#39;écran 2026-06-05 175507" src="https://github.com/user-attachments/assets/127bd65d-31b7-4dba-9e73-19d585c33062" />
<img width="965" height="196" alt="Capture d&#39;écran 2026-06-05 175444" src="https://github.com/user-attachments/assets/e9c0066b-f462-473a-9d43-183ddcce38f7" />



## 11. Identification de la fonction anti-Frida / anti-Xposed

La chaîne `/proc/self/maps` est référencée dans la fonction :

```text
FUN_001037c0
```

Les références observées sont :

```text
FUN_001037c0:001037d1
FUN_001037c0:001037f3
FUN_001037c0:00103859
```

Dans cette fonction, on observe l’ouverture du fichier `/proc/self/maps` avec un appel à `fopen` :

```asm
001037d1  LEA  RDI,[s_/proc/self/maps_00103b50]
001037df  CALL fopen
```

Cette fonction est donc responsable de l’analyse de la mémoire du processus pour détecter Frida, Xposed ou d’autres traces d’instrumentation.

<img width="1097" height="312" alt="Capture d&#39;écran 2026-06-05 180340" src="https://github.com/user-attachments/assets/8b4916ae-e4af-4cb4-94ee-f73982c14402" />


## 12. Patch natif avec Ghidra

La fonction `FUN_001037c0` commençait par l’instruction suivante :

```asm
001037c0  PUSH RBP
```

En architecture x86_64, le byte correspondant était :

```text
55
```

Pour neutraliser la fonction, cette première instruction a été remplacée par :

```asm
RET
```

Après patch, la fonction commence par :

```asm
001037c0  c3  RET
```

Le byte `c3` correspond à l’instruction `RET` en x86_64. Ainsi, la fonction retourne immédiatement sans exécuter les vérifications anti-Frida et anti-Xposed.

<img width="1151" height="203" alt="Capture d&#39;écran 2026-06-05 180640" src="https://github.com/user-attachments/assets/7c6f47ce-b8ff-40fb-a30c-185eb0ba386d" />


## 13. Export de la librairie patchée

Après le patch dans Ghidra, la librairie modifiée a été exportée avec :

```text
File > Export Program > Format: Original File
```

Chemin d’export utilisé :

```text
C:\APK-Analysis\Lab17-Uncrackable3\patched_so\libfoo.so
```

La librairie patchée a ensuite remplacé l’ancienne librairie dans le dossier apktool :

```powershell
Copy-Item "C:\APK-Analysis\Lab17-Uncrackable3\patched_so\libfoo.so" "C:\APK-Analysis\Lab17-Uncrackable3\uncrackable3\lib\x86_64\libfoo.so" -Force
```

---

## 14. Reconstruction, signature et installation de l’APK native patchée

L’APK a été reconstruite une deuxième fois après modification de la librairie native.

Commande de reconstruction :

```powershell
java -jar .\apktool.jar b .\uncrackable3 -o .\UnCrackable-Level3-nativepatched-unsigned.apk
```

Commande de signature :

```powershell
$apksigner="C:\Users\halim\AppData\Local\Android\Sdk\build-tools\37.0.0\apksigner.bat"

& $apksigner sign --ks "$env:USERPROFILE\.android\debug.keystore" --ks-pass pass:android --key-pass pass:android --out .\UnCrackable-Level3-nativepatched.apk .\UnCrackable-Level3-nativepatched-unsigned.apk
```

Commande d’installation :

```powershell
$adb="$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"

& $adb install -r .\UnCrackable-Level3-nativepatched.apk
```

Résultat obtenu :

```text
Performing Incremental Install
Performing Streamed Install
Success
```

<img width="953" height="92" alt="image" src="https://github.com/user-attachments/assets/8dedd28e-af9d-427e-a0f6-7e000e1b0a68" />


## 15. Analyse de la vérification du secret

Dans le code Java, la méthode `verify(View view)` récupère la chaîne entrée par l’utilisateur puis appelle la méthode :

```java
check.check_code(string)
```

Cette méthode ne contient pas la logique finale en Java. Elle délègue la vérification à la librairie native `libfoo.so`.

La vérification native repose sur une comparaison octet par octet d’une valeur reconstruite dynamiquement. Le secret n’est pas stocké directement en clair. Il est encodé sous forme de 24 octets, puis décodé avec une opération XOR.

Les octets encodés identifiés sont :

```text
1d 08 11 13 0f 17 49 15 0d 00 03 19 5a 1d 13 15 08 0e 5a 00 17 08 13 14
```

La clé XOR utilisée est basée sur la répétition du mot :

```text
pizza
```

---

## 16. Décodage du secret avec Python

Un script Python a été utilisé pour décoder les 24 octets avec XOR.

Fichier créé :

```text
decode_key.py
```

Contenu du script :

```python
# Décodage du secret OWASP UnCrackable Level 3

encoded = bytes.fromhex(
    "1d0811130f1749150d0003195a1d1315080e5a0017081314"
)

# Répéter "pizza" puis garder exactement 24 octets
xor_key = (b"pizza" * 10)[:24]

secret = bytes(a ^ b for a, b in zip(encoded, xor_key))

print("Longueur encoded :", len(encoded))
print("Longueur xor_key :", len(xor_key))
print("Clé secrète trouvée :", secret.decode())
```

Commande d’exécution :

```powershell
python .\decode_key.py
```

Résultat obtenu :

```text
Longueur encoded : 24
Longueur xor_key : 24
Clé secrète trouvée : making owasp great again
```

Le secret final est donc :

```text
making owasp great again
```

<img width="645" height="98" alt="image" src="https://github.com/user-attachments/assets/2e3c904f-adbc-45a9-ab59-0e24ae859758" />


## 17. Validation finale dans l’application

Le secret suivant a été saisi dans l’application :

```text
making owasp great again
```

Après clic sur le bouton **VERIFY**, l’application affiche :

```text
Success!
This is the correct secret.
```

Cela confirme que le secret extrait est correct.

<img width="368" height="740" alt="Capture d&#39;écran 2026-06-05 181059" src="https://github.com/user-attachments/assets/4c679ce3-fc9f-4a41-93a1-5de5572890c1" />


## 18. Réponses aux questions de réflexion

### 18.1 Pourquoi l’obfuscateur ajoute-t-il autant d’instructions répétitives ?

L’obfuscateur ajoute beaucoup d’instructions inutiles pour compliquer la lecture du programme. Les boucles répétitives, les allocations mémoire et les listes chaînées augmentent artificiellement la taille du code et ralentissent l’analyse statique. L’objectif est de cacher la logique réellement utile au milieu d’un grand volume de code parasite.

### 18.2 Pourquoi les écritures finales dans le buffer sont-elles plus importantes que les nombreux `malloc` ?

Les nombreux `malloc` font partie de l’obfuscation et ne représentent pas directement la logique de vérification. En revanche, les écritures finales dans le buffer construisent les octets réellement utilisés pour comparer l’entrée utilisateur. Ce buffer est donc beaucoup plus important que les allocations répétitives.

### 18.3 Quel avantage de sécurité apporte une vérification native par rapport à une vérification Java ?

Une vérification native est plus difficile à analyser qu’une vérification Java. Le Java peut être décompilé facilement avec JADX, alors que le code natif nécessite des outils comme Ghidra et une compréhension de l’assembleur. Le code natif permet aussi d’utiliser des mécanismes bas niveau comme `ptrace`, `/proc/self/maps` et des contrôles anti-instrumentation.

### 18.4 Comment un développeur défensif pourrait-il rendre cette clé plus difficile à extraire ?

Un développeur pourrait éviter de stocker un secret fixe dans l’application. Il pourrait déplacer la vérification côté serveur, diviser la clé en plusieurs fragments, générer certaines parties dynamiquement, utiliser une obfuscation plus forte, vérifier l’intégrité de la librairie native et renforcer la détection des environnements d’analyse.

---

## 19. Conclusion

Ce lab a permis de comprendre les principales protections utilisées par l’application OWASP UnCrackable Android Level 3. L’analyse Java a montré que l’application effectue des vérifications anti-root, anti-debug et d’intégrité avant d’appeler une vérification native. Le patch smali a permis de contourner la détection Java, tandis que l’analyse de `libfoo.so` avec Ghidra a permis d’identifier et de neutraliser la fonction anti-Frida / anti-Xposed `FUN_001037c0`.

Enfin, l’analyse de la logique native a montré que le secret était encodé par XOR. Après décodage avec Python, le secret obtenu est :

```text
making owasp great again
```

La validation dans l’application a affiché le message **Success!**, confirmant la réussite complète du lab.

---

## 20. Checklist de validation

| Élément | État |
|---|---|
| APK originale installée | Validé |
| Application originale lancée | Validé |
| Analyse Java avec JADX | Validé |
| Décompilation avec apktool | Validé |
| Patch smali anti-root/tampering | Validé |
| Reconstruction APK patchée | Validé |
| Signature APK patchée | Validé |
| Installation APK patchée | Validé |
| ABI émulateur vérifiée (`x86_64`) | Validé |
| Analyse `libfoo.so` avec Ghidra | Validé |
| Identification de `FUN_001037c0` | Validé |
| Patch natif `RET` à l’adresse `001037c0` | Validé |
| Export de la librairie patchée | Validé |
| Reconstruction APK native patchée | Validé |
| Signature et installation finale | Validé |
| Décodage XOR avec Python | Validé |
| Secret trouvé | Validé |
| Message `Success!` obtenu | Validé |
