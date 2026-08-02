

openpilot-tools
============

Repositorio que contiene herramientas para facilitar el desarrollo y la depuración de [openpilot](openpilot.comma.ai).

![Imgur](https://i.imgur.com/IdfBgwK.jpg)


Tabla de Contenidos
============

<!--ts-->
 * [Requisitos](#requirements)
 * [Configuración](#setup)
 * [Ejemplos de herramientas](#tool-examples)
   * [Reproducir datos de conducción](#replay-driving-data)
   * [Depurar controles del vehículo](#debug-car-controls)
   * [Transmitir mensajes CAN reproducidos a EON](#stream-replayed-can-messages-to-eon)
   * [Transmitir datos de video de EON a una PC](#stream-eon-video-data-to-a-pc)
 * [Contribuciones bienvenidas](#welcomed-contributions)
<!--te-->


Requisitos
============

`openpilot-tools` y los siguientes pasos de configuración se desarrollan y prueban en Ubuntu 16.04 y Python 2.7.


Configuración
============

1. Instalar dependencias nativas

    - herramientas principales
        ```bash
        sudo apt install git curl python-pip
        sudo pip install --upgrade pip==9.0.3
        ```

    - ffmpeg (probado con 3.3.2)
        ```bash
        sudo apt install ffmpeg libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev libavresample-dev libavfilter-dev
        ```

    - herramientas de compilación
        ```bash
        sudo apt install autoconf automake clang clang-3.8 libtool pkg-config build-essential
        ```

    - libarchive-dev (probado con 3.1.2-11ubuntu0.16.04.4)
        ```bash
        sudo apt install libarchive-dev
        ```

    - enlace de qt para python (probado con python-qt4, 4.11.4+dfsg-1build4)
        ```bash
        sudo apt install python-qt4
        ```

    - capnp 0.6.1 (requerido para la reproducción)
        ```bash
        curl -O https://capnproto.org/capnproto-c++-0.6.1.tar.gz
        tar xvf capnproto-c++-0.6.1.tar.gz
        cd capnproto-c++-0.6.1
        ./configure --prefix=/usr CPPFLAGS=-DPIC CFLAGS=-fPIC CXXFLAGS=-fPIC LDFLAGS=-fPIC --disable-shared --enable-static
        make -j4
        sudo make install

        cd ..
        git clone https://github.com/commaai/c-capnproto.git
        cd c-capnproto
        git submodule update --init --recursive
        autoreconf -f -i -s
        CFLAGS="-fPIC" ./configure --prefix=/usr/local
        make -j4
        sudo make install
        ```

    - zmq 4.2.3 (requerido para la reproducción)
        ```bash
        curl -LO https://github.com/zeromq/libzmq/releases/download/v4.2.3/zeromq-4.2.3.tar.gz
        tar xfz zeromq-4.2.3.tar.gz
        cd zeromq-4.2.3
        ./autogen.sh
        ./configure CPPFLAGS=-DPIC CFLAGS=-fPIC CXXFLAGS=-fPIC LDFLAGS=-fPIC --disable-shared --enable-static
        make
        sudo make install
        ```

2. Clonar openpilot si aún no lo has hecho

    ```bash
    git clone https://github.com/commaai/openpilot.git
    cd openpilot
    ```

3. Clonar openpilot_tools dentro de openpilot e instalar las dependencias

    ```bash
    git clone https://github.com/commaai/openpilot-tools.git openpilot_tools
    cd openpilot_tools
    # sudo pip install si no estás usando un entorno virtual
    pip install -r requirements.txt
    pip install -r ../requirements_openpilot.txt
    ```

4. Agregar openpilot a tu `PYTHONPATH`.

    Para usuarios de bash:
    ```bash
    echo 'export PYTHONPATH="$PYTHONPATH:<path-to-openpilot>"' >> ~/.bashrc
    source ~/.bashrc
    ```

5. Agregar algunas carpetas a la raíz
    ```bash
    sudo mkdir /data
    sudo mkdir /data/params
    sudo chown $USER /data/params
    ```

6. ¡Prueba algunas herramientas!


Ejemplos de herramientas
============


Reproducir datos de conducción
-------------

***Hardware necesario**: ninguno*

`unlogger.py` reproduce los datos recopilados con [chffrplus](https://github.com/commaai/chffrplus) o [openpilot](https://github.com/commaai/openpilot).

Necesitarás descargar los archivos de registro y de cámara a un directorio local. Descárgalos desde el pie de página del [explorer](https://my.comma.ai) de comma o mediante SCP desde tu dispositivo.

Uso:

```
python replay/unlogger.py <route-name> <path-to-data-directory>

# Ejemplo:

#python replay/unlogger.py '99c94dc769b5d96e|2018-11-14--13-31-42' /home/batman/unlogger_data

# Dentro de /home/batman/unlogger_data:
#  99c94dc769b5d96e|2018-11-14--13-31-42--0--fcamera.hevc
#  99c94dc769b5d96e|2018-11-14--13-31-42--0--rlog.bz2
#  ...

# En otra terminal puedes ejecutar un visualizador de depuración:
python replay/ui.py   # Define la variable de entorno HORIZONTAL si el diseño de la interfaz es demasiado alto
```
![Imgur](https://i.imgur.com/Yppe0h2.png)


Depurar controles del vehículo
-------------

***Hardware necesario**: [panda](panda.comma.ai), [giraffe](https://comma.ai/shop/products/giraffe/), joystick*

Usa el puerto OBD-II de la panda para conectarte con tu coche y un cable USB para conectar la panda a tu PC.
Además, conecta un joystick a tu PC.

`joystickd.py` ejecuta un demonio que lee las entradas de un joystick y las publica a través de zmq.
`boardd.py` envía los mensajes CAN desde tu PC a la panda.
`debug_controls` es una versión simulada de `controlsd.py` y utiliza la entrada de un joystick para enviar controles a tu vehículo.

Uso:
```
python carcontrols/joystickd.py

# En otra terminal:
selfdrive/boardd/boardd.py # Asegúrate de que la configuración de seguridad esté codificada como ALL_OUTPUT

# En otra terminal:
python carcontrols/debug_controls.py

```
![Imgur](steer.gif)


Transmitir mensajes CAN reproducidos a EON
-------------

***Hardware necesario**: 2 x [panda](panda.comma.ai), [placa de depuración](https://comma.ai/shop/products/panda-debug-board/), [EON](https://comma.ai/shop/products/eon-gold-dashcam-devkit/).*

Es posible reproducir los mensajes CAN tal como fueron registrados y reenviarlos a un EON. 
Conecta 2 pandas a la placa de depuración. Una panda se conecta a la PC, la otra panda se conecta al EON.

Uso:
```
# Con MOCK=1, boardd leerá los mensajes CAN registrados desde una reproducción y los enviará a la panda.
MOCK=1 openpilot_tools/replay/boardd.py

# En otra terminal:
python replay/unlogger.py <route-name> <path-to-data-directory>

```
![Imgur](https://i.imgur.com/AcurZk8.jpg)


Transmitir datos de video de EON a una PC
-------------

***Hardware necesario**: [EON](https://comma.ai/shop/products/eon-gold-dashcam-devkit/), [comma Smays](https://comma.ai/shop/products/comma-smays-adapter/).*

Puedes conectar tu EON a tu PC usando el cable Ethernet proporcionado con el adaptador comma Smays y podrás transmitir datos desde tu EON, en tiempo real, con baja latencia. Una aplicación útil es poder transmitir los fotogramas de video en bruto a 20fps, tal como los captura la cámara del EON.

Uso:
```
# Conéctate por SSH al EON y ejecuta loggerd con la bandera "--stream". En ../selfdrive/manager.py puedes cambiar:
# ...
# "loggerd": ("selfdrive/loggerd", ["./loggerd"]),
# ...
# por:
# ...
# "loggerd": ("selfdrive/loggerd", ["./loggerd", "--stream"]),
# ...

# En la PC:
# Para recibir fotogramas del EON y re-publicarlos. Establece la variable de entorno PYGAME si deseas mostrar la transmisión de video
python streamer/streamerd.py
```

![Imgur](stream.gif)


Contribuciones bienvenidas
=============

* Documentación: comentarios en el código, mejores tutoriales, etc.
* Soporte para otras plataformas distintas a Ubuntu 16.04.
* Mejoras de rendimiento: las herramientas se han desarrollado en estaciones de trabajo de alto rendimiento (12+ núcleos lógicos con 32+ GB de RAM), por lo que no están optimizadas para ejecutarse eficientemente. Por ejemplo, `ui.py` podría no poder ejecutarse en tiempo real en la mayoría de las PCs.
* Más herramientas: cualquier cosa que creas que pueda ser útil para otros.
