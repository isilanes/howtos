Podemos instalar varios SOs en una sola partición Btrfs. Básicamente vamos a hacer un subvolumen para HOME, y uno más para cada uno de los SOs que queramos instalar. He seguido los pasos detallados en [1], aunque cambiando algunas cosas.

# Primera instalación

He instalado Linux Mint 20.1. Arrancamos con el pendrive del instalador, y hacemos 3 particiones:

* 4 GB para swap.
* 4 GB vacíos (por si acaso), y dentro una partición de 1 GB para EFI.
* Resto para una partición Btrfs.

Hecho esto, instalamos normal, usando la tercera partición para /, y diciendo que la formatee. Esto hará que nos genere dos subvolúmenes, `@` y `@home`, para `/` y `/home`, respectivamente.

```bash
$ btrfs subvolume list /
```

Arrancaremos en el SO recién instalado y haremos lo siguiente: dejaremos `@home` tal cual, y usaremos un nuevo subvolumen para `/`.

## Snapshot de @

Supongamos que la partición Btrfs es `/dev/p3`. Empezaremos por hacer un snapshot de `@`, para a continuación vaciarlo:

```bash
$ mkdir /mnt/btrfs
$ mount -t btrfs -o subvolid=5 /dev/p3 /mnt/btrfs/
$ btrfs subvolume snapshot /mnt/btrfs/@ /mnt/btrfs/@root-mint
$ rm -rf /mnt/btrfs/@/  # will delete everything but directory itself
```

## Hacer que LM arranque de @root-mint

En primer lugar editamos el fichero `/boot/efi/EFI/ubuntu/grub.cfg`, y en la segunda línea cambiamos `@` por `@root-mint`.

Luego editamos `/mnt/btrfs/@root-mint/etc/fstab`, y dejamos un par de líneas como las siguientes:

```bash
UUID=<valor-de-uuid>  /      btrfs   subvolid=<subvolid-de-@root-mint>,defaults,noatime  0  1
UUID=<valor-de-uuid>  /home  btrfs   subvolid=<subvolid-de-@home>,defaults,noatime       0  1
```

En este punto también dejaremos un fichero en el directorio que será `/`, para chivarnos que es el subvolumen correcto:

```bash
$ touch /mnt/btrfs/@root-mint/root-mint.txt
```

## Primer arranque con nueva raíz

Una vez hecho lo precedente, reiniciaremos el PC. En el menú de grub presionaremos "e" tras seleccionar la entrada deseada, para editarla. En el texto mostrado cambiaremos `@` por `@root-mint`. Esto tendremos que hacerlo sólo esta vez. Tras hacer ese cambio podemos reanudar el arranque con F10.

Una vez arrancado el sistema nos aseguraremos de que hemos arrancado con la raíz correcta. Esto lo veremos haciendo `ls /`, ya que ahí debería aparecernos el fichero `root-mint.txt` que creamos antes. Además podemos ver explícitamente el subvolumen con:

```bash
$ btrfs subvolume show /
```

Esto debería mostrarnos en la primera línea `Name: @root-mint`.

Si ambos chequeos son exitosos, hemos arrancado con `/` en el subvolumen que toca. En ese caso, actualizaremos grub:

```bash
$ update-grub2
```

Una vez hecho esto, deberíamos poder reiniciar y, sin tocar nada en el menú grub, entrar directamente al subvolumen (es decir, al SO), deseado.

# Siguientes instalaciones

Se pueden hacer todas las instalaciones subsecuentes de sistemas operativos que se quieran, repitiendo casi todos los pasos explicados con la primera instalacíón. A saber:

1. Instalar desde el pendrive de instalación de manera normal, teniendo en cuenta lo siguiente:
    * No haremos ninguna partición nueva.
    * Reusaremos la partición swap.
    * Reusaremos la partición EFI **sin formatearla**.
    * Reusaremos la partición Btrfs **sin formatearla**.
2. Una vez instalado el sistema, se nos habrá sobreescrito `@` (recordemos que lo dejamos vacío), y se habrá reusado `@home` (dependiendo de qué SO instalemos, quizá se generen otros subvolúmenes).
3. Arrancaremos nuevamente, y deberíamos entrar al nuevo SO, sin hacer nada.
4. Dentro del nuevo SO, repetiremos el proceso de:
    * hacer snapshot de `@` a `@root-<lo-que-sea>`.
    * borrar el contenido de `@`.
    * editar `/boot/efi/EFI/ubuntu/grub.cfg`.
    * editar `/mnt/btrfs/@root-<lo-que-sea>/etc/fstab` como corresponda.
    * dejar un fichero `root-<lo-que-sea>.txt` en `/mnt/btrfs/@root-<lo-que-sea>/`, como chivato. 
5. Reiniciaremos y editaremos la entrada del menú grub con la tecla "e".
6. Una vez arrancado:
   * chequear que estamos en `@root-<lo-que-sea>`.
   * ejecutaremos `update-grub2`.
7. Deberíamos reiniciar una vez más, y ver que entramos al nuevo SO, sin hacer nada.

# Cambiar de un SO a otro SO

Cada vez que queramos arrancar en un SO diferente tendremos que editar el fichero `/boot/efi/EFI/ubuntu/grub.cfg`. Esto lo podremos hacer con el sistema arrancado y funcionando, o arrancando con un pendrive, por ejemplo. En ese fichero cambiaremos `@root-<uno>` a `@root-<otro>`, para dejar de arrancar en `<uno>` y empezar a arrancar en `<otro>`. Este paso es un poco engorroso, pero al menos sólo hay que hacerlo la primera vez que queramos arrancar en `<otro>`.

# Snapshots de / y volver atrás en el tiempo

[TODO]

# Referencias

[1] https://www.jujens.eu/posts/en/2020/Feb/23/opensuse-install-btrfs-subvolumes/
