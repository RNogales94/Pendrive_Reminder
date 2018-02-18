# Pendrive_Reminder
Pequeña aplicación para no olvidar el pendrive, para GNU/Linux

## Introducción
Esta aplicación tiene como intención hacer de recordatorio a la hora de usar tu pendrive en un PC ajeno.
El funcionamiento es muy simple: si intentas apagar el ordenador con el pendrive conectado, la aplicación te mandará un aviso, y te bloqueará el apagado hasta que desconectes el pendrive

## Requisitos
- Sistema Operativo GNU/Linux
- Polkit
- Udev
- libnotify

## Implementación

### Udev
Para nuestra aplicación, se han usado dos reglas udev, asociadas a los eventos de conexión y desconexión de un dispositivo USB.

- La primera regla udev, [10-usbmount.rules](https://github.com/AlmuHS/Pendrive_Reminder/blob/master/udev-rules/10-usbmount.rules), al detectar el evento de conexión de un dispositivo de almacenamiento USB, invocará al script [usbdevinserted.sh](https://github.com/AlmuHS/Pendrive_Reminder/blob/master/aux_scripts/usbdevinserted.sh). Este script escribirá el identificador del USB en un fichero, creándolo en caso de no existir.
Este fichero nos servirá de testigo para saber si queda algún dispositivo de almacenamiento USB conectado en el sistema.

	Como identificador usaremos la variable `DEVPATH` asociada al dispositivo, y el fichero generado estará ubicado en `/tmp/usbdevinfo`


- La segunda regla udev,[11-usbdisconnect.rules](https://github.com/AlmuHS/Pendrive_Reminder/blob/master/udev-rules/11-usbdisconnect.rules), detectará el evento de desconexión del dispositivo USB, e invocará al script [usbdevgone.sh](https://github.com/AlmuHS/Pendrive_Reminder/blob/master/aux_scripts/usbdevgone.sh). que buscará el identificador del dispositivo en el fichero testigo y, en caso de existir, lo borrará. Una vez borrado el identificador, si el fichero está vacío (no queda ningún dispositivo conectado) borrará el fichero.

	Dadas las diferencias entre distribuciones, esta segunda regla udev se ha asociado a dos eventos distintos: unbind y remove, que representan el evento de desconexión en diferentes distribuciones.

### Polkit
Añadido a las reglas udev, también se han creado dos reglas polkit. Estas servirán para detectar el evento de apagado y denegar la autorización para el mismo, en caso de que haya algún dispositivo de almacenamiento USB conectado.

Debido a las diferencias entre las versiones 0.106 (que admite ficheros .rules en javascript) y las anteriores (que funcionan con ficheros de autorización) se han seguido dos implementaciones para este comportamiento:


- Para las versiones modernas de polkit (>= 0.106), se ha usado un fichero .rules (`10-inhibit-shutdown.rules`) que, al detectar el evento de apagado, invoca a un script (`check_pendrive.sh`) que indica si el fichero testigo existe en el sistema, devolviendo 0 (correcto) en caso de que no exista y 1 (error) en caso de que no exista.

	En caso de error, se deniega el permiso, y se invoca a otro script (`send_notify.sh`) que envía una notificación al usuario, indicando que debe desconectar el pendrive para poder apagar el ordenador.
	
- Para las versiones antiguas de polkit (< 0.106), se ha usado un fichero de autorización .pkla (`50-inhibit-shutdown.pkla`).
		Este fichero será copiado por el script `usbdevinserted.sh` durante el evento de conexión del pendrive. Al copiarlo, se activará el bloqueo del apagado.
		Una vez el pendrive se desconecte, el script `usbdevgone.sh`, en caso de que no quede ningún dispositivo conectado, borrará el fichero de autorización para desactivar el bloqueo.
		
Dadas las diferencias entre distribuciones y/o entornos de escritorio, estas reglas polkit estan asociadas varios eventos distintos: `org.freedesktop.consolekit.system.stop`, `org.freedesktop.login1.power-off`, `org.freedesktop.login1.power-off-multiple-sessions` y `org.xfce.session.xfsm-shutdown-helper` 

## Comportamiento
El comportamiento de la aplicación dependerá de la versión de polkit usada por el sistema.

- Si la versión de polkit es >= 0.106, al conectar el pendrive, la opción de apagado del sistema desaparecerá del entorno de escritorio. Si el usuario pulsa en el gestor de sesiones (botón de apagar), se le enviará una notificación indicando que debe desconectar el pendrive para eliminar el bloqueo (WIP)

- Si la versión es < 0.106, la opción de apagado permanecerá al conectar el pendrive, pero al pulsarla no se realizará ninguna acción.

En ambos casos, el sistema volverá a la normalidad, desbloqueando el apagado, al desconectar el pendrive.

## Instalación

Para instalar la aplicación, únicamente hay que descargar el repositorio y ejecutar el script de instalación.

- Para descargar, se puede usar `git` con el comando:

	`git clone https://github.com/AlmuHS/Pendrive_Reminder.git`
	
- Una vez descargado, para instalarlo hay que ejecutar:

	`cd Pendrive_Reminder`
	
	`sudo ./installer.sh`

### Directorios de instalación

Los scripts se copiarán en el directorio `/usr/bin/pendrive-reminder`. 

La regla polkit se copiará en `/usr/share/polkit-1/rules.d/` en caso de polkit >= 0.106. 
En caso de polkit < 0.106, el fichero .pkla se ubicará temporalmente en `/usr/bin/pendrive-reminder` y, una vez conectado el pendrive, se copiará a `/etc/polkit-1/localauthority/50-local.d/`, de donde se borrará una vez se desconecte el pendrive
