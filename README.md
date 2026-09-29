
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

------------------------------------------------------------------------

*D'autres notes sur la VENTUNO Q pourront être ajoutées à ce dépôt au fil des essais.*

*D'autres notes sur la VENTUNO Q pourront être ajoutées à ce dépôt  au fil
des essais.*
