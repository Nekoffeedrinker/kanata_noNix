# Mis configs de Kanata en formato dotfile

En los archivos declaro las siguientes teclas
  - caps_lock -> Esc + Ctrl

Hago este repo para instalar Kanata en Windows y en Linux FHS, donde no puedo
usar configuraciones de NixOS. A continuación expongo los pasos para instalar
esta configuración.

## Clonar el repo

```bash
git clone https://github.com/Nekoffeedrinker/kanata_noNix.git ~/.config/kanata
```

## Usar en Linux

### Instalar Kanata en Linux

Busca la última version _release_ (que no es "prerelease"), despliega las
intstrucciones para Linux y descarga el archivo `linux-binaries-x64.zip`.

Extrae el contenido del zip, navega dentro de la carpeta (si lo extrajo en una
carpeta) y enfócate en el archivo `kanata_linux_x64`. El otro archivo (el que
lleva 'cmd_allowed') no lo usaremos, lo puedes borrar.

Abre una terminal en la ubicación (o abrela y navega hasta ahí) y ejecuta los
siguientes comandos para cambiarle el nombre, darle permisos de ejcución y mover
el binario a `/usr/local/bin/`.

```bash
mv kanata_linux_x64 kanata
chmod +x kanata 
sudo mv kanata /usr/local/bin/
```

**Importante:** El nombre de este binario será el nombre del comando de linux.

Si hiciste todo correctamente, deberás poder correr `kanata --version`.


### Hacer kanata ejecutable sin sudo

Ahora mismo puedes correr la configuración de este repo y deberá funcionar, pero
requiere ejecutarla como sudo:

```bash
sudo kanata -c ~/.config/kanata/config.kbd
```

Debido a que vamos a configurar el autoarranque como servicio de usuario,
necesitamos que kanata se pueda ejecutar sin permisos de super usuario. Para
ello vamos a seguir los siguiente pasos.


1. Crear el grupo uinput (si no existe) y agregar tu usuario:

```bash
sudo groupadd uinput
sudo usermod -aG uinput $USER
```

2. Crear una regla udev para que el grupo tenga acceso al dispositivo:

```bash
echo 'KERNEL=="uinput", GROUP="uinput", MODE="0660"' | sudo tee /etc/udev/rules.d/99-uinput.rules
```

3. Recargar las reglas udev:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

4. Cerrar sesión y volver a iniciarla (o reiniciar), porque el cambio de grupo
   no se aplica hasta que vuelvas a loguearte.

Después de reiniciar sesión, deberás de poder ejecutar sin sudo:

```bash
kanata -c ~/.config/kanata/config.kbd
```


### Incluir nuestro usuario en el grupo input

Ademáß de estar en el gupo `uinput` debemos estar en `input` para que kanata
detecte nuestro teclado cuando es lanzado como servicio de sistemd.

```bash
sudo usermod -aG input $USER
```


### Crear el servicio de autoarranque

Hay que crear el archivo del servicio:

```bash
mkdir -p ~/.config/systemd/user
nvim ~/.config/systemd/user/kanata.service
```

y dentro de este pegar lo siguiente:

```ini
[Unit]
Description=Kanata keyboard remapper
Documentation=https://github.com/jtroo/kanata

[Service]
Environment=PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/bin
Type=simple
ExecStart=/usr/local/bin/kanata --cfg %h/.config/kanata/config.kbd
Restart=on-failure

[Install]
WantedBy=default.target
```

Luego activamos y arrancamos el servicio:

```bash
systemctl --user daemon-reload
systemctl --user enable kanata.service
systemctl --user start kanata.service
```

y verificamos que esté corriendo

```bash
systemctl --user status kanata.service
```

Con esto, el servicio de kanata se ejecutará cada que iniciemos sesión.

### Cosas útiles

Si modificamos el archivo de configuración, hay que reiniciar el servicio para aplicar los cambios en Kanata.

```bash
systemctl --user restart kanata.service
```