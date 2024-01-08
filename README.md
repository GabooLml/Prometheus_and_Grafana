# Instalación y configuración de Prometheus y Grafana para monitoreo de máquinas virtuales.

### Requisitos
- **Servidor**: Máquina con Rocky Linux 9.
- **Máquinas minitoreadas**: 2 máquinas virtuales Rocky Linux.

## Configuración del servidor
1. Configurar el backend de la política criptográfica predeterminada en SHA-1 y reiniciar el servidor.
   ```bash
   $ sudo update-crypto-policies --set DEFAULT:SHA1
   $ sudo reboot
   ```
   ![Alt text](image.png)

2. Añadir el repositorio de Grafana.
   ```bash
   $ sudo vim /etc/yum.repos.d/grafana.repo
   ```
   Agregar las siguientes líneas al archivo.
   ```bash
   [grafana]
   name=grafana
   baseurl=https://rpm.grafana.com
   repo_gpgcheck=1
   enabled=1
   gpgcheck=1
   gpgkey=https://rpm.grafana.com/gpg.key
   sslverify=1
   sslcacert=/etc/pki/tls/certs/ca-bundle.crt
   ```
   ![Alt text](image-1.png)

3. Con el repositorio agregado ahora se puede instalar Grafana.
   ```bash
   $ sudo dnf install -y grafana
   ```
   ![Alt text](image-2.png)

4. Una vez instalado Grafana habrá que recargar los privilegios de systemd.
   ```bash
   $ sudo systemctl daemon-reload
   ```

5. Inicialización y habilitación del servicio de Grafana
   ```bash
   $ sudo systemctl start grafana-server
   $ sudo systemctl enable grafana-server
   ```
   ![Alt text](image-3.png)

6. Para la instalación de Prometheus también habrá que agregar el repositorio a nuestro servidor.
   ```bash
   $ sudo vim /etc/yum.repos.d/prometheus.repo
   ```
   Se deben agregar las siguientes líneas.
   ```bash
   [prometheus]
   name=prometheus
   baseurl=https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch
   repo_gpgcheck=1
   enabled=1
   gpgkey=https://packagecloud.io/prometheus-rpm/release/gpgkey
       https://raw.githubusercontent.com/lest/prometheus-rpm/master/RPM-GPG-KEY-prometheus-rpm
   gpgcheck=1
   metadata_expire=300
   ```
   ![Alt text](image-4.png)

7. Instalar Prometheus
   ```bash
   $ sudo dnf install -y prometheus2
   ```
   ![Alt text](image-5.png)

8. Iniciar y habilitar el servicio de Prometheus.
   ```bash
   $ sudo systemctl start prometheus
   $ sudo systemctl enable prometheus
   ```
   ![Alt text](image-6.png)

9.  Configuración del servicio de Prometheus, por lo que habrá que acceder al archivo */etc/prometheus/prometheus.yml*
    ```bash
    $ sudo vim /etc/prometheus/prometheus.yml
    ```
    y agregar las siguientes líneas al final del archivo.
    ```bash
    ...
    ...
    - job_name: "node_exporter"

        static_configs:
          - targets: ["IP_Maquina_Virtual_1:9100"]
          - targets: ["IP_Maquina_Virtual_2:9100"]
    ```
    ![Alt text](image-7.png)

10. Ahora solo se debe reiniciar el servicio de Prometheus.
    ```bash
    $ sudo systemctl restart prometheus
    ```
    Hasta aquí el servidor, queda listo para poder reicibir los datos de las máquinas virtuales, ahora solo se debe ir a configurar los equipos que se van a monitorear.

### Configuración de las máquinas virtuales
Como se ha comentado inicialmente estos pasos se llevan a cabo en Rocky Linux 9.
1. Añadir el repositorio de Prometheus para poder hacer la descarga de node_exporter, el cual nos permitirá la exposición de todos los datos de los equipos monitoreados.
   ```bash
   $ sudo vim /etc/yum.repos.d/prometheus.repo
   ```
   con el siguiente contenido.
   ```bash
   [prometheus]
   name=prometheus
   baseurl=https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch
   repo_gpgcheck=1
   enabled=1
   gpgkey=https://packagecloud.io/prometheus-rpm/release/gpgkey
       https://raw.githubusercontent.com/lest/prometheus-rpm/master/RPM-GPG-KEY-prometheus-rpm
   gpgcheck=1
   metadata_expire=300
   ```
   ![Alt text](image-8.png)

2. Configurar el backend de la política criptográfica predeterminada en SHA-1 y reiniciar el servidor.
   ```bash
   $ sudo update-crypto-policies --set DEFAULT:SHA1
   $ sudo reboot
   ```
   ![Alt text](image-10.png)

2. Instalar *node_exporter*
   ```bash
   $ sudo dnf install -y node_exporter
   ```
   ![Alt text](image-11.png)

3. Se inicia y habilita el servicio de node_exporter.
   ```bash
   $ sudo systemctl start node_exporter
   $ sudo systemctl enable node_exporter
   ```
   ![Alt text](image-12.png)

4. Como el servicio trabaja por defecto en el puerto 9100, habrá que agregar este al firewall.
   ```bash
   $ sudo firewall-cmd --permanent --add-port=9100/tcp
   $ sudo firewall-cmd --reload
   ```
   ![Alt text](image-13.png)
   
   Este paso se repite con la segunda máquina a monitorear, si se sigue trabajando con Rocky Linux 9.
   
   Si se quiere monitorear otro sistema operativo, se puede emplear el uso de Docker para poder facilitar el uso del servicio de node_exporter.

### Levantar Docker para uso del node_exporter en otros sistemas operativos.
1. Creación de archivo docker-compose.yml con el siguiente contenido.
   ```bash
   version: '3'

   services:
        node_exporter:
            image: quay.io/prometheus/node-exporter:latest
            container_name: node_exporter
            ports:
                - "9100:9100"
            command:
                - '--path.rootfs=/host'
            pid: host
            restart: unless-stopped
            volumes:
                - '/:/host:ro,rslave'
   ```
   ![Alt text](image-14.png)

2. Para levantar el contenedor se usa el siguiente comando.
   ```bash
   docker compose up -d
   ```
   ![Alt text](image-15.png)

### Configuración de Grafana
1. Se debe acceder a la direción del nuestro servidor Prometheus y Grafana para poder ver que las configuraciones hechas en las máquinas a monitorear sean adecuadas.\
   En este caso la IP del servidor es 192.168.126.128 y el puerto que atiende este servicio es el 9090. Para ver la comunicación entre node_exporter y Prometheus, desde el navegador web se accede a la página de Prometheus y desde la sección de targets, se puede ver la conexión entre el server y las máquinas virtuales.

   ![Alt text](image-16.png)
   Como se logra observar, ahí están las dos máquinas virtuales, la primera con la configuración nativa y la segunda con el uso de un Docker.

2. Ahora, para acceder a Grafana se emplea la misma IP (192.168.126.128) pero con el puerto 3000, así podremos ver la interfaz de este servicio.
   ![Alt text](image-17.png)

   Al ingresar a la interfaz web se nos pedirán las credenciales pero como estás no han sido configuradas, están las por defecto que son:
    - **username**: admin
    - **Password:** admin
  
   ![Alt text](image-18.png)

   una vez ingresadas esas credenciales, se pedirá actualizar la contraseña.
   ![Alt text](image-19.png)

3. Una vez que se ha accedido, se debe agregar un Datastore, por lo que en el apartado de este se debe añadir.
   ![Alt text](image-20.png)

4. Se selecciona la opción de Prometheus
   ![Alt text](image-21.png)

5. Se configura el nombre del datasource y se añade la IP del servidor.
   ![Alt text](image-23.png)

6. Se guarda la configuración y se realiza el test.
   ![Alt text](image-24.png)

7. Para añadir un Dashboard a todos los datos recibidos del datastore se debe ir a la parte superior de la configuracion y dar clic en la opción de **build dashboard**
   ![Alt text](image-25.png)

8. Se selecciona la opción de importar dashboard.
   ![Alt text](image-26.png)

9.  Se emplea el ID 1890 para tener un dashboard muy completo y se da clic en **Load**.
    ![Alt text](image-27.png)

10. Se configura el nombre del dashboard y se selecciona la opción del datastore que se acaba de crear y se da clic en **Import**.
    ![Alt text](image-28.png)

11. Finalmente ya se puede empezar a ver todos los datos de las máquinas virtuales y en la pestaña de host se puede seleccionar alguna de las máquinas monitoreadas.
    ![Alt text](image-30.png)
    ![Alt text](image-31.png)

Finalmente, así quedaría el escenario para poder llevar a cabo el monitoreo de diversos equipos o en este caso de máquinas virtuales.