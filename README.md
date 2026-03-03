# AutoDRIVE - ROS 2 Humble (AIROS) 🏎️

Este repositorio proporciona una guía detallada para instalar el simulador AutoDriver y configurar el puente de comunicación entre el **AutoDRIVE Simulator** y **ROS 2 Humble**. El objetivo es permitir el control y la adquisición de datos del vehículo **F1TENTH** para proyectos de robótica y navegación autónoma.

---

## 📋 Requisitos Previos

Antes de comenzar, asegúrate de contar con el siguiente entorno:

* **Sistema Operativo:** Ubuntu 22.04 
* **Middleware:** ROS 2 Humble Hawksbill.

---

## 🕹️ 1. Instalación del Simulador

Sigue estos pasos para preparar el entorno de simulación:

1. **Descarga** el archivo comprimido (.zip o .tar.gz) del simulador para Linux.
   
2. **Guarda** el archivo en un lugar de fácil acceso y **descomprímelo** (ej. `~/Downloads/AutoDRIVE_Sim`).

3. **Accede a la terminal** y dirígete a la carpeta donde descomprimiste el simulador. Ejemplo:
   ```bash
   cd ~/Downloads/AutoDRIVE_Sim
4. Otorga permisos de ejecución al binario principal:
      ```bash
   chmod +x "AutoDRIVE Simulator.x86_64"
6. Inicia el simulador ejecutando el siguiente comando en la terminal:
     ```bash
   ./"AutoDRIVE Simulator.x86_64"

## 🛠️ 2. DEVKIT ROS2

**2.1Configuración del Workspace y Entorno Virtual**

Para evitar conflictos entre las versiones de librerías requeridas por el Devkit y las del sistema (Ubuntu 22.04), utilizaremos un **Entorno Virtual (venv)**. Esto permite aislar las dependencias de Python 3.8 en un entorno controlado.

### 2.2Crear Espacio de Trabajo
Ejecuta los siguientes comandos para crear la estructura de carpetas:

   ```bash
mkdir -p ~/autodrive_ws/src
cd ~/autodrive_ws
```
### 2.3Configurar el Entorno Virtual (venv)
```bash
Crea y activa el entorno dentro de la raíz de tu workspace:
# Crear el entorno
python3 -m venv venv

# Activar el entorno (Debes hacer esto en cada terminal nueva)
source venv/bin/activate
```
Instalación de Dependencias

Con el entorno activo (venv), instala las librerías necesarias con las versiones probadas para estabilidad:
```bash
pip install eventlet==0.33.3 Flask-SocketIO==4.1.0 python-socketio==4.2.0 \
            python-engineio==3.13.0 gevent-websocket==0.10.1 \
            transforms3d attrdict numpy==1.23.5 opencv-contrib-python
```

Parche de Compatibilidad Crítico

La librería attrdict presenta incompatibilidades con los módulos de colecciones en Python 3.10. Para solucionar el error de importación sin afectar el sistema global, aplica el siguiente parche directamente a los archivos dentro de tu venv:```bash
```bash
# Corregir error de importación de Mapping en attrdict
find ~/autodrive_ws/venv/lib/python3.10/site-packages/attrdict/ -name "*.py" -exec sed -i 's/from collections import Mapping/from collections.abc import Mapping/g' {} +

# Corregir variantes de importación múltiple
find ~/autodrive_ws/venv/lib/python3.10/site-packages/attrdict/ -name "*.py" -exec sed -i 's/from collections import Mapping, MutableMapping, Sequence/from collections.abc import Mapping, MutableMapping, Sequence/g' {} +

```

## 2.4 Clonación y Organización del Workspace

Para que ROS 2 compile correctamente, debemos extraer únicamente el metapaquete necesario del repositorio oficial y limpiar los archivos redundantes.

### Clonación del Repositorio
Dirígete a la carpeta de código de tu espacio de trabajo y clona el Devkit:

```bash
cd ~/autodrive_ws/src
git clone --single-branch --branch AutoDRIVE-Devkit [https://github.com/Tinker-Twins/AutoDRIVE.git](https://github.com/Tinker-Twins/AutoDRIVE.git)
```
Extracción y Limpieza

Para evitar errores de paquetes duplicados durante la compilación, mueve la carpeta de ROS 2 a la raíz de src y elimina el resto del repositorio:
```bash
# Mover el metapaquete de ROS 2
mv ~/autodrive_ws/src/AutoDRIVE/ADSS\ Toolkit/autodrive_ros2 ~/autodrive_ws/src/

# Eliminar el repositorio original para evitar conflictos
rm -rf ~/autodrive_ws/src/AutoDRIVE
```


## 2.5 Compilación del Workspace

Una vez organizada la carpeta src, procedemos a construir los paquetes. Asegúrate de tener el entorno virtual activo para que ROS reconozca las dependencias de Python instaladas.
```bash

# Regresar a la raíz del workspace
cd ~/autodrive_ws

# Activar entorno y cargar ROS 2
source venv/bin/activate
source /opt/ros/humble/setup.bash

# Compilar
colcon build --symlink-install

# Cargar la configuración del workspace recién compilado
source install/setup.bash
```


