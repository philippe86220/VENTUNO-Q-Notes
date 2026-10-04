
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

Pour revenir en arrière :

``` bash
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    if [ -f "$cpu/cpufreq/scaling_governor" ]; then
        echo schedutil | sudo tee "$cpu/cpufreq/scaling_governor"
    fi
done
``` 

pour verifier :

``` 
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

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

## 16. Configuration utilisée lors du diagnostic

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

## Langue française et paramètres régionaux

L'image Ubuntu fournie avec la VENTUNO Q peut ne contenir par défaut que les locales anglaises.  
Vous pouvez vérifier les locales actuellement disponibles avec :

```bash
locale
locale -a
```

### Installer la langue française

Mettez à jour la liste des paquets et installez les paquets de langue française :

```bash
sudo apt update
sudo apt install language-pack-fr language-pack-gnome-fr
```

Générez la locale française :

```bash
sudo locale-gen fr_FR.UTF-8
```

Vérifiez qu'elle est disponible :

```bash
locale -a | grep -i fr
```

Le résultat doit notamment contenir :

```text
fr_FR.utf8
```

Vous pouvez ensuite ouvrir :

**Paramètres → Système → Région et langue**

et sélectionner :

- **Langue :** Français
- **Formats :** France

Déconnectez-vous puis reconnectez-vous si la nouvelle langue ou les formats régionaux ne sont pas immédiatement disponibles.

Lors de la première connexion en français, GNOME peut demander si les dossiers standards de l'utilisateur doivent être renommés, par exemple :

```text
Desktop    → Bureau
Downloads  → Téléchargements
Pictures   → Images
Music      → Musique
```

Choisissez selon votre préférence.

### Régler le fuseau horaire français

Vérifiez le fuseau horaire actuellement utilisé :

```bash
timedatectl
```

Si le système utilise UTC, définissez le fuseau horaire de Paris :

```bash
sudo timedatectl set-timezone Europe/Paris
```

Vérifiez le résultat :

```bash
timedatectl
```

Le fuseau horaire doit maintenant être indiqué comme `Europe/Paris`.

L'utilisation de `Europe/Paris` gère automatiquement le passage entre l'heure normale d'Europe centrale (**CET, UTC+1**) et l'heure d'été d'Europe centrale (**CEST, UTC+2**). Il n'est donc pas nécessaire de régler manuellement l'horloge lors des changements d'heure.

------------------------------------------------------------------------

#  04 octobre 2026 : UNO Media Carrier sur VENTUNO Q --- notes et tests audio

Cette section rassemble les essais réalisés avec le **UNO Media
Carrier** monté sur une **VENTUNO Q**, ainsi que les commandes utilisées
pour identifier et tester ses sorties audio.

> **Important :** le *User Manual* en ligne du UNO Media Carrier est la
> référence la plus explicite pour le fonctionnement des sorties audio.
> Le datasheet PDF contient une description contradictoire concernant le
> Line Out : il le présente comme une sortie stéréo, alors que le User
> Manual le décrit comme une sortie mono différentielle.

## 1. Les trois sorties audio

Le UNO Media Carrier possède trois jacks audio 3,5 mm :

-   **MIC-IN / Headphones Out** : entrée microphone + sortie casque
    stéréo ;
-   **Line Out** : sortie ligne mono différentielle (*balanced*) ;
-   **Earphones Out / Ear Out** : canal droit en sortie différentielle,
    prévu notamment pour un petit haut-parleur.

Sur la VENTUNO Q testée, ALSA expose la carte audio sous le nom
`monaco-gertrude`.

``` bash
aplay -l
```

Lors des essais, on obtient notamment :

``` text
card 0: monacogertrude [monaco-gertrude], device 0: MultiMedia1 Playback
card 0: monacogertrude [monaco-gertrude], device 2: MultiMedia3 Playback
```

Le codec audio matériel visible sur le schéma de la VENTUNO Q est un
**MAX98091ETM+ (U23)**.

## 2. Headphones --- sortie stéréo

La sortie **Headphones** est une véritable sortie stéréo.

``` bash
speaker-test -D hw:0,0 -c 2 -t wav
```

Résultat observé :

``` text
Front Left  -> haut-parleur gauche
Front Right -> haut-parleur droit
```

Tests séparés :

``` bash
speaker-test -D hw:0,0 -c 2 -t sine -s 1
speaker-test -D hw:0,0 -c 2 -t sine -s 2
```

Les essais ont été concluants avec un câble **TRS → TRS** correctement
inséré ainsi qu'avec un câble **TRRS → TRS** adapté.

### Volume Headphones

``` bash
amixer -c 0 get Headphone
amixer -c 0 set Headphone 20
```

Sur la VENTUNO Q testée, 20/31 donne un niveau d'écoute confortable.

Une sauvegarde ALSA peut être effectuée avec :

``` bash
sudo alsactl store 0
```

Mais lors des essais, WirePlumber/UCM réinitialisait ensuite le niveau.
Cela a été mis en évidence avec :

``` bash
systemctl --user restart wireplumber
```

Le niveau `Headphone` revenait alors de 20 à 10.

## 3. Line Out --- mono différentiel, pas stéréo

Au départ, le comportement du **Line Out** semblait anormal :

``` bash
speaker-test -D hw:0,0 -c 2 -t wav
```

``` text
Front Left  -> silence
Front Right -> son
```

Le même résultat a été reproduit :

-   avec un câble **TRS → TRS** ;
-   avec un câble **TRRS → TRS** ;
-   avec **deux UNO Media Carrier différents**.

À l'inverse, Headphones restitue correctement les deux canaux dans le
même environnement de test.

### Explication

Le *UNO Media Carrier User Manual* précise que le Line Out expose une
paire différentielle :

``` text
LINEOUT_P
LINEOUT_M
```

Il ne s'agit donc **pas** de Left et Right. Les deux conducteurs
constituent les deux phases d'un **unique canal mono symétrique**
(*balanced mono*), destiné notamment aux équipements possédant une
entrée symétrique.

``` text
TRS stéréo asymétrique       TRS mono symétrique

Tip    = Left                Tip    = signal +
Ring   = Right               Ring   = signal -
Sleeve = GND                 Sleeve = GND / blindage
```

Un connecteur TRS ne signifie donc pas nécessairement « stéréo ».

### Commandes indiquées par le User Manual pour Line Out

``` bash
amixer -c0 cset iface=MIXER,name='RX_CODEC_DMA_RX_0 Audio Mixer MultiMedia2' 1
amixer -c0 cset iface=MIXER,name='RX_MACRO RX0 MUX' 1
amixer -c0 cset iface=MIXER,name='RX INT0_1 MIX1 INP0' 'RX0'
amixer -c0 cset iface=MIXER,name='RX INT0 DEM MUX' 1
amixer -c0 cset iface=MIXER,name='LO_RDAC Switch' 1
amixer -c0 cset iface=MIXER,name='RX_RX0 Digital Volume' 80
```

Lecture :

``` bash
aplay -D plughw:0,1 /usr/share/sounds/alsa/Front_Center.wav
```

Fermeture du chemin :

``` bash
amixer -c0 cset iface=MIXER,name='RX_CODEC_DMA_RX_0 Audio Mixer MultiMedia2' 0
amixer -c0 cset iface=MIXER,name='RX_MACRO RX0 MUX' 'ZERO'
amixer -c0 cset iface=MIXER,name='RX INT0_1 MIX1 INP0' 'ZERO'
amixer -c0 cset iface=MIXER,name='LO_RDAC Switch' 0
```

> Ces commandes proviennent du User Manual du UNO Media Carrier. Les
> essais décrits plus haut sur la VENTUNO Q ont principalement utilisé
> `speaker-test` et la configuration audio du système.

## 4. Contradiction entre le User Manual et le datasheet

Le **User Manual en ligne** indique clairement que Line Out est un
**canal mono différentiel** (`LINEOUT_P / LINEOUT_M`).

En revanche, le **datasheet PDF du UNO Media Carrier** décrit Line Out
comme :

> "Stereo line-level output for connection to external amplifiers or
> powered speakers."

Ces deux descriptions sont contradictoires.

Les essais réalisés sur la VENTUNO Q correspondent au comportement
décrit par le **User Manual** : Line Out ne se comporte pas comme une
sortie casque stéréo Left/Right.

## 5. Ear Out --- canal droit différentiel

Le User Manual indique explicitement que **Earphone Output fournit le
canal droit sous forme d'une paire différentielle**.

Il précise qu'un petit haut-parleur peut y être connecté avec une
impédance comprise entre :

``` text
10,67 Ω à 32 Ω
```

Le haut-parleur se connecte **entre les deux conducteurs actifs** de la
paire différentielle ; la masse n'est pas utilisée comme retour du
haut-parleur.

``` text
EAR_P  --------+
               | haut-parleur
EAR_M  --------+

GND   ---------X   non utilisé comme retour du HP
```

### Essai effectué

Un petit haut-parleur **8 Ω** a été utilisé uniquement pour un essai
ponctuel :

``` text
Front Right -> son
Front Left  -> silence
```

Le niveau sonore était faible, mais le comportement correspond à la
documentation : **Ear Out fournit uniquement le canal droit**.

> **Attention :** 8 Ω est inférieur à la plage documentée de 10,67--32
> Ω. Ce test n'est pas une recommandation d'utilisation. Pour une
> utilisation normale, respecter la plage d'impédance indiquée par
> Arduino.

### Commandes indiquées par le User Manual pour Ear Out

``` bash
amixer -c0 cset iface=MIXER,name='RX_CODEC_DMA_RX_0 Audio Mixer MultiMedia2' 1
amixer -c0 cset iface=MIXER,name='RX_MACRO RX0 MUX' 1
amixer -c0 cset iface=MIXER,name='RX INT0_1 MIX1 INP0' 'RX0'
amixer -c0 cset iface=MIXER,name='RX INT0 DEM MUX' 1
amixer -c0 cset iface=MIXER,name='EAR_RDAC Switch' 1
amixer -c0 cset iface=MIXER,name='HPHL Switch' 1
amixer -c0 cset iface=MIXER,name='RX_RX0 Digital Volume' 80
```

Lecture :

``` bash
aplay -D hw:0,1 /home/arduino/recording.wav
```

Fermeture du chemin :

``` bash
amixer -c0 cset iface=MIXER,name='RX_CODEC_DMA_RX_0 Audio Mixer MultiMedia2' 0
amixer -c0 cset iface=MIXER,name='RX_MACRO RX0 MUX' 'ZERO'
amixer -c0 cset iface=MIXER,name='RX INT0_1 MIX1 INP0' 'ZERO'
amixer -c0 cset iface=MIXER,name='RX INT0 DEM MUX' 'NORMAL_DSM_OUT'
amixer -c0 cset iface=MIXER,name='EAR_RDAC Switch' 0
amixer -c0 cset iface=MIXER,name='HPHL Switch' 0
```

## 6. Résumé pratique

  -----------------------------------------------------------------------
  Sortie            Type              Canaux            Usage
  ----------------- ----------------- ----------------- -----------------
  **Headphones**    Stéréo            Left + Right      Casque,
                    asymétrique                         enceinte/AUX
                                                        stéréo

  **Line Out**      Mono différentiel Un canal mono     Entrée
                    / balanced                          symétrique, audio
                                                        pro/industriel

  **Ear Out**       Différentiel      Right uniquement  Petit
                                                        haut-parleur
                                                        10,67--32 Ω
  -----------------------------------------------------------------------

Pour une application comme une **WebRadio reliée à une enceinte JBL par
son entrée AUX stéréo**, **Headphones** est la sortie la plus adaptée.

## 7. Références Arduino

-   UNO Media Carrier :
    https://docs.arduino.cc/hardware/uno-media-carrier/
-   UNO Media Carrier User Manual :
    https://docs.arduino.cc/tutorials/uno-media-carrier/user-manual/
-   UNO Media Carrier datasheet :
    https://docs.arduino.cc/resources/datasheets/ASX00083-datasheet.pdf
-   UNO Media Carrier schematics :
    https://docs.arduino.cc/resources/schematics/ASX00083-schematics.pdf
-   VENTUNO Q : https://docs.arduino.cc/hardware/ventuno-q/

## Conclusion

Les essais audio ont d'abord laissé penser à un problème de canal gauche
sur Line Out. La comparaison avec Headphones, l'utilisation de plusieurs
câbles et de deux Media Carriers différents ont permis d'écarter un
défaut simple de câble ou de carte.

La consultation du **UNO Media Carrier User Manual** a finalement
clarifié l'architecture :

-   **Headphones** est stéréo ;
-   **Line Out** est une sortie **mono différentielle** ;
-   **Ear Out** fournit le **canal droit en différentiel** et accepte,
    selon Arduino, un petit haut-parleur de **10,67 à 32 Ω**.

Cette distinction est importante : un connecteur **TRS** peut
transporter soit un signal stéréo asymétrique, soit un signal mono
symétrique selon la conception de l'équipement.


## 8. Commandes utilisées pendant les tests sur la VENTUNO Q

Cette section regroupe les principales commandes réellement utilisées pendant les essais. Elles ont permis de distinguer ce qui relevait du flux audio ALSA, du routage vers les différentes sorties et du réglage du volume.

### Lister les périphériques audio ALSA

```bash
aplay -l
```

Cette commande affiche les cartes et périphériques ALSA disponibles pour la lecture audio.

Elle a notamment permis d'identifier sur la VENTUNO Q la carte :

```text
monaco-gertrude
```

ainsi que les périphériques de lecture disponibles, dont `MultiMedia1 Playback`.

---

### Tester les deux canaux avec une annonce vocale

```bash
speaker-test -D hw:0,0 -c 2 -t wav
```

Explication des options :

- `-D hw:0,0` : utilise directement la carte ALSA 0, périphérique 0 ;
- `-c 2` : demande un flux à deux canaux ;
- `-t wav` : utilise les fichiers WAV de test de `speaker-test`.

La commande annonce alternativement :

```text
Front Left
Front Right
```

Elle a été essentielle pour comparer les sorties physiques.

Avec **Headphones** :

```text
Front Left  -> gauche
Front Right -> droite
```

Avec **Line Out** :

```text
Front Left  -> silence
Front Right -> son
```

Le comportement Line Out a été reproduit avec deux types de câbles et deux UNO Media Carrier différents.

---

### Tester uniquement Front Left

```bash
speaker-test -D hw:0,0 -c 2 -t sine -s 1
```

- `-t sine` génère un signal sinusoïdal ;
- `-s 1` sélectionne le premier canal, **Front Left**.

Cette commande permet de tester un canal sans attendre l'alternance automatique de `speaker-test`.

Sur Line Out, aucun son n'a été obtenu avec ce canal.

---

### Tester uniquement Front Right

```bash
speaker-test -D hw:0,0 -c 2 -t sine -s 2
```

- `-s 2` sélectionne le second canal, **Front Right**.

Sur Line Out, ce test produit du son.

Il a également permis de vérifier le fonctionnement de **Ear Out** avec le petit haut-parleur utilisé pour l'essai.

---

### Afficher le volume de la sortie Headphones

```bash
amixer -c 0 get Headphone
```

- `amixer` permet de consulter ou modifier les contrôles du mixer ALSA ;
- `-c 0` sélectionne la carte audio 0 ;
- `get Headphone` affiche l'état et le niveau du contrôle `Headphone`.

Cette commande a notamment montré que le niveau par défaut était revenu à **10/31** après redémarrage.

---

### Régler le volume Headphones

```bash
amixer -c 0 set Headphone 20
```

Cette commande règle le contrôle `Headphone` à **20/31**.

Ce niveau s'est révélé adapté aux essais réalisés.

---

### Sauvegarder l'état ALSA

```bash
sudo alsactl store 0
```

Cette commande sauvegarde l'état courant des contrôles ALSA de la carte 0.

Pendant les essais, la valeur 20 était bien enregistrée, mais elle revenait ensuite à 10 après redémarrage.

---

### Restaurer manuellement l'état ALSA

```bash
sudo alsactl restore 0
```

Cette commande recharge manuellement l'état ALSA précédemment sauvegardé.

Lors des essais, elle restaurait correctement le niveau `Headphone` à 20.

Cela a permis de vérifier que la sauvegarde ALSA elle-même était correcte.

---

### Vérifier l'influence de WirePlumber

```bash
systemctl --user restart wireplumber
```

Cette commande redémarre **WirePlumber**, le gestionnaire de session utilisé avec PipeWire.

Pendant les tests, son redémarrage faisait revenir le niveau `Headphone` de **20 à 10**.

Cela a montré que le changement de volume après démarrage ne provenait pas d'un échec de `alsactl store`, mais d'une réinitialisation ultérieure du chemin audio par la couche PipeWire/WirePlumber/UCM.

---

### Examiner les commutateurs Headphone

```bash
amixer -c 0 get 'Headphone Left'
amixer -c 0 get 'Headphone Right'
```

Ces commandes permettent de vérifier séparément si les chemins analogiques gauche et droit de la sortie Headphones sont activés.

Pendant les essais, les deux étaient sur `on`.

---

### Examiner les commutateurs Receiver utilisés par Line Out

```bash
amixer -c 0 get 'Receiver Left'
amixer -c 0 get 'Receiver Right'
```

Ces commandes ont permis de constater que les deux contrôles `Receiver` étaient activés dans ALSA.

Ce résultat, pris isolément, pouvait faire penser que Line Out devait être stéréo. La documentation du Media Carrier a ensuite clarifié que sa sortie physique Line Out est en réalité un **canal mono différentiel**.

---

### Examiner le mode Line Out

```bash
amixer -c 0 cget name='LINMOD Mux'
```

Le contrôle retournait notamment les choix :

```text
Item #0 'Left Only'
Item #1 'Left and Right'
```

avec :

```text
values=1
```

Ce contrôle faisait partie des éléments étudiés pendant la recherche de la cause. Il décrit un réglage interne du chemin audio ; il ne suffit pas, à lui seul, à déterminer la nature électrique du jack Line Out.

La documentation matérielle reste déterminante : le jack expose `LINEOUT_P / LINEOUT_M` comme une **paire différentielle mono**.

---

### Vérifier les niveaux des mixers Receiver

```bash
amixer -c 0 get 'Receiver Left Mixer'
amixer -c 0 get 'Receiver Right Mixer'
```

Ces commandes ont servi à rechercher une éventuelle différence de gain entre les deux chemins internes.

Pendant les essais, les deux étaient réglés au même niveau :

```text
2 [67%] [-6.00dB]
```

Aucune asymétrie évidente n'a donc été trouvée à ce niveau.

---

### Ce que ces commandes ont permis d'établir

Les commandes seules ne permettaient pas d'expliquer complètement le comportement de Line Out. Elles ont cependant permis de vérifier que :

- le flux envoyé à `hw:0,0` pouvait contenir deux canaux ;
- Headphones reproduisait correctement Left et Right ;
- Line Out ne reproduisait que Front Right dans notre configuration de test ;
- le problème n'était pas simplement dû à un contrôle Left désactivé ou à un niveau différent ;
- le réglage de volume Headphones était réappliqué par la pile audio après la restauration ALSA.

C'est finalement la consultation du **UNO Media Carrier User Manual**, combinée aux essais, qui a permis d'interpréter correctement le Line Out comme une sortie **mono différentielle** et Ear Out comme une sortie différentielle du **canal droit**.


### Rendre le volume Headphones permanent après redémarrage

Un point supplémentaire concernant la sortie audio du Media Carrier : j'ai trouvé une méthode permettant de conserver mon niveau de volume préféré pour la sortie casque après un redémarrage.

Sur ma VENTUNO Q, le contrôle ALSA `Headphone` utilise une plage de 0 à 31.

La valeur par défaut après le démarrage était :

```text
Headphone = 10 / 31
```

Pour mon casque et mes enceintes PC amplifiées, je préfère :

```text
Headphone = 20 / 31
```

Cette valeur peut être réglée manuellement avec :

```bash
amixer -c 0 set Headphone 20
```

Cependant, cette valeur n'était pas conservée après un redémarrage.

#### Pourquoi le volume revenait à 10

J'ai d'abord essayé de sauvegarder l'état ALSA avec :

```bash
sudo alsactl store 0
```

L'état sauvegardé contenait bien :

```text
name 'Headphone Volume'
value.0 20
value.1 20
```

et une restauration manuelle avec :

```bash
sudo alsactl restore 0
```

rétablissait correctement le volume à 20.

Cependant, après un redémarrage, le volume revenait à 10.

La raison se trouve dans la configuration UCM utilisée par le codec MAX98090.

La configuration UCM de la VENTUNO Q :

```text
/usr/share/alsa/ucm2/Qualcomm/qcs8300/monaco-gertrude/HiFi.conf
```

inclut :

```text
/codecs/max98090/EnableSeq.conf
```

et ce fichier contient :

```text
cset "name='Headphone Volume' 10"
cset "name='Speaker Volume' 10"
```

Ainsi, lorsque WirePlumber initialise le périphérique audio et active la configuration UCM, le volume de la sortie casque est initialisé à 10.

Je l'ai confirmé expérimentalement : le redémarrage de WirePlumber faisait revenir la valeur ALSA `Headphone` de 20 à 10.

#### Ma solution

Plutôt que de modifier les fichiers UCM du système situés dans `/usr/share/alsa/ucm2/`, j'ai créé un petit service systemd utilisateur qui applique la valeur souhaitée après que WirePlumber a initialisé le système audio.

Créer le répertoire des services utilisateur si nécessaire :

```bash
mkdir -p ~/.config/systemd/user
```

Puis créer :

```text
~/.config/systemd/user/ventuno-headphone-volume.service
```

avec le contenu suivant :

```ini
[Unit]
Description=Set VENTUNO Q headphone volume
After=wireplumber.service
Requires=wireplumber.service

[Service]
Type=oneshot
ExecStartPre=/usr/bin/sleep 5
ExecStart=/usr/bin/amixer -c 0 set Headphone 20
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

Le délai de **5 secondes** est nécessaire sur mon système car `wireplumber.service` utilise `Type=simple`. systemd considère donc WirePlumber comme démarré avant que celui-ci ait terminé l'initialisation du périphérique audio ALSA/UCM.

Sans ce délai, mon service réglait bien le volume à 20, mais l'initialisation UCM effectuée ensuite le ramenait à 10.

J'ai ensuite rechargé la configuration systemd utilisateur :

```bash
systemctl --user daemon-reload
```

puis activé le service :

```bash
systemctl --user enable ventuno-headphone-volume.service
```

Après avoir redémarré la VENTUNO Q, j'ai vérifié avec :

```bash
amixer -c 0 get Headphone
```

et j'obtiens maintenant :

```text
Front Left: 20 [65%] [-7.00dB] Playback [on]
Front Right: 20 [65%] [-7.00dB] Playback [on]
```

Le réglage est donc maintenant automatiquement rétabli à la valeur souhaitée après chaque connexion/redémarrage, sans modifier les fichiers UCM fournis par le système.

---

## Bureautique – LibreOffice Writer et Calc

La VENTUNO Q peut également être utilisée comme un ordinateur de bureau classique.

Pour un usage bureautique léger, il n'est pas nécessaire d'installer l'intégralité de la suite LibreOffice. Il est possible d'installer uniquement :

- **LibreOffice Writer** : traitement de texte
- **LibreOffice Calc** : tableur
- **Interface française de LibreOffice**

### Installation

Mettre à jour la liste des paquets :

```bash
sudo apt update
```

Installer Writer, Calc et la localisation française :

```bash
sudo apt install libreoffice-writer libreoffice-calc libreoffice-l10n-fr
```

Dans mon cas, APT indique :

```text
Après cette opération, 357 Mo d'espace disque supplémentaires seront utilisés.
```

L'installation reste donc très légère par rapport aux 64 Go d'eMMC de la VENTUNO Q.

### Lancement depuis le terminal

Writer :

```bash
libreoffice --writer
```

Calc :

```bash
libreoffice --calc
```

Les deux applications sont également disponibles directement depuis le menu des applications d'Ubuntu.

### Résultat du test

Sur ma VENTUNO Q 16 Go / 64 Go, **LibreOffice Writer et Calc fonctionnent parfaitement et rapidement**.

Après installation :

```text
Sys. de fichiers Taille Utilisé Dispo Uti% Monté sur
/dev/mmcblk0p71     55G     20G   33G  38% /
```

L'installation de Writer et Calc occupe environ **357 Mo supplémentaires**. La variation visible avec `df -h` est arrondie et ne permet donc pas de mesurer précisément l'espace réellement consommé.

Ce premier test confirme que la VENTUNO Q peut être utilisée pour des tâches bureautiques classiques, tout en conservant ses possibilités de développement avec le MPU et le MCU.

---

Ubuntu 26.04.1 LTS est proposé automatiquement par le gestionnaire de mises à jour, mais mise à niveau différée dans l'attente d'une confirmation de compatibilité VENTUNO Q.

---

------------------------------------------------------------------------
*D'autres notes sur la VENTUNO Q pourront être ajoutées à ce dépôt  au fil
des essais.*
------------------------------------------------------------------------

