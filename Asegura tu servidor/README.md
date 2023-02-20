# Asegura tu servidor

- Comandos utilizados en el video:
- Estos comandos son para servidores con Ubuntu y derivadas.

## Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade
```
## Crear nuevo usuario y bloquear pass de root

Crear y añadir al grupo sudo el nuevo usuario.
```bash
sudo adduser usuario
sudo usermod -aG sudo usuario 
```
Ahora deberás de cerrar la sesión e iniciar una nueva con el nuevo usuario para ejecutar un comando como root.
```bash
sudo apt update
```
Bloquear el pass de root.
```bash
sudo passwd -l root
```

## SSH
### Crear keys
Desde nuestra computadora personal crearemos las llaves SSH y las copiaremos a nuestro server.
```bash
ssh-keygen -t ed25519
ssh-copy-id user@server
```
### Configuraciones de seguridad
Crearemos un respaldo del archivo **sshd_config** y lo editaremos.
```bash
cd /etc/ssh/
sudo cp sshd_config sshd_config_back
sudo vi sshd_config
```
Para descomentar y modificar los siguientes parámetros:
- **PaswordAuthentication** permite utilizar SSH con el password de usuario, descomentar y cambiar a **no** una vez se tengan las keys.
- **PermitRootLogin** permite conectarse por SSH como root, descomentar y cambiar a **no**.
- **Port** especifica el puerto que utilizará SSH, por defecto es el 22, modificarlo es opcional pero recomendado. **IMPORTANTE** si se modifica el puerto este se deberá de permitir en el firewall.

Se deberá de guardar el archivo y reiniciar SSH.
```bash
sudo service sshd restart
```

En caso de cambiar el puerto se utilizará el siguiente comando para conectarse con el nuevo puerto.
```bash
ssh -p 9999 user@server
```

## Actualizaciones automáticas
Instalar **Unattended-upgrades**.
```bash
sudo apt install unattended-upgrades
```
Para definir que actualizaciones hacer se deberá de editar el archivo **50unattended-upgrades**.
```bash
cd /etc/apt/apt.conf.d/
sudo vi 50unattended-upgrades
```
A continuación se deberá de editar el archivo.
```bash
sudo vi 20auto-upgrades
```
Y agregar el siguiente bloque de texto:
```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```
Finalmente podemos probar nuestra configuración con:
```bash
sudo unattended-upgrades --dry-run --debug
```

## Fail2ban
Durante la instalación se nos pedirá reiniciar algunos servicios, solo presionaremos enter y continuaremos.
```bash
sudo apt install fail2ban
```
Lo iniciaremos con el comando:
```bash
sudo systemctl enable fail2ban --now
```
Para ver el status de fail2ban y alguna jail específica.
```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
Para activar una cárcel preconfigurada crearemos una copia de **jail.conf** y la editaremos.
```bash
cd /etc/fail2ban/
sudo cp jail.conf jail.local
sudo vi jail.local
```
Para activar la cárcel agregaremos el siguiente parámetro en su respectivo bloque de texto.
```bash
enabled = true
```
Por último reiniciaremos fail2ban.
```bash
sudo fail2ban-client reload
```

## UFW
