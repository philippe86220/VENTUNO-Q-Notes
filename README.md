
# VENTUNO Q --- Notes et tests

Notes personnelles concernant mes premiers essais avec la **Arduino
VENTUNO Q**.

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

*D'autres notes sur la VENTUNO Q pourront être ajoutées à ce Gist au fil
des essais.*
