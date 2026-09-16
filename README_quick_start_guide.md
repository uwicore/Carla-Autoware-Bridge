# Guía rápida de uso

[Guía rápida original de Mario](doc/README_quick_start_guide_Mario.md)

[Repositorio de scripts de simulación](https://github.com/uwicore/Carla-Autoware-Scripts)

Esta guía asume que la plataforma ya está instalada y configurada. Los componentes deben iniciarse en el orden indicado y cada apartado debe ejecutarse en una terminal diferente.

## 1. CARLA

Configura los búferes de red y ejecuta CARLA:

```bash
sudo sysctl -w net.core.rmem_max=2147483647
sudo sysctl -w net.core.wmem_max=2147483647
sudo sysctl -w net.ipv4.ipfrag_time=3
sudo sysctl -w net.ipv4.ipfrag_high_thresh=134217728

cd /home/uwicore_pc4/proyectos/CARLA_0.9.15
./CarlaUE4.sh -carla-rpc-port=1403 -prefernvidia -quality-level=Medium -resX=640 -resY=480
```

Para ejecutarlo sin ventana gráfica, añade `-RenderOffScreen` al último comando.

## 2. CARLA-Autoware-Bridge

Inicia el contenedor del puente:

```bash
docker run -it \
  --name carlaBridge_docker \
  -e ROS_DOMAIN_ID=4 \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  --network host \
  tumgeka/carla-autoware-bridge:latest
```

Dentro del contenedor, lanza el puente para Town04:

```bash
ros2 launch carla_autoware_bridge carla_aw_bridge.launch.py port:=1403 town:=Town04
```

Espera a que el puente termine de crear los objetos antes de continuar.

## 3. Vista de CARLA (opcional)

Si necesitas la vista auxiliar de Town04:

```bash
cd /home/uwicore_pc4/proyectos/TUM/Carla-Autoware-Scripts
source /home/uwicore_pc4/venvs/carla0915/bin/activate
python vista4.py
```

## 4. Autoware

Inicia el contenedor de Autoware:

```bash
rocker \
  --persist-image \
  --network=host \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e LIBGL_ALWAYS_SOFTWARE=1 \
  -e ROS_DOMAIN_ID=4 \
  --x11 \
  --nvidia \
  --volume /home/uwicore_pc4/proyectos/TUM/ \
  --name 'autoware_docker' \
  -- autoware-fixed
```

Dentro del contenedor, carga el entorno y lanza Autoware:

```bash
cd /home/uwicore_pc4/proyectos/TUM/autoware
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch autoware_launch e2e_simulator.launch.xml \
  vehicle_model:=carla_t2_vehicle \
  sensor_model:=carla_t2_sensor_kit \
  map_path:=/home/uwicore_pc4/proyectos/TUM/CarlaMaps/Town04 \
  rviz_config:=/home/uwicore_pc4/proyectos/TUM/autoware/src/launcher/autoware_launch/autoware_launch/rviz/RViz_config_Jaime.rviz
```

## 5. Nodo de información de pointcloud

Abre otra terminal y entra en el contenedor de Autoware:

```bash
docker exec -it autoware_docker bash
```

Dentro del contenedor, ejecuta el nodo:

```bash
cd /home/uwicore_pc4/proyectos/TUM/autoware
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 run pointcloud_info pointcloud_info_node
```

## 6. Comprobación del pointcloud (opcional)

En otra terminal, entra en el contenedor:

```bash
docker exec -it autoware_docker bash
```

Comprueba que el tópico publica información:

```bash
source /opt/ros/humble/setup.bash
source /home/uwicore_pc4/proyectos/TUM/autoware/install/setup.bash
ros2 topic echo --once /sensing/lidar/concatenated/pointcloud_info
```

## 7. Simulación y grabación

Este proceso se ejecuta en el **host**, después de iniciar CARLA, el puente y Autoware. El contenedor de Autoware debe estar activo con el nombre `autoware_docker`.

La configuración actual utiliza el escenario `CocheAleja` de SUMO y una duración de simulación de 17 segundos.

> [!WARNING]
> Antes de iniciar una nueva tanda, comprueba que no existan rosbags ni CSV con los mismos índices. `Grabar.sh` comienza siempre en `001` y no limpia resultados anteriores antes de la primera ejecución.
>
> Si se reutiliza el nombre `prueba_ros`, deben moverse o eliminarse previamente las carpetas `prueba_ros_001`, `prueba_ros_002`, etc. de:
>
> `/home/uwicore_pc4/proyectos/TUM/autoware/ResultadosSimulaciones/CocheAleja/`
>
> También deben moverse o eliminarse los CSV anteriores con los mismos índices de:
>
> `/home/uwicore_pc4/proyectos/TUM/Trazas resultados/`
>
> Los CSV se llaman `carla_trace_001.csv`, `centerpoint_objects_001.csv`, `detection_objects_001.csv`, `kinematic_state_001.csv`, etc. Su nombre no incluye `prueba_ros`, por lo que cambiar solamente el nombre base de la rosbag no evita el conflicto. Algunos CSV se sobrescriben, pero si un tópico no contiene datos puede conservarse un archivo antiguo y confundirse con el resultado nuevo.

### Ejecutar las simulaciones

Activa el entorno virtual de CARLA y ejecuta `Grabar.sh`:

```bash
cd /home/uwicore_pc4/proyectos/TUM/Carla-Autoware-Scripts
source /home/uwicore_pc4/venvs/carla0915/bin/activate
./Grabar.sh prueba_ros 3
```

Los argumentos son:

- `prueba_ros`: nombre base de las grabaciones.
- `3`: número de simulaciones que se ejecutarán.

Las ejecuciones se numeran automáticamente como `prueba_ros_001`, `prueba_ros_002` y `prueba_ros_003`.

### Proceso automático

Por cada simulación, `Grabar.sh` llama a `EjecutarSimulacion.sh`, que realiza automáticamente lo siguiente:

1. Inicia una grabación ROS 2 dentro del contenedor de Autoware.
2. Registra en CSV los vehículos de CARLA mediante `grabar_carla.py`.
3. Ejecuta el escenario de SUMO.
4. Detiene y cierra correctamente la rosbag.
5. Convierte los datos de Autoware a CSV mediante `rosbag2csv.py`.
6. Reintenta la simulación si se produce un error controlado.

No es necesario ejecutar `raw_data_recorder` ni realizar manualmente el postprocesado descrito en la guía original de Mario.

### Datos registrados

La rosbag contiene actualmente estos tópicos:

```text
/clock
/localization/kinematic_state
/perception/object_recognition/detection/centerpoint/objects
/perception/object_recognition/detection/objects
/perception/object_recognition/objects
```

Las rosbags se guardan en:

```text
/home/uwicore_pc4/proyectos/TUM/autoware/ResultadosSimulaciones/CocheAleja/
```

Los CSV generados se guardan en:

```text
/home/uwicore_pc4/proyectos/TUM/Trazas resultados/
```

Para cada ejecución se pueden generar los siguientes archivos:

```text
carla_trace_001.csv
centerpoint_objects_001.csv
detection_objects_001.csv
kinematic_state_001.csv
```

Los CSV de detecciones solo se crean cuando el tópico contiene datos.

> **Nota:** el tópico `/sensing/lidar/concatenated/pointcloud_info` no está incluido actualmente en la lista de tópicos grabados por `EjecutarSimulacion.sh`. Por ello, el nodo `pointcloud_info` puede utilizarse para comprobar el LiDAR, pero sus datos no quedan guardados en estas simulaciones.

### Comprobación opcional de las detecciones

Entra en el contenedor de Autoware:

```bash
docker exec -it autoware_docker bash
```

Carga el entorno y comprueba los tópicos:

```bash
source /opt/ros/humble/setup.bash
source /home/uwicore_pc4/proyectos/TUM/autoware/install/setup.bash

ros2 topic echo --once /perception/object_recognition/detection/centerpoint/objects
ros2 topic echo --once /perception/object_recognition/detection/objects
```
