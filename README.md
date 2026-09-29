
# VENTUNO Q — Notes et tests

Notes personnelles concernant mes premiers essais avec l'Arduino VENTUNO Q.

## Pavé numérique sous Ubuntu / Wayland

Le pavé numérique de mon clavier HP était bien reconnu par Linux, mais
fonctionnait comme un pavé de navigation (`Home`, `↑`, `Page Up`, etc.)
au lieu de saisir des chiffres.

### Diagnostic

La session graphique utilise Wayland :

``` bash
echo $XDG_SESSION_TYPE
# wayland
```

GNOME indiquait pourtant que Num Lock était activé et mémorisé :

``` bash
gsettings get org.gnome.desktop.peripherals.keyboard numlock-state
# true

gsettings get org.gnome.desktop.peripherals.keyboard remember-numlock-state
# true
```

Un contrôle avec `libinput` a confirmé que le clavier envoyait
correctement les codes du pavé numérique. Par exemple, la touche `7`
envoyait bien :

``` text
KEY_KP7 (71)
```

### Solution

Réinitialisation de l'état Num Lock dans GNOME :

``` bash
gsettings set org.gnome.desktop.peripherals.keyboard numlock-state false
gsettings set org.gnome.desktop.peripherals.keyboard numlock-state true
```

Le changement n'a pas pris effet immédiatement dans la session Wayland
en cours.

Après une **déconnexion puis reconnexion de la session Ubuntu**, le pavé
numérique fonctionnait normalement.

### Remarque

`numlockx` a également été testé, mais n'était pas nécessaire dans cette
configuration Wayland. Il a donc été désinstallé.

------------------------------------------------------------------------

## Vérification syntaxique d'un fichier Python

Lors des essais de la WebRadio sur la **VENTUNO Q**, une erreur de syntaxe a été introduite accidentellement dans le fichier `radio_service.py`.

Le problème n'était pas immédiatement évident dans les logs : l'application principale pouvait être indiquée comme démarrée alors que le service Python utilisé par la WebRadio ne fonctionnait pas correctement.

### Vérification avec `py_compile`

Python permet de vérifier rapidement la syntaxe d'un fichier sans lancer l'application complète.

Si l'emplacement du fichier n'est pas connu, il peut d'abord être recherché avec :

```bash
find ~ -name "radio_service.py" 2>/dev/null
```

Dans le cas de la WebRadio, le fichier se trouvait ici :

```text
/home/arduino/ArduinoApps/uno-q-webradio-brick-main/bricks/webradio/radio_service.py
```

Il suffit ensuite de se placer dans le répertoire correspondant :

```bash
cd ~/ArduinoApps/uno-q-webradio-brick-main/bricks/webradio
```

puis d'effectuer la vérification :

```bash
python3 -m py_compile radio_service.py
```

Si le fichier ne contient aucune erreur de syntaxe, la commande ne produit aucun message et rend simplement la main.

### Exemple avec une erreur volontaire

Pour vérifier le fonctionnement de cette méthode, une erreur a été volontairement ajoutée au début du fichier :

```python
icifrom http.server import BaseHTTPRequestHandler, HTTPServer
```

La commande :

```bash
python3 -m py_compile radio_service.py
```

signale immédiatement l'erreur :

```text
File "radio_service.py", line 1
    icifrom http.server import BaseHTTPRequestHandler, HTTPServer
            ^^^^
SyntaxError: invalid syntax
```

Cette commande constitue donc un premier test très simple après la modification d'un fichier Python :

```bash
python3 -m py_compile fichier.py
```

Elle permet de vérifier la syntaxe du fichier avant de rechercher un éventuel problème dans App Lab, Docker, ALSA ou le matériel.

---

# VENTUNO Q - Diagnostic et correction de l'accélération GPU de Firefox

## Contexte

Sur la VENTUNO Q, Firefox présentait une certaine lenteur. Lancé depuis
un terminal, il affichait également plusieurs erreurs Mesa liées au GPU
Adreno 623.

L'objectif du diagnostic était de déterminer si le problème provenait : 

- du processeur ou de la mémoire ; 
- du stockage eMMC ;
- du pilote graphique du système ;
- des permissions d'accès au GPU ;
- ou de l'environnement Snap utilisé par Firefox.

Le diagnostic a finalement montré que l'accélération graphique de la
VENTUNO Q fonctionnait correctement au niveau du système, mais que
Firefox utilisait une ancienne version du runtime graphique `mesa-2404`
de Snap.

------------------------------------------------------------------------

## 1. Vérification de la mémoire

``` bash
free -h
```

Résultat observé :

``` text
              total        used        free      shared  buff/cache   available
Mem:           14Gi        2.4Gi       10Gi       293Mi       2.7Gi        12Gi
Swap:            0B          0B          0B
```

La mémoire disponible était très importante. Le problème ne provenait
donc pas d'un manque de RAM.

L'ajout d'un fichier swap sur l'eMMC n'aurait pas accéléré Firefox : le
swap sert essentiellement à faire face à une pression mémoire et reste
beaucoup plus lent que la RAM.

------------------------------------------------------------------------

## 2. Vérification du gouverneur CPU

``` bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Le gouverneur utilisé était `schedutil`.

Pour effectuer un essai temporaire en mode performances :

``` bash
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    if [ -f "$cpu/cpufreq/scaling_governor" ]; then
        echo performance | sudo tee "$cpu/cpufreq/scaling_governor"
    fi
done
```

Une légère amélioration a été constatée, mais cela n'expliquait pas le
problème graphique de Firefox.

Cette modification est temporaire et n'a pas été rendue permanente.

------------------------------------------------------------------------

## 3. Temps de démarrage du système

``` bash
systemd-analyze
```

Résultat observé :

``` text
Startup finished in 8.289s (kernel) + 20.475s (userspace) = 28.764s
graphical.target reached after 19.816s in userspace.
```

Pour rechercher les services les plus longs au démarrage :

``` bash
systemd-analyze blame
```

Les principaux services observés comprenaient notamment
`NetworkManager-wait-online`, Docker, `cloud-init` et Snap.

Ces éléments peuvent influencer le démarrage du système, mais ils
n'expliquaient pas le problème d'accélération de Firefox.

------------------------------------------------------------------------

## 4. Vérification de l'eMMC

Le stockage système est une eMMC d'environ 64 Go.

Une mesure simple a été réalisée avec :

``` bash
sudo hdparm -Tt /dev/mmcblk0
```

Résultats observés :

``` text
Timing cached reads: 8438 MB in 2.00 seconds = 4223.58 MB/sec
Timing buffered disk reads: 534 MB in 3.00 seconds = 177.96 MB/sec
```

La première valeur mesure essentiellement les accès via le cache
mémoire. La seconde, environ 178 MB/s, donne une indication plus utile
des lectures séquentielles de l'eMMC.

Ces performances ne permettaient pas d'expliquer à elles seules le
comportement graphique de Firefox.

------------------------------------------------------------------------

## 5. Lancement de Firefox depuis le terminal

``` bash
firefox
```

Avant correction, plusieurs erreurs graphiques apparaissaient :

``` text
MESA: error: fd_pipe_new2:49: unsupported GPU id 0x26f / chip id 0x6020300
libEGL warning: egl: failed to create dri2 screen
TU: error: device (chip_id = 6020300, gpu_id = 623) is unsupported
MESA: error: ZINK: failed to choose pdev
```

Dans Firefox, la page :

``` text
about:support
```

indiquait notamment :

``` text
WebRender (Software)
llvmpipe
```

`llvmpipe` est le moteur de rendu logiciel de Mesa : les calculs
graphiques étaient donc effectués par le CPU au lieu d'utiliser
normalement l'Adreno 623.

------------------------------------------------------------------------

## 6. Vérification des périphériques DRM

``` bash
ls -l /dev/dri/
```

Résultat :

``` text
card0
renderD128
```

Les périphériques DRM nécessaires étaient donc présents.

Vérification des groupes de l'utilisateur :

``` bash
groups
```

L'utilisateur appartenait notamment aux groupes :

``` text
video
render
```

Le problème ne provenait donc pas d'une absence évidente de permission
d'accès au GPU.

------------------------------------------------------------------------

## 7. Vérification du pilote noyau Qualcomm

``` bash
lsmod | grep -E 'msm|drm'
```

Le module `msm` était chargé, ainsi que les composants DRM associés.

Une autre vérification :

``` bash
cat /sys/class/drm/card*/device/uevent
```

montrait notamment :

``` text
DRIVER=msm_dpu
OF_COMPATIBLE_0=qcom,qcs8300-dpu
```

`msm_dpu` concerne le contrôleur d'affichage Qualcomm. Cette information
seule ne suffit toutefois pas à prouver que le rendu 3D utilise bien le
GPU.

------------------------------------------------------------------------

## 8. Test décisif : vérification de Mesa avec EGL

L'outil `eglinfo` a été utilisé pour déterminer quel moteur de rendu
était réellement disponible au niveau du système.

Après installation de l'outil si nécessaire :

``` bash
sudo apt install mesa-utils-extra
```

puis :

``` bash
eglinfo -B
```

Le résultat a montré :

``` text
OpenGL core profile vendor: freedreno
OpenGL core profile renderer: Adreno623
OpenGL core profile version: 4.6 (Core Profile) Mesa 25.2.8
```

Le même GPU était correctement détecté avec Wayland, GBM, X11 et en mode
surfaceless.

### Conclusion intermédiaire

Le système Ubuntu de la VENTUNO Q utilisait correctement :

``` text
Mesa 25.2.8
      ↓
freedreno
      ↓
Adreno 623
```

Le GPU et la pile graphique du système fonctionnaient donc correctement.

Le problème était spécifique à Firefox.

------------------------------------------------------------------------

## 9. Identification de Firefox comme application Snap

``` bash
snap list
```

Firefox était installé sous forme de Snap.

La liste montrait notamment :

``` text
firefox
core24
gnome-46-2404
mesa-2404
```

À ce moment-là, `mesa-2404` utilisait :

``` text
25.0.7-snap211
revision 1166
```

alors que le système Ubuntu utilisait déjà Mesa 25.2.8.

Cela expliquait pourquoi le système reconnaissait correctement l'Adreno
623 tandis que Firefox échouait et revenait à `llvmpipe`.

------------------------------------------------------------------------

## 10. Vérification des versions disponibles de mesa-2404

Commande :

``` bash
snap info mesa-2404
```

Résultat déterminant :

``` text
tracking: latest/stable

latest/stable: 25.2.8-snap288 2026-07-13 (1836)
latest/beta:   25.2.8-snap300 2026-09-29 (1976)
latest/edge:   25.2.8-snap300 2026-09-29 (1970)

installed:     25.0.7-snap211            (1166)
```

La VENTUNO Q possédait donc une ancienne révision de `mesa-2404`, alors
qu'une version 25.2.8 était déjà disponible dans le canal **stable**.

Il n'était pas nécessaire d'utiliser les canaux `beta` ou `edge`.

------------------------------------------------------------------------

## 11. Correction

Firefox a d'abord été fermé.

La mise à jour du runtime Mesa Snap a ensuite été effectuée :

``` bash
sudo snap refresh mesa-2404
```

Résultat :

``` text
mesa-2404 25.2.8-snap288 from Canonical refreshed
```

Le runtime est ainsi passé de :

``` text
25.0.7-snap211 (1166)
```

à :

``` text
25.2.8-snap288 (1836)
```

------------------------------------------------------------------------

## 12. Vérification après mise à jour

Firefox a été relancé :

``` bash
firefox
```

Les erreurs précédentes concernant :

``` text
unsupported GPU id
gpu_id = 623 is unsupported
ZINK: failed to choose pdev
failed to create dri2 screen
```

avaient disparu.

Un message relatif à `GL_EXT_shader_texture_lod` pouvait encore
apparaître, mais il était distinct du problème initial de détection du
GPU.

La vérification finale a été effectuée dans :

``` text
about:support
```

Firefox indiquait désormais :

``` text
Compositing:        WebRender
GPU:                Adreno623
Driver vendor:      mesa/msm
Driver version:     25.2.8.0
Window protocol:    Wayland
WebGL 1 renderer:   freedreno -- Adreno623
WebGL 2 renderer:   freedreno -- Adreno623
```

Les fonctions suivantes étaient également indiquées comme disponibles :

``` text
HW_COMPOSITING
OPENGL_COMPOSITING
WEBRENDER
DMABUF
HARDWARE_VIDEO_DECODING
```

Le décodage matériel était notamment disponible pour H.264, VP9 et HEVC.

------------------------------------------------------------------------

## 13. Avant / après

### Avant

``` text
Firefox Snap
     ↓
mesa-2404 25.0.7
     ↓
Adreno 623 non correctement pris en charge dans ce runtime
     ↓
llvmpipe
     ↓
rendu logiciel par le CPU
```

### Après

``` text
Firefox Snap
     ↓
mesa-2404 25.2.8
     ↓
Mesa / freedreno
     ↓
pilote msm
     ↓
Adreno 623
     ↓
accélération GPU
```

------------------------------------------------------------------------

## 14. Commandes essentielles pour reproduire le diagnostic

Pour un diagnostic rapide, les commandes réellement déterminantes sont :

### Vérifier le moteur de rendu du système

``` bash
eglinfo -B
```

Rechercher :

``` text
freedreno
Adreno623
```

### Vérifier les Snaps installés

``` bash
snap list
```

### Vérifier la version de mesa-2404 et les versions disponibles

``` bash
snap info mesa-2404
```

### Mettre à jour uniquement si une version stable plus récente est proposée

``` bash
sudo snap refresh mesa-2404
```

### Vérifier Firefox

Lancer :

``` bash
firefox
```

puis ouvrir :

``` text
about:support
```

Rechercher notamment :

``` text
WebRender
Adreno623
freedreno
llvmpipe
```

Si `llvmpipe` est utilisé comme renderer principal, Firefox effectue
encore le rendu graphique en logiciel.

------------------------------------------------------------------------

## 15. Remarque importante concernant les mises à jour

App Lab indiquait :

``` text
System is up to date
```

pour l'image système de la VENTUNO Q.

Cela n'impliquait cependant pas que chaque composant Snap disposait de
sa dernière révision stable.

Dans ce cas précis :

``` text
Image système / Ubuntu : à jour
Mesa système           : 25.2.8
mesa-2404 Snap         : 25.0.7 → mise à jour disponible en 25.2.8
```

Les mises à jour de l'image système Arduino et celles des paquets Snap
suivent donc des mécanismes distincts.

Cette distinction peut être utile lors du diagnostic d'autres
applications Snap utilisant l'accélération graphique.

------------------------------------------------------------------------

## Configuration utilisée lors du diagnostic

``` text
Carte :             Arduino VENTUNO Q
Ubuntu :            24.04.4 LTS
Kernel :            6.8.0-1084-qcom
Image VENTUNO Q :   20260710-235
GPU :               Qualcomm Adreno 623
Session graphique : Wayland
Firefox :           151.0.4
Mesa système :      25.2.8
mesa-2404 corrigé : 25.2.8-snap288 (revision 1836)
```

------------------------------------------------------------------------

*D'autres notes sur la VENTUNO Q pourront être ajoutées à ce dépôt  au fil
des essais.*
