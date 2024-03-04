# Qtile Desktop

## Instalación
echo x11-libs/cairo X glib opengl svg >> /etc/portage/package.use/qtile

emerge --ask x11-wm/qtile

## Qtile como compositor de Wayland

emerge --ask dev-python/pywlroots

### Soporte para aplicaciones X11 (XWayland)
emerge --ask x11-base/xwayland

## Personalización

python3 -m py_compile ~/.config/qtile/config.py

paquetes

transparncia
    
    picon

Fondonde de patalla tambie ver imagenes
    feh

Menu de aplicaciones
    rofi

terminal
    x11-terms/alacritty

session
    lightdm

para abir termial (default es, coltol + alt + f2 o f3 )
    MOTH (Windows) + enter (se debe configural),

Configurar teclado
x11-apps/setxkbmap es (español)

Ingles US Variante Intenacional
setxkbmap -layout us -variant intl

Compositor
    xcompmgr

ventas
    abjo en los numero abajo cambia, y moth + tab




"fch --bg-scale /home/freddy/fondo.jpg",
    "xcompmgr &",
    "nm-applet &",


## COMANDOS
- Recargar la configuracion
    
    mod + ctrl r

- Abrir Terminal

    mod + enter

- Ver las terminales en multiples ventanas

    mod + tab

    


cmd = [
    "setxkbmap -layout us -variant intl",
    "nm-applet &",
    "feh --bg-scale /home/$USER/Imágenes/fondo.jpg",
    "picom &"
]

for x in cmd:
    os.system(x)
