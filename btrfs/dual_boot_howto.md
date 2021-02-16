Podemos instalar varios SOs en una sola partición Btrfs. Básicamente vamos a hacer un subvolumen para HOME, y uno más para cada uno de los SOs que queramos instalar. He seguido los pasos detallados en [1], aunque cambiando algunas cosas.

# Primera instalación

He instalado Linux Mint 20.1. Arrancamos con el pendrive del instalador, y hacemos 3 particiones:

* 4 GB para swap.
* 4 GB vacíos (por si acaso), y dentro una partición de 1 GB para EFI.
* Resto para una partición Btrfs.

Hecho esto, instalamos normal, usando la tercera partición para /, y diciendo que la formatee. Esto hará que nos genere dos subvolúmenes, @ y @home, para / y /home, respectivamente.

```bash
$ btrfs subvolume list /
```

Arrancaremos en el SO recién instalado y haremos lo siguiente:

* @home se queda para HOME, no hace falta tocar nada.
* Hacemos un snapshot de @ a @root-mint
* Vaciamos @
* Hacemos que LM arranque de @root-mint

## Hacer snapshot

Supongamos que la partición Btrfs es `/dev/p3`.

```bash
$ mkdir /mnt/btrfs
$ mount -t btrfs -o subvolid=5 /dev/p3 /mnt/btrfs/
$ btrfs subvolume snapshot /mnt/btrfs/@ /mnt/btrfs/@root-mint
```

## Vaciar @

```bash
$ rm -rf /mnt/btrfs/@/  # will delete everything but directory itself
```

## Hacer que LM arranque de @root-mint

En primer lugar editamos el fichero `/boot/efi/EFI/ubuntu/grub.cfg`, y en la segunda línea cambiamos "@" por "@root-mint".

Luego editamos `/mnt/btrfs/@root-mint/etc/fstab`, y dejamos un par de líneas como las siguientes:

```bash
UUID=<valor-de-uuid>  /      btrfs   subvolid=<subvolid-de-@root-mint>,defaults,noatime  0  1
UUID=<valor-de-uuid>  /home  btrfs   subvolid=<subvolid-de-@home>,defaults,noatime       0  1
```


# Siguientes instalaciones


# Referencias

[1] https://www.jujens.eu/posts/en/2020/Feb/23/opensuse-install-btrfs-subvolumes/
