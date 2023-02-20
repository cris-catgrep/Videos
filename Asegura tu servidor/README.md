# Asegura tu servidor

- Comandos utilizados en el video:https://youtu.be/_Hcm9ksOkqU
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

Podemos crear reglas sencillas especificando si será para permitir(**allow**) o denegar (**deny**) y al final el servicio o puerto y protocolo.
```bash
sudo ufw allow ssh
sudo ufw allow 9999
sudo ufw allow 9999/tcp
sudo ufw deny http
sudo ufw deny https comment 'comentario'
```
Tenemos también las reglas por defecto.
```bash
sudo ufw default deny incoming comment 'deniega el tráfico entrante'
sudo ufw default deny outgoing comment 'deniega el tráfico que sale'
```
Tras agregar las reglas deberemos de habilitarlas con el comando.
```bash
sudo ufw enable
```
Podemos ver las reglas activas con:
```bash
sudo ufw status
```
### UFW y docker
Para que funcione junto con docker deberemos de editar el archivo **after.rules**.
```bash
sudo nvim /etc/ufw/after.rules
```
Y agregar el siguiente bloque de configuración al final der archivo antes de **COMMIT**.
```bash
# BEGIN UFW AND DOCKER```
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

# END UFW AND DOCKER
```
Ahora reniciaremos UFW.
```bash
sudo ufw reload
```
Una vez hecho esto podremos acceder a nuestros servicios de docker desde nuestra red local, pero el resto de internet no podrá. Si quieres hacer público uno de estos servicios deberás de utilizar el siguiente comando:
```bash
ufw route allow proto tcp from any to any port 80
```
