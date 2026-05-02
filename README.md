# LAB 7 — Analyse Dynamique Mobile avec MobSF
 
## Informations générales
 
| Champ | Détail |
|---|---|
| **Outil** | MobSF (Mobile Security Framework) v4.5.0 |
| **Cible** | DIVA Android — `DivaApplication.apk` |
| **Package** | `jakhar.aseem.diva` |
| **Environnement** | Docker + Android Studio AVD (API 30 — Google APIs) |
| **Score sécurité** | 36 / 100 |
| **Date** | 02 Mai 2026 |
 
---
 
## Objectif
 
Réaliser une **analyse dynamique** d'une application Android vulnérable (DIVA) avec MobSF, en connectant un émulateur Android via ADB depuis un conteneur Docker.
 
---
 
## Étape 1 — Lancement de Docker Desktop
 
Lors de la première tentative de pull de l'image MobSF, Docker Desktop n'était pas lancé.
 
```
error during connect: open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
```
 
> **Fix :** Lancer Docker Desktop depuis le menu Démarrer et attendre l'initialisation complète.
 
![Docker Desktop non démarré](screenshots/01_docker_not_running.png)
 
---
 
## Étape 2 — Dépôt GitHub DIVA
 
L'APK de DIVA se trouve dans le repo `payatu/diva-android`. Le repo contient le code source et l'APK précompilé dans `app/app-debug.apk`.
 
![Repo GitHub DIVA](screenshots/02_github_diva.png)
 
---
 
## Étape 3 — Erreur de connexion MobSF (première fois)
 
MobSF ne pouvait pas joindre l'émulateur Android.
 
```
Cannot Connect to host.docker.internal:5555
```
 
> **Cause :** Le conteneur Docker était lancé sans la variable `MOBSF_ANALYZER_IDENTIFIER`.
 
![Erreur connexion MobSF 1](screenshots/03_mobsf_connexion_error.png)
 
---
 
## Étape 4 — Configuration ADB
 
Vérification et connexion de l'émulateur en mode TCP.
 
```bash
adb devices
adb tcpip 5555
adb connect 127.0.0.1:5555
```
 
> **Résultat :** `connected to 127.0.0.1:5555`
 
![ADB connect](screenshots/04_adb_connect.png)
 
---
 
## Étape 5 — Erreur connexion MobSF (persistante)
 
Même après la connexion ADB, MobSF continuait à échouer.
 
```
Cannot Connect to host.docker.internal:5555
```
 
> **Cause :** La commande Docker ne passait pas `MOBSF_ANALYZER_IDENTIFIER=emulator-5554` — ce nom est local à Windows et inaccessible depuis Docker. Il faut utiliser `host.docker.internal:5555`.
 
![Erreur connexion MobSF 2](screenshots/05_mobsf_connexion_error2.png)
 
---
 
## Étape 6 — Deux devices ADB en conflit
 
En relançant `adb tcpip 5555`, ADB retournait une erreur car deux devices étaient connectés simultanément.
 
```
error: more than one device/emulator
```
 
```bash
# Solution — déconnecter l'entrée TCP en doublon
adb disconnect 127.0.0.1:5555
 
# Puis cibler explicitement l'émulateur
adb -s emulator-5554 tcpip 5555
```
 
![ADB deux devices](screenshots/06_adb_two_devices.png)
 
---
 
## Étape 7 — Sélection de la bonne image AVD
 
MobSF nécessite une image **sans Google Play Store** pour pouvoir écrire sur `/system`.
 
| Image | Compatible MobSF |
|---|---|
| Google Play | ❌ NON — `/system` en lecture seule |
| **Google APIs** | ✅ OUI |
 
> Image choisie : **Google APIs Intel x86 Atom System Image — API 30**
 
![Sélection image AVD](screenshots/07_avd_google_apis.png)
 
---
 
## Étape 8 — Erreur /system non accessible
 
Même avec la bonne image, l'émulateur lancé normalement bloquait MobSF :
 
```
VM's /system is not writable. This VM cannot be used for Dynamic Analysis.
Please start the AVD as per MobSF documentation!
```
 
> **Fix :** Lancer l'émulateur avec le script MobSF ou le flag `-writable-system` :
 
```bash
# Via script MobSF
scripts\start_avd.ps1 Pixel_5
 
# Ou manuellement
emulator -avd Pixel_5 -writable-system -no-snapshot
```
 
![Erreur system non writable](screenshots/08_mobsf_dynamic_error.png)
 
---
 
## Étape 9 — Lancement réussi de l'émulateur MobSF
 
Le script `start_avd.ps1` confirme que le système est accessible en écriture :
 
```
WARNING | System image is writable   ✅
Starting AVD Pixel_5 on port 5554.
Waiting for emulator to boot...
```
 
![Emulateur lancé MobSF](screenshots/09_avd_start_success.png)
 
---
 
## Étape 10 — MobSF démarré avec succès
 
Commande finale correcte :
 
```bash
docker run -it --rm \
  -p 8000:8000 \
  -p 1337:1337 \
  -e MOBSF_ANALYZER_IDENTIFIER=host.docker.internal:5555 \
  opensecurity/mobile-security-framework-mobsf:latest
```
 
> MobSF v4.5.0 démarre et est accessible sur `http://localhost:8000`
 
![MobSF démarré](screenshots/10_mobsf_started.png)
 
---
 
## Étape 11 — Analyse Statique de DIVA
 
Upload de `DivaApplication.apk` → MobSF effectue l'analyse statique automatiquement.
 
| Indicateur | Résultat |
|---|---|
| **Score sécurité** | 36 / 100 |
| Trackers détectés | 0 / 432 |
| Activités exportées | 2 / 17 |
| Providers exportés | 1 / 1 |
| Target SDK | 23 |
 
![Analyse statique DIVA](screenshots/11_static_analysis.png)
 
---
 
## Étape 12 — Démarrage de l'Analyse Dynamique
 
Clic sur **Start Dynamic Analysis** → MobSF installe l'APK sur l'émulateur, injecte Frida et démarre l'instrumentation.
 
Scripts Frida activés automatiquement :
- ✅ API Monitoring
- ✅ SSL Pinning Bypass
- ✅ Root Detection Bypass
- ✅ Debugger Check Bypass
- ✅ Clipboard Monitor
```
Collecting data...
Downloading logs.
Stopping Application.
Application data dumped!
Generating Report...Please Wait!
```
 
![Dynamic Analyzer](screenshots/12_dynamic_analyzer.png)
 
---
 
## Étape 13 — Interaction avec DIVA (Insecure Data Storage - Part 1)
 
Objectif du challenge : trouver comment les credentials sont stockés.
 
- Username : `google`
- Password : `••••••`
- Action : `SAVE`
> L'application stocke les données **en clair** sans chiffrement (SharedPreferences).
 
![DIVA Data Storage](screenshots/13_diva_data_storage.png)
 
---
 
## Étape 14 — Application DIVA en cours d'exécution
 
L'application DIVA tourne sur l'émulateur avec tous les modules disponibles :
 
1. Insecure Logging
2. Hardcoding Issues - Part 1
3. Insecure Data Storage - Part 1 ← *testé*
4. Insecure Data Storage - Part 2
5. Insecure Data Storage - Part 3
6. Insecure Data Storage - Part 4
7. Input Validation Issues - Part 1
8. Input Validation Issues - Part 2
9. Access Control Issues - Part 1
10. Access Control Issues - Part 2
![DIVA App](screenshots/14_diva_app.png)
 
---
 
## Étape 15 — Flux Logcat
 
MobSF capture en temps réel les logs système de l'application via `http://localhost:8000/logcat/?package=jakhar.aseem.diva`.
 
Événements notables capturés :
- Installation du package `jakhar.aseem.diva` (broadcast `PACKAGE_ADDED`)
- Démarrage de `MainActivity` via `ActivityTaskManager`
- Appels à `MediaProvider.read_external_storage`
- Activité `ChromeSync` et `AffiliationManager` liée au package
![Logcat](screenshots/15_logcat.png)
 
---
 
## Résumé des Problèmes et Solutions
 
| Problème | Cause | Solution |
|---|---|---|
| Docker ne démarre pas | Docker Desktop non lancé | Lancer Docker Desktop |
| `Cannot Connect to host.docker.internal:5555` | Variable `MOBSF_ANALYZER_IDENTIFIER` manquante | Ajouter `-e MOBSF_ANALYZER_IDENTIFIER=host.docker.internal:5555` |
| `emulator-5554` inaccessible depuis Docker | Nom ADB local non résolvable dans le container | Utiliser `host.docker.internal:5555` |
| `more than one device/emulator` | Deux devices ADB simultanés | `adb disconnect 127.0.0.1:5555` |
| `/system is not writable` | Émulateur lancé sans `-writable-system` | Utiliser `scripts\start_avd.ps1 Pixel_5` |
| Image AVD incompatible | Image Google Play Store | Choisir image **Google APIs** sans Play Store |
 
---





