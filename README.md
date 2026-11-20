# Proyecto-Administracion-Sistemas-Informaticos

Proyecto de administración de sistemas informáticos: Mansible
Objetivos
El objetivo de este proyecto práctico de desarrollo en pareja, aunque también puede realizarse de forma individual, es revisar de forma aplicada algunos de los conceptos estudiados en la parte teórica de la asignatura tales como:
La administración de los dispositivos de almacenamiento mediante volúmenes.
El uso de SSH para realizar la administración de equipos remotos.
Los fundamentos de las técnicas usadas para automatizar la administración.
El modo de operación de Ansible, que es la herramienta de gestión de configuración más usada.
La programación de scripts de bash.
Además, si opta por usar Docker como plataforma de desarrollo, podrá familiarizarse con esa herramienta.
NOTA: El tema dedicado a Ansible se presenta un poco más adelante, pero para afrontar la práctica no es necesario un conocimiento profundo de esta herramienta, sino que basta con entender, en términos generales, su modo de operación.
Planteamiento general
El trabajo se centra en la creación de una infraestructura de automatización de la administración similar a Ansible, que es la herramienta de gestión de configuración más popular actualmente y que se estudia en la parte teórica de la asignatura. Se trata de una versión muy simplificada, evidentemente, a la que denominaremos Mansible (por mini Ansible), pero que permite apreciar el fundamento y modo de operación de esta herramienta.
Ansible está organizado como una colección de módulos instalados en la máquina maestra de control que son copiados mediante SCP en las máquinas administradas y ejecutados usando SSH. Con esta herramienta, las operaciones de configuración pueden ejecutarse puntualmente como mandatos ad hoc o, lo habitual, englobarse dentro de un playbook. A continuación, se presenta un ejemplo de cada caso:

Un mandato ad hoc que instala el paquete hello en todas las máquinas gestionadas usando el módulo package:
ansible all -m package -a "name=hello state=present"
Un playbook que crea en todas las máquinas gestionadas un volumen lógico sobre un grupo de volúmenes basado en dos particiones, que también ha creado previamente, lo formatea con el sistema de ficheros ext4 y lo monta en un directorio recién creado:
---
- name: Crea un LV sobre un VG con dos particiones usando un SF ext4 y lo monta
  hosts: all 
  become: true
  tasks:

    - name: Instala parted
      package:
        name: parted
        state: present

    - name: Instala lvm2
      package:
        name: lvm2
        state: present

    - name: Crea primera partición
      community.general.parted:
        device: /dev/nvme1n1
        number: 1
        part_start: 0%
        part_end: 40%
        state: present

    - name: Crea segunda partición
      community.general.parted:
        device: /dev/nvme1n1
        number: 2
        part_start: 40%
        part_end: 100%
        state: present

    - name: Crea el grupo de volúmenes
      community.general.lvg:
          vg: miVG
          pvs: /dev/nvme1n1p1,/dev/nvme1n1p2

    - name: Crea el volumen lógico
      community.general.lvol:
          vg: miVG
          lv:  miLV
          size: 40%VG

    - name: Crea el directorio
      ansible.builtin.file:
        path: /mnt/misDatos
        state: directory
        mode: '0755'

    - name: Formatea con un SF ext4
      community.general.filesystem:
        fstype: ext4
        dev: /dev/miVG/miLV

    - name: Monta el LV sobre el directorio
      ansible.posix.mount:
        path: /mnt/misDatos
        src: /dev/miVG/miLV
        fstype: ext4
        state: mounted
Este playbook contenido en el fichero almacenamiento.yml se ejecutaría con el mandato ansible-playbook:
ansible-playbook almacenamiento.yml
En la práctica se plantea desarrollar una herramienta que siga este modo de operación de Ansible, pero de forma simplificada, lo que implica la implementación de los scripts de bash que corresponden a las tres siguientes fases:
Módulos: se tendrán que implementar tres módulos independientes correspondientes a la gestión de volúmenes físicos PV (módulo pv), de grupos de volúmenes VG (módulo vg) y de volúmenes lógicos LV (módulo lv), con una valoración de 2 puntos por cada módulo.
El mandato que permite la ejecución de operaciones ad hoc, al que denominaremos mansible. Tiene una calificación de 2 puntos y solo se puede evaluar esta fase si ha obtenido en la primera al menos 3,5 puntos.
El mandato que permite la ejecución de playbooks, al que denominaremos mansible-playbook. Tiene una calificación de 2 puntos y solo se puede evaluar esta fase si ha obtenido en la primera al menos 3,5 puntos.
Se trata, por tanto, del desarrollo de cinco scripts de bash tal que los tres primeros corresponden a la primera fase y no requieren su ejecución remota para depurarlos por lo que puede desarrollarlos en un único equipo. Los dos últimos scripts, que corresponden a las dos últimas fases, sí requieren usar al menos dos equipos.
Plataforma de desarrollo
La distribución de Linux usada para preparar la práctica ha sido Ubuntu.
Para el desarrollo de la práctica se presentan varias opciones:

Máquinas reales. El problema con esta opción es que los errores que puedan producirse durante el desarrollo de la práctica pueden dejar el equipo no operativo.
Escritorio virtual Ubuntu de la UPM.
Máquinas virtuales, como, por ejemplo, las proporcionadas por VirtualBox.
Contenedores Docker. Se incluye una sección explicando cómo usar esta opción ya que se considera que puede ser interesante desde el punto de vista didáctico trabajar con esta tecnología.
Cualquiera de las opciones sería adecuada para el desarrollo de la primera fase de la práctica. Sin embargo, en las dos últimas fases, se requiere usar varios equipos, para lo que pueden ser convenientes las dos últimas plataformas planteadas. Nótese que se podrían hacer pruebas en un solo equipo usando esa única máquina tanto como de nodo de control como de nodo configurable utilizando como dirección localhost.
A continuación, se presentan algunas recomendaciones a la hora de configurar la plataforma de desarrollo de la práctica aplicables a cualquiera de los entornos.

Necesidad de dispositivos de almacenamiento
Para poder probar los módulos, el equipo debería disponer de al menos un disco que pueda reutilizarse a discreción.
En caso de no disponer de ninguno libre, una opción aplicable a cualquier plataforma es el uso de dispositivos de tipo loop:

# crea un fichero de 1GiB, que inicialmente no ocupa espacio
$ truncate -s 1GiB disco1
$ sudo losetup -f --show disco1 # le asocia un dispositivo de tipo loop e imprime el nombre del dispositivo creado
/dev/loop17
A partir de ese momento, puede usar ese dispositivo (en el ejemplo, /dev/loop17) en cualquier mandato que espere como parámetro un disco. Nótese que este tipo de "discos" desaparecen al reiniciar el sistema por lo que debería eliminar todos los volúmenes lógicos, grupos de volúmenes y volúmenes fisicos asociados con los mismos antes de apagar el sistema.
Puede usar directamente estos discos o crear particiones en los mismos y utilizar esas particiones.

Para probar el primer módulo se requiere cambiar el tamaño de un dispositivo de almacenamiento previamente creado. En el caso de estar usando directamente uno de estos discos, puede ejecutar los siguientes mandatos:

$ truncate -s +1MiB disco1 # aumenta en 1MiB el tamaño del fichero
$ sudo losetup -c /dev/loop17 # ajusta el dispositivo al nuevo tamaño
En el caso de usar particiones, basta con cambiar el tamaño de la partición.
Una vez completado el uso del dispositivo, puede eliminarse:

$ sudo losetup -f /dev/loop17 # elimina el dispositivo
Uso de sudo sin contraseña
La ejecución de los módulos requiere el uso del mandato sudo, por lo que para facilitar el desarrollo de la práctica, es conveniente poder usar ese mandato sin necesidad de introducir la contraseña.
Para ello, se configurará el fichero /etc/sudoers (mandato sudo visudo /etc/sudoers) para que no requiera contraseña. En Ubuntu habría que realizar las siguientes modificaciones en el fichero:

# Allow members of group sudo to execute any command
#%sudo  ALL=(ALL:ALL) ALL            # comentar esta línea
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL   # y añadir esta
Instalación y configuración del servidor de ssh
Para las dos últimas fases, la práctica requiere usar ssh entre la máquina maestra de control y los equipos que se pretende configurar. En Ubuntu, no viene instalado por defecto el servidor ssh, por lo que es necesario instalar el paquete openssh-server. Dado que los dos últimos scripts se basan en el uso de ssh y, además, no se permite la ejecución interactiva, hay que asegurarse de que ssh está configurado en la máquina de control tal que el acceso a todas las máquinas use un modo de operación sin contraseñas. Para ello, puede ejecutar en la máquina de control la siguiente secuencia (suponiendo que 172.17.0.2 es la dirección IP de la máquina a la que se quiere acceder con ssh sin contraseña):
# genera las claves dejando vacía la passphrase
ssh-keygen
# copiar las claves a la máquina remota usando todavía la contraseña
ssh-copy-id 172.17.0.2
Uso de Docker
El objetivo de esta sección es explicar cómo se puede usar Docker como infraestructura para el desarrollo de la práctica, en caso de que haya optado por esta alternativa, lo que conlleva varias ventajas:
Proporciona un entorno virtual ágil que, ante cualquier incidencia que deje corrupto el estado de la máquina, permite crear rápidamente una nueva instancia.
Posibilita crear de forma eficiente una red de máquinas lo que permite probar las dos últimas fases de la práctica sin necesidad de tener varias máquinas físicas o virtuales.
La práctica se corregirá en un entorno basado en contenedores.
Sin embargo, hay que resaltar que la gestión de volúmenes en Linux no está contenedorizada: un volumen físico, un grupo de volúmenes o un volumen lógico creado en un contenedor es visible en el anfitrión, lo que causará algunas complicaciones, como se irá viendo a lo largo del documento. En cualquier caso, a pesar de que precisamente esta funcionalidad de gestión de volúmenes no sea la más adecuada para apreciar la potencia de los contenedores debido a esas complicaciones, seguimos considerando que merece la pena usar Docker, desde un punto didáctico. Nótese que, aunque presente ciertas dificultades a la hora de desarrollar la primera fase de la práctica, proporciona muchas facilidades para las otras dos fases.

Creación de la imagen Docker
Se proporciona un Dockerfile que permite crear una imagen adaptada a los requisitos de la práctica:
Está basado en Ubuntu 24.04. En caso de que en su equipo esté instalada otra versión de Ubuntu, debe modificar el fichero Dockerfile para que se corresponda con dicha versión.
Tiene instalado el paquete que corresponde a la funcionalidad servidor de SSH.
Incluye el paquete que corresponde al editor nano, para poder disponer de un editor dentro del contenedor.
Dispone del paquete que habilita la funcionalidad de SUDO.
Crea un usuario con el nombre especificado a la hora de construir la imagen y con ese mismo nombre como contraseña.
Configura el nuevo usuario para que pueda usar SUDO sin contraseña.
Establece una disposición de manera que al arrancar un contenedor de esa imagen se active el servicio SSH.
Especifica que el contenedor ejecutará en el contexto del nuevo usuario tal que su directorio base sea el directorio de trabajo inicial.
Establece que al activarse la imagen en un contenedor se arrancará un proceso que ejecuta bash.
El primer paso es, evidentemente, instalar Docker.
Una vez instalado podemos crear la imagen, a la que denominaremos ASI/image, para lo que ejecutaremos este mandato en el directorio donde reside el Dockerfile especificando en el argumento USUARIO cuál será el nombre del usuario en cuyo contexto se ejecutará un contenedor creado con esta imagen:

docker build --build-arg USUARIO=$USER -t ASI/image .
De cara a la práctica, queremos que el usuario sea el mismo que el que está construyendo la imagen.
Finalizada la generación de la imagen, podemos comprobar que se ha creado:

$ docker image ls ASI/image
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
ASI/image    latest    4743b4da2af8   15 hours ago   276MB
Creación de la infraestructura con Docker
Para la primera fase de la práctica, bastaría con crear un contenedor y trabajar dentro del mismo, lo que asegura que el equipo no sufre ningún daño. Para ello, el contenedor compartirá un directorio con el anfitrión, concretamente, el directorio de la práctica será visible en el directorio /ASI del contenedor, lo que permite cambiar el código de los scripts desde el anfitrión, sin necesidad de volver a crear el contenedor. Dado que el usuario en el contenedor puede tener un UID y GID distinto al del asignado en el anfitrión, los scripts deberían tener permiso de lectura y ejecución para todos los usuarios.
Como se ha explicado previamente, se plantea crear dispositivos de tipo loop, acción que se llevaría a cabo en el equipo anfitrión, que se harían visibles al contenedor mediante el argumento device.

Por tanto, suponiendo que se va a usar directamente un dispositivo de tipo loop (recuerde que también se pueden crear particiones y pasárselas del mismo modo al contenedor), el mandato de arranque de un contenedor denominado equipo1 basado en la imagen construida sería:

truncate -s 1GiB disco1 # crea 1 fichero vacío de 1GiB
export DISCO1=$(sudo losetup -f --show disco1) # crea dispo. de bloques de tipo loop
# mandato de arranque del contenedor
docker run --rm -it --name equipo1 -v $PWD:/ASI -e DISCO1 --device $DISCO1 ASI/image
Revisemos los argumentos del mandato que crea un contenedor de nombre equipo1:
--rm: el contenedor desaparece (se elimina el "equipo") cuando termina.
-it: modo de operación interactivo con un terminal asociado.
--name: nombre del contenedor.
-v $PWD:/ASI: deja accesible el directorio actual, que contiene los módulos, a través del directorio /ASI del contenedor.
-e: exporta esa variable de entorno al contenedor.
--device: da visibilidad de ese disco dentro del contenedor.
ASI/image: imagen que se instancia.
Ese es el mandato que se usaría para las pruebas del primer módulo, que necesita un solo dispositivo.

Las pruebas planteadas para el segundo requieren tres dispositivos. Nótese que se usará un argumento -e y uno --device por cada disco que se le quiera pasar al contenedor.

truncate -s 1GiB disco1 disco2 disco3 # crea 3 ficheros vacíos de 1GiB
export DISCO1=$(sudo losetup -f --show disco1) # crea dispo. de bloques de tipo loop
export DISCO2=$(sudo losetup -f --show disco2) # crea dispo. de bloques de tipo loop
export DISCO3=$(sudo losetup -f --show disco3) # crea dispo. de bloques de tipo loop
docker run --rm -it --name equipo1 -v $PWD:/ASI -e DISCO1 --device $DISCO1 -e DISCO2 --device $DISCO2 -e DISCO3 --device $DISCO3 ASI/image
En el desarrollo del tercer módulo es donde nos vamos a encontrar con las complicaciones anticipadas previamente. Para poder crear volúmenes lógicos dentro de un contenedor es necesario instanciarlo como privilegiado (--privileged) y tal que se le habilita acceso a todo el directorio de dispositivos (-v /dev:/dev), rompiendo, por tanto, el aislamiento que proporcionan los contenedores, que es el factor clave que nos había hecho elegir Docker a la hora de desarrollar la práctica sin peligrar la integridad del equipo:
docker run --rm -it --privileged --name equipo1 -v $PWD:/ASI -v /dev:/dev -e DISCO1 --device $DISCO1 ASI/image
Para este tercer módulo, además de instanciar un contenedor privilegiado con total visibilidad del directorio de dispositivos, es necesario configurar LVM2 una vez instalado en el contenedor, añadiendo en el fichero /etc/lvm/lvm.conf las dos siguientes líneas, que desactivan el uso en LVM2 de udev, que no está habilitado en el contenedor (busque esos términos en el fichero y coloque esas líneas descomentadas cerca de donde aparecen):
udev_sync = 0
udev_rules = 0
Por último, para las dos últimas fases, se creará un contenedor convencional por cada una de las máquinas que se pretenden gestionar, ejerciendo el anfitrión el rol de máquina maestra de control. Dado que no se van a usar los módulos desarrollados para probar la práctica, no es necesario usar dispositivos de almacenamiento y, por tanto, los mandatos de activación de cada instancia/"máquina" serían:

docker run --rm -it --name equipo1 -v $PWD:/ASI ASI/image
docker run --rm -it --name equipo2 -v $PWD:/ASI ASI/image
.........................................................
Recuerde que, como se ha explicado previamente, para evitar un comportamiento interactivo de los scripts, es necesario copiar la clave pública del anfitrión en los contenedores (ssh-copy-id; recuerde que en la imagen creada la contraseña es el propio nombre del usuario).

Primera fase: desarrollo de módulos
Como ocurre en Ansible, a la hora de crear un nuevo módulo, es necesario seguir las pautas establecidas en ese entorno en lo que se refiere a cómo recibe los argumentos y cómo genera los resultados. En Mansible, se han establecido las siguientes reglas:
Los módulos reciben sus argumentos con el mismo formato que en Ansible: parejas con el nombre de la variable y su valor separados por un igual (argumento=valor). En el caso de que el argumento sea una lista, los valores se especificarán separados por comas (argumento=valor1,valor2,valor3).
Para facilitar el manejo de ese formato, se proporciona el script prologue que genera por cada argumento una variable cuyo nombre y valor asociado corresponden a los del argumento.
En el caso de una lista, además, se crea una variable adicional con el mismo nombre, pero con el sufijo _split que tendrá como valor todos los elementos de la lista separados por espacios.
En el material de apoyo, además del fichero prologue, se proporciona el script test_prologue que permite apreciar cómo opera prologue.
El módulo terminará con un valor 0 si no ha habido errores y la operación ha realizado algún tipo de actualización, con un valor 128 si la operación ha sido correcta, pero no ha producido ninguna actualización, usándose el resto de los valores para codificar los distintos tipos de error.
Los módulos no deben escribir por la salida estándar. Para facilitar la depuración de la práctica, puede incluir mensajes por la salida de error.
Los módulos no deben ser interactivos: no pueden quedarse a la espera de alguna entrada de teclado. Algunos mandatos que requieren entrada de datos tienen una opción para asumir un valor por defecto. En aquellos mandatos interactivos que no tengan esa opción, se puede inyectarles por la entrada estándar el valor requerido:
# este mandato nos pide "yes" para continuar
mandato <<< "yes"
El modo de operación de un módulo debe seguir la pauta de Ansible: deben ser idempotentes, comprobando todas las condiciones de error previamente y ejecutando de forma atómica.
La funcionalidad de un módulo debe estar centrada en un solo aspecto. Así, por ejemplo, si un módulo requiere la instalación de un paquete, ese módulo no realizará dicha operación, sino que el administrador previamente deberá invocar en una tarea el módulo de instalación de paquetes para llevar a cabo esa labor.
Los módulos planteados tienen, respectivamente, una funcionalidad similar a los siguiente módulos de Ansible: lvm_pv, lvg y lvol, cuya implementación en Python está disponible en GitHub. En cualquier caso, este trabajo plantea una funcionalidad mucho más reducida.

Módulo pv
Se encarga de la gestión de volúmenes físicos (PV) a partir de dispositivos de bloques.
Estos son los argumentos del módulo:

device: nombre del dispositivo de bloques, que es un argumento de carácter obligatorio.
state: el valor present indica que hay que crear el PV mientras que absent especifica que se debe eliminar. El valor por defecto es present.
resize: ajusta el tamaño del PV al del dispositivo subyacente. Tiene como valores true o false, siendo, por defecto, false, y solo es consultado si present y el dispositivo ya es un PV.
force: fuerza la operación de crear o destruir un PV aunque las condiciones no lo aconsejen (por ejemplo, si el PV ya forma parte de un VG). Tiene como valores true o false, siendo, por defecto, false. Esta opción causa que se incluya en el mandato correspondiente el argumento que especifica que se fuerza la realización de la operación.
En cuanto al valor devuelto:
0: se ha completado correctamente realizando cambios: se ha creado o eliminado un PV.
128: se ha completado correctamente, pero sin realizar cambios.
1: error por la ejecución sin ser superusuario (ya incluido en el script inicial).
2: error en los argumentos.
3: error por no estar instalado el paquete requerido.
4: error porque el dispositivo no existe o no es de bloques.
5: error en mandato que crea el PV.
6: error en mandato que elimina el PV.
7: error en mandato que redimensiona el PV.
En primer lugar, el módulo debe verificar los argumentos terminando con un 2 en caso de error. A continuación, debe comprobar que, con independencia del tipo de operación, el paquete que realiza la gestión de volúmenes (lvm2) está instalado, devolviendo un valor 3 en caso contrario. Asimismo, hay que verificar que el dispositivo existe y es de bloques, retornando un 4 en caso contrario.
Una vez superada la fase de comprobación de errores, se presentan tres escenarios:

Crear un nuevo PV, si el estado es present y no existe previamente, tratando adecuadamente la opción force y asegurándose de que el mandato no tiene un modo de operación interactivo.
Redimensionar uno existente, comprobando si el tamaño ha cambiado después del redimensionamiento o no, retornando 0 o 128, respectivamente, y asegurándose de que el mandato no tiene un modo de operación interactivo. Nótese que hay obtener el tamaño antes del redimensionamiento y después para determinar el valor de retorno.
Eliminar un PV existente, tratando adecuadamente la opción force y asegurándose de que el mandato no tiene un modo de operación interactivo.
Nótese que en mi solución he usado los mandatos lblk, pvs, pvcreate, pvresize y pvremove.
NOTA: A lo largo del desarrollo de estos tres módulos, si el sistema de almacenamiento creado se queda en un estado inestable, para intentar recuperarlo, puede usar la solución drástica de borrar cualquier tipo de metadato de los dispositos involucrados:
sudo wipefs -af /dev/loop17
Pruebas
Antes de nada, vamos a crear un disco virtual para evitar que un disco real del equipo pueda verse afectado en caso de error (en el caso de Docker, ejecutaremos este fragmento en el anfitrión y le pasaremos el dispositivo resultante al contenedor, tal como se explicó en la sección dedicada a Docker).
truncate -s 1GiB disco1 # fichero que soporta el disco virtual
export DISCO1=$(sudo losetup -f --show disco1) # crea dispo. bloques tipo loop
A continuación, se especifican algunas pruebas para este módulo que podrán ser ejecutadas en la plataforma que considere oportuno (disponibles en el fichero pruebas_pv del material de apoyo).
$ ./pv  # no SUDO; error 1
$ echo $?
1
$ sudo ./pv state=present # no device; error 2
$ echo $?
2
$ sudo ./pv state=yes device=$DISCO1 # state erróneo; error 2 
$ echo $?
2
$ sudo ./pv state=present device=$DISCO1 resize=yes # resize erróneo; error 2
$ echo $?
2
$ sudo ./pv state=present device=$DISCO1 force=yes # force erróneo; error 2
$ echo $?
2
# asumiendo que LVM2 no está instalado
$ sudo ./pv state=present device=$DISCO1 # lvm2 no instalado; error 3
$ echo $?
3
# instale lvm2

$ sudo ./pv state=present device=noexiste # device no existe; error 4
$ echo $?
4
$ sudo ./pv state=present device=/etc/passwd # device no es un dispositivo de bloques; error 4
$ echo $?
4
$ sudo ./pv state=present device=$DISCO1 # crea el PV; retorna 0
$ echo $?
0
$ sudo pvs $DISCO1
  PV            VG Fmt  Attr PSize   PFree  
  /dev/loop25    lvm2 ---  1.00g 1.00g

$ sudo ./pv state=present device=$DISCO1 # ya creado; retorna 128
$ echo $?
128

# extienda el disco $DISCO1

$ sudo ./pv device=$DISCO1 # retorna 128
$ echo $?
128
$ sudo ./pv device=$DISCO1 resize=false # retorna 128
$ echo $?
128
$ sudo ./pv device=$DISCO1 resize=true # retorna 0
$ echo $?
0
$ sudo ./pv device=$DISCO1 resize=true # retorna 128
$ echo $?
128
$ sudo ./pv state=absent device=$DISCO1 # elimina el PV; retorna 0 
$ echo $?
0
$ sudo ./pv state=absent device=$DISCO1 # ya eliminado el PV; retorna 128 
$ echo $?
128

# crea directamente un grupo de volúmenes para hace la prueba del force
$ sudo vgcreate VG $DISCO1
$ sudo ./pv state=absent device=$DISCO1 # error al elimimarlo; error 6
$ echo $?
6
$ sudo ./pv state=absent device=$DISCO1 force=true # retorna 0
$ echo $?
0
Módulo vg
Se encarga de la creación, reajuste y eliminación de un grupo de volúmenes (VG) construido sobre un conjunto de dispositivos de bloques. Tenga en cuenta que en LVM2 si se crea o extiende un VG, no es necesario que los dispositivos especificados sean PV, ya que el mandato que realiza la creación o extensión convierte automáticamente los dispositivos de bloques en PV. Como consecuencia, el desarrollo de este módulo no depende del módulo pv anterior, lo que permite desarrollarlos en paralelo.
Estos son los argumentos del módulo:

vg: el nombre del VG, de carácter obligatorio.
state: el valor present indica que hay que crear el VG mientras que absent especifica que se debe eliminar. El valor por defecto es present.
pvs: especifica la lista de dispositivos de bloques que se usarán para crear el VG. Es obligatorio para la opción present y se ignora para absent. Si el VG ya existe, se reajusta la lista de PV asociados al VG teniendo cuenta este argumento y el siguiente.
remove_extra_pvs: si es verdadero, cuando se reajusta un VG existente, se eliminan los PVs actuales que no estén especificados en el argumento pvs, mientras que, si es falso, se mantienen. Tiene como valores true o false, siendo, por defecto, true.
force: fuerza la operación de crear o destruir un VG aunque las condiciones no lo aconsejen (por ejemplo, si hay un LV sobre el VG). Tiene como valores true o false, siendo, por defecto, false. Esta opción causa que se incluya en el mandato correspondiente el argumento que especifica que se fuerce la realización de la operación.
En cuanto al valor devuelto:
0: se ha completado correctamente realizando cambios (se ha creado o eliminado el VG).
128: se ha completado correctamente, pero sin realizar cambios.
1: error por la ejecución sin ser superusuario.
2: error en los argumentos.
3: error por no estar instalado el paquete requerido.
4: error porque alguno de los dispositivos no existe o no es de bloques.
5: error en mandato que crea el VG.
6: error en mandato que elimina el VG.
7: error en mandato que reajusta el VG.
En primer lugar, el módulo debe verificar los argumentos terminando con un 2 en caso de error. A continuación, debe comprobar que, con independencia del tipo de operación, el paquete que realiza la gestión de volúmenes (lvm2) está instalado, devolviendo un valor 3 en caso contrario.
En la operación de creación del VG, hay que verificar primero que todos los dispositivos de bloques especificados son correctos, retornando un 4 en caso contrario. Recuerde que el argumento pvs se ignora en la operación de eliminación.

Por lo demás, solamente resaltar cómo se debe gestionar la lista de PV asociados a un VG en aras de optimizar su comportamiento:

Si remove_extra_pvs es falso, solo se podrá ejecutar una vez el mandato vgextend, en caso de que sea necesario, incluyendo en el mismo solo los PV que actualmente no forman parte del VG.
Si remove_extra_pvs es verdadero, con respecto a los nuevos PV, se debe cumplir lo mismo que en el caso anterior. En cuanto a la eliminación de PV, solo se podrá ejecutar una vez el mandato vgreduce, si es necesario, incluyendo en el mismo solo los PV que actualmente forman parte del VG, pero que deben dejar de hacerlo.
Nótese que en mi solución he usado los mandatos lblk, vgs, vgcreate, vgextend, vgreduce y vgremove.

Pruebas
En este caso, vamos a crear tres discos virtuales para las pruebas (para Docker, se explicó cómo hacerlo en la sección dedicada a esta tecnología).
truncate -s 1GiB disco1 disco2 disco3 # crea 3 ficheros vacíos de 1GiB
export DISCO1=$(sudo losetup -f --show disco1) # crea dispo. de bloques de tipo loop
export DISCO2=$(sudo losetup -f --show disco2) # crea dispo. de bloques de tipo loop
export DISCO3=$(sudo losetup -f --show disco3) # crea dispo. de bloques de tipo loop
A continuación, se especifican algunas pruebas para este módulo que podrán ser ejecutadas en la plataforma que considere oportuno (disponibles en el fichero pruebas_vg del material de apoyo).
$ echo $DISCO1 $DISCO2 $DISCO3
/dev/loop25 /dev/loop26 /dev/loop27
./vg  # no SUDO; error 1
$ echo $?
1
$ sudo ./vg state=present # no vg; error 2
$ echo $?
2
$ sudo ./vg state=present vg=VG # no pvs; error 2
$ echo $?
2
$ sudo ./vg state=yes vg=VG pvs=$DISCO1 # state erróneo; error 2
$ echo $?
2
$ sudo ./vg state=present vg=VG pvs=$DISCO1 remove_extra_pvs=yes # remove_extra_pvs erróneo; error 2
$ echo $?
2
$ sudo ./vg state=present vg=VG pvs=$DISCO1 force=yes # force erróneo; error 2
$ echo $?
2
# asumiendo que LVM2 no está instalado
$ sudo ./vg state=present vg=VG pvs=$DISCO1 # lvm2 no instalado; error 3
$ echo $?
3

# instale lvm2

$ sudo ./vg state=present vg=VG pvs=$DISCO1,noexiste,$DISCO2 # un dispo no existe; error 4
$ echo $?
4
$ sudo ./vg state=present vg=VG pvs=$DISCO1,/etc/passwd # un dispo no es de bloques; error 4
$ echo $?
4
$ sudo ./vg state=present vg=VG pvs=$DISCO1,$DISCO2 # crea el VG con 2 PV; retorna 0
$ echo $?
0
$ sudo vgs
  VG #PV #LV #SN Attr   VSize  VFree 
  VG   2   0   0 wz--n- <2.00g <2.00g
$ sudo pvs
  PV          VG Fmt  Attr PSize    PFree   
  /dev/loop25 VG lvm2 a--     1.00g    1.00g
  /dev/loop26 VG lvm2 a--  1020.00m 1020.00m

$ sudo ./vg state=present vg=VG pvs=$DISCO2,$DISCO1 # ya creado; retorna 128
$ echo $?
128
$ sudo ./vg state=present vg=VG pvs=$DISCO3,$DISCO1 # quedarán $DISCO1 y DISCO3; retorna 0
$ echo $?
0
$ sudo vgs 
  VG #PV #LV #SN Attr   VSize  VFree 
  VG   2   0   0 wz--n- <2.00g <2.00g
$ sudo pvs
  PV          VG Fmt  Attr PSize    PFree   
  /dev/loop25 VG lvm2 a--     1.00g    1.00g
  /dev/loop26    lvm2 ---     1.00g    1.00g
  /dev/loop27 VG lvm2 a--  1020.00m 1020.00m

$ sudo ./vg state=present vg=VG pvs=$DISCO1,$DISCO2 # quedarán $DISCO1 y DISCO2; retorna 0
$ echo $?
0
$ sudo vgs 
  VG #PV #LV #SN Attr   VSize  VFree 
  VG   2   0   0 wz--n- <2.00g <2.00g
$ sudo pvs
  PV          VG Fmt  Attr PSize    PFree   
  /dev/loop25 VG lvm2 a--     1.00g    1.00g
  /dev/loop26 VG lvm2 a--  1020.00m 1020.00m
  /dev/loop27    lvm2 ---     1.00g    1.00g
$ sudo ./vg state=present vg=VG pvs=$DISCO3,$DISCO1 remove_extra_pvs=false # estarán los 3; retorna 0
$ echo $?
0
$ sudo vgs VG
  VG #PV #LV #SN Attr   VSize VFree
  VG   3   0   0 wz--n- 2.99g 2.99g
$ sudo pvs
  PV          VG Fmt  Attr PSize    PFree   
  /dev/loop25 VG lvm2 a--     1.00g    1.00g
  /dev/loop26 VG lvm2 a--  1020.00m 1020.00m
  /dev/loop27 VG lvm2 a--  1020.00m 1020.00m
$ sudo ./vg state=present vg=VG pvs=$DISCO3 remove_extra_pvs=false # seguirán los 3; retorna 128
$ echo $?
128
$ sudo ./vg state=absent vg=VG # elimina el VG; retorna 0 
$ echo $?
0
$ sudo vgs
$ sudo ./vg state=absent vg=VG # ya eliminado el VG; retorna 128
$ echo $?
128
Módulo lv
Se encarga de la creación, redimensionamiento y eliminación de un volumen lógico (LV).
Estos son los argumentos del módulo:

lv: el nombre del LV, de carácter obligatorio.
vg: el nombre del VG, de carácter obligatorio.
state: el valor present indica que hay que crear el LV mientras que absent especifica que se debe eliminar. El valor por defecto es present.
size: especifica el tamaño del LV. Es obligatorio para la opción present y se ignora para absent. Puede usar los siguientes sufijos, tal como se explica en la página de manual de lvcreate: b|B is bytes, s|S is sectors of 512 bytes, k|K is KiB, m|M is MiB, g|G is GiB, t|T is TiB, p|P is PiB, e|E is EiB (si no se especifica una unidad, se consideran MiB).
active: activa y desactiva un LV. Tiene como valores true o false, respectivamente, siendo, por defecto, true.
force: fuerza la operación de crear o destruir un LV aunque las condiciones no lo aconsejen. Tiene como valores true o false, siendo, por defecto, false. Esta opción causa que se incluya en el mandato correspondiente el argumento que especifica que se fuerce la realización de la operación.
En cuanto al valor devuelto:
0: se ha completado correctamente realizando cambios (se ha creado o eliminado el LV).
128: se ha completado correctamente, pero sin realizar cambios.
1: error por la ejecución sin ser superusuario.
2: error en los argumentos.
3: error por no estar instalado el paquete requerido.
4: error porque el VG no existe.
5: error en mandato que crea el LV.
6: error en mandato que elimina el LV.
7: error en mandato que redimensiona el LV.
8: error en mandato que activa/desactiva el LV.
En primer lugar, el módulo debe verificar los argumentos terminando con un 2 en caso de error. A continuación, debe comprobar que, con independencia del tipo de operación, el paquete que realiza la gestión de volúmenes (lvm2) está instalado, devolviendo un valor 3 en caso contrario. Asimismo, hay que verificar que el VG existe, retornando un 4 en caso contrario.
A la hora de crear un nuevo LV, para simplificar, solo se va a permitir especificar el tamaño de forma absoluta (opción -L del mandato lvcreate), usándose directamente el valor recibido en size.

En el caso del valor present, si el LV ya existe, hay que determinar si es necesario el redimensionamiento. En este caso, se va a usar una estrategia anticipada, de manera que solo se llamará a lvresize si realmente hay un cambio de tamaño. Por tanto, es necesario comparar el tamaño actual del LV y el especificado como argumento. Para facilitar la labor, se proporciona una función (size_in_bytes) que calcula el tamaño en bytes del valor recibido como argumento, procesando los diversos sufijos y teniendo en cuenta el redondeo en la asignación, escribiendo por la salida estándar el valor resultante y devolviendo un 1 en caso de error de formato y un 0 si es correcto. Nótese que no es necesario revisar el código de la función y puede usarlo como una caja negra.

Con respecto, a la activación y desactivación, tenga en cuenta que al crearse comienza en estado activo y podrá cambiarse entre ambos estados usando lvchange.

Nótese que en mi solución he usado los mandatos lvs, vgs, lvcreate, lvresize, lvchange y lvremove.

Pruebas
En este caso, basta con un disco virtual para las pruebas (recuerde que, para Docker, esta fase es problemática, tal como se explicó en la sección dedicada a esta tecnología, y es necesario crear un contenedor privilegiado con total visibilidad del directorio de dispositivos, como se especificó en dicha sección).
truncate -s 1GiB disco1  # crea fichero vacíos de 1GiB
export DISCO1=$(sudo losetup -f --show disco1) # crea dispo. de bloques de tipo loop
A continuación, se especifican algunas pruebas para este módulo que podrán ser ejecutadas en la plataforma que considere oportuno (disponibles en el fichero pruebas_lv del material de apoyo).
./lv  # no SUDO; error 1
$ echo $?
1
$ sudo ./lv state=present # no lv; error 2
$ echo $?
2
$ sudo ./lv state=present lv=LV1 # no vg; error 2
$ echo $?
2
$ sudo ./lv state=present lv=LV1 vg=VG1 # falta size; error 2
$ echo $?
2
$ sudo ./lv state=yes lv=LV1 vg=VG1 # state erróneo; error 2
$ echo $?
2
$ sudo ./lv state=present lv=LV1 vg=VG1 active=yes # active erróneo; error 2
$ echo $?
2
$ sudo ./lv state=present lv=LV1 vg=VG1 force=yes # force erróneo; error 2
$ echo $?
2
# asumiendo que LVM2 no está instalado
$ sudo ./lv state=present lv=LV1 vg=VG1 size=128M # lvm2 no instalado; error 3
$ echo $?
3

# instale lvm2; recuerde que si está usando Docker una vez completada la instalación debe incluir en el fichero /etc/lvm/lvm.conf estas dos líneas:
udev_sync = 0
udev_rules = 0

$ sudo ./lv state=present lv=LV1 vg=VG1 size=128M # vg no existe; error 4
$ echo $?
4
# se crea directamente un VG (para no depender del módulo previo)
$ sudo vgcreate VG1 $DISCO1  
  Physical volume "/dev/loop26" successfully created.
  Volume group "VG1" successfully created

$ sudo ./lv state=present lv=LV1 vg=VG1 size=128M # crea el LV; retorna 0
$ echo $?
0
$ sudo lvs VG1/LV1
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1 -wi-a----- 128.00m   

$ sudo ./lv state=present lv=LV1 vg=VG1 size=128M # LV ya creado con mismo tamaño; retorna 128
$ echo $?
128
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M # redimensiona el LV; retorna 0
$ echo $?
0
$ sudo lvs VG1/LV1
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1 -wi-a----- 256.00m   
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M # mantiene mismo tamaño; retorna 128
$ echo $?
128
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M active=false # desactiva LV; retorna 0
$ echo $?
0
$ sudo lvs VG1/LV1
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1 -wi------- 256.00m 
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M active=false # ya desactivado; retorna 128
$ echo $?
128
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M active=true # reactiva LV; retorna 0
$ echo $?
0
$ sudo lvs VG1/LV1
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1 -wi-a----- 256.00m 
$ sudo ./lv state=present lv=LV1 vg=VG1 size=256M active=true # ya activado; retorna 128
$ echo $?
128
$ sudo ./lv state=absent lv=LV1 vg=VG1 # elimina LV; retorna 0
$ echo $?
0
$ sudo lvs VG1/LV1
  Failed to find logical volume "VG1/LV1"
$ sudo ./lv state=absent lv=LV1 vg=VG1 # ya eliminado; retorna 128
$ echo $?
128
Mandato mansible
Permite la ejecución de un módulo con unos argumentos en el conjunto de máquinas especificadas. Por tanto, el formato del mandato es el siguiente:
./mansible "host..." módulo "args..."
Tanto este mandato como el siguiente se ejecutarán sin sudo, pero sí lo usarán internamente en la ejecución de los scripts remotos. Nótese que se asume que en todos los equipos existe un mismo usuario con acceso a sudo y a ssh sin contraseña, de manera que la ejecución remota se realice en el contexto de este usuario.

La funcionalidad de este mandato es la siguiente:

Debe comprobar que el número de argumentos es correcto (tres: equipos, módulo y argumentos del módulo), imprimiendo un mensaje por la salida de error en caso de que no lo sea y devolviendo el valor 1.
Está organizado como un bucle que ejecuta una iteración por cada máquina remota especificada.
Debe copiar con scp el módulo y el prólogo al directorio /tmp del equipo remoto.
Y ejecutarlo con ssh en el contexto de un sudo en el equipo remoto pasándole los argumentos. Nótese que debe usarse la opción -n que impide que ssh lea de la entrada estándar.
Este mandato no escribirá nada por la salida estándar ni por la de error excepto lo que se indica a continuación.
La ejecución del módulo puede dar cuatro posibles resultados que se reflejarán en la salida estándar:
El equipo remoto X es inaccesible: X | UNREACHABLE!
La ejecución del módulo ha retornado un 0: X | CHANGED
La ejecución del módulo ha retornado un 128: X | SUCCESS
La ejecución del módulo ha retornado un error Y: X | FAILED (code=Y)
En cualquier caso, el mandato pasará a ejecutar la siguiente iteración que corresponde a otro equipo.
Para probar esta fase, se ha preparado una versión muy simplificada de un módulo de gestión de usuarios (user), preparada específicamente para esta fase, que permite desarrollarla con independencia de las pruebas previas.
Pruebas
A continuación, se realizan pruebas de este mandato usando el módulo user con dos equipos existentes y otro que no existe (si usa Docker, se trata de dos instancias, tal como se explicó en la sección dedicada a esta tecnología).
# la tercera máquina no existe
$ ./mansible "172.17.0.2 172.17.0.3 172.17.0.10" user "state=present name=test"
172.17.0.2 | CHANGED
172.17.0.3 | CHANGED
172.17.0.10 | UNREACHABLE!
# verificando que ha funcionado
$ ssh 172.17.0.2 getent passwd test
test:x:1002:1002::/home/test:/bin/sh
$ ssh 172.17.0.3 getent passwd test
test:x:1002:1002::/home/test:/bin/sh
# probando la idempotencia
$ ./mansible "172.17.0.2 172.17.0.3 172.17.0.10" user "state=present name=test"
172.17.0.2 | SUCCESS
172.17.0.3 | SUCCESS
172.17.0.10 | UNREACHABLE!
# probando mandato erróneo (nombre de usuario tiene solo números)
$ ./mansible "172.17.0.2 172.17.0.3 172.17.0.10" user "state=present name=12345"
2.17.0.10" user "name=12345"
172.17.0.2 | FAILED (code=3)
172.17.0.3 | FAILED (code=3)
172.17.0.10 | UNREACHABLE!
# eliminando usuario
$ ./mansible "172.17.0.2 172.17.0.3 172.17.0.10" user "name=test state=absent"
172.17.0.2 | CHANGED
172.17.0.3 | CHANGED
172.17.0.10 | UNREACHABLE!
# verificando que ha funcionado
$ ssh 172.17.0.2 getent passwd test
$ ssh 172.17.0.3 getent passwd test
# probando la idempotencia
$ ./mansible "172.17.0.2 172.17.0.3 172.17.0.10" user "name=test state=absent"
172.17.0.2 | SUCCESS
172.17.0.3 | SUCCESS
172.17.0.10 | UNREACHABLE!
Mandato mansible-playbook
Se trata del mandato que ejecuta playbooks, que es la esencia de Ansible:
mansible-playbook fich_playbook
El formato del fichero de configuración es el que se muestra a continuación (nótese que es una versión simplificada del usado en Ansible; tenga en cuenta que Ansible también permite la notación basada en el símbolo = aunque no sea la que se usa habitualmente en los playbooks):
La primera línea especifica a qué equipos se le aplica el playbook (el inventario).
Cada una de las líneas restantes es una tarea que especifica a su vez:
un nombre, que no puede incluir espacios.
un módulo.
los argumentos con los que se ejecutará el módulo (siempre habrá como mínimo uno).
En cuanto a la salida del mandato, debe seguir estrictamente el siguiente formato (puede ver el detalle en las pruebas):
Una línea con la etiqueta PLAY y el nombre del fichero que se está procesando.
Una línea con la etiqueta TASK y el nombre de la tarea por cada una que se vaya ejecutando.
Por cada tarea una línea por cada equipo remoto con el resultado de la ejecución de la tarea:
El equipo remoto X es inaccesible: unreachable: [X]
La ejecución del módulo ha retornado un 0: changed: [X]
La ejecución del módulo ha retornado un 128: ok: [X]
La ejecución del módulo ha retornado un error: fatal: [X]
Una línea con la etiqueta PLAY RECAP al final de la ejecución y, a continuación, una línea por cada equipo X con el siguiente formato:
X: ok=n changed=n unreachable=n failed=n
tal que cada contador indica cuántas tareas han producido ese resultado en ese equipo (aunque sea contraintuitivo, siguiendo la pauta de Ansible, ok incluye también changed).
La funcionalidad de este mandato es la siguiente:
Debe comprobar que recibe un único argumento, imprimiendo un mensaje por la salida de error en caso de error y retornando un valor de 1.
Debe comprobar que corresponde a un fichero normal con acceso de lectura, imprimiendo un mensaje por la salida de error en caso de que no lo sea y retornando un valor de 2.
De la primera línea debe extraer los equipos remotos.
Por cada una de las siguientes líneas debe:
Comprobar que el formato es correcto: debe incluir los campos correspondientes y el módulo debe ser un fichero ejecutable. En caso de error, se pasa a procesar la siguiente línea.
Con respecto al procesado de cada línea, es similar al realizado por el mandato mansible, adaptando la salida al formato indicado.
Terminado todo el procesamiento hay que imprimir la recapitulación. Se sugiere el uso de arrays asociativos para implementar esta contabilidad.
Para probar esta fase, se ha preparado una versión muy simplificada de un módulo de gestión de usuarios (user), preparada específicamente para esta fase, que permite desarrollarla con independencia de las pruebas previas.
Pruebas
A continuación, se realizan pruebas de este mandato usando el módulo user con dos equipos existentes y otro que no existe (si usa Docker, se trata de dos instancias, tal como se explicó en la sección dedicada a esta tecnología).
Este es el fichero test.play:

172.17.0.2 172.17.0.3 172.17.0.10
crea_usuario user name=test state=present
recrea_usuario user name=test state=present
crea_usuario_error user name=12345 state=present
elimina_usuario user name=test state=absent
reelimina_usuario user name=test state=absent
El resultado de la ejecución es:
./mansible-playbook test.play
PLAY [test.play] *********************************************

TASK [crea_usuario] *********************************************
changed: [172.17.0.2]
changed: [172.17.0.3]
unreachable: [172.17.0.10]

TASK [recrea_usuario] *********************************************
ok: [172.17.0.2]
ok: [172.17.0.3]
unreachable: [172.17.0.10]

TASK [crea_usuario_error] *********************************************
fatal: [172.17.0.2]
fatal: [172.17.0.3]
unreachable: [172.17.0.10]

TASK [elimina_usuario] *********************************************
changed: [172.17.0.2]
changed: [172.17.0.3]
unreachable: [172.17.0.10]

TASK [reelimina_usuario] *********************************************
ok: [172.17.0.2]
ok: [172.17.0.3]
unreachable: [172.17.0.10]

PLAY RECAP *********************************************
172.17.0.2: ok=4 changed=2 unreachable=0 failed=1
172.17.0.3: ok=4 changed=2 unreachable=0 failed=1
172.17.0.10: ok=0 changed=0 unreachable=5 failed=0
Entrega de la práctica
El plazo se extiende hasta el final del 27 de enero de 2027 en la convocatoria ordinaria y hasta el 5 de julio de 2027 en la extraordinaria. Se realizará en la máquina triqui, usando el mandato:
entrega.asi proyecto.2026
Este mandato recogerá los siguientes ficheros:
autores.txt: Fichero con los datos de los autores:
DNI APELLIDOS NOMBRE MATRÍCULA
pv
vg
lv
mansible
mansible-playbook