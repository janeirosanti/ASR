# Práctica 2 ASR - ENTREGA Santiago Janeiro Catoira
## **1a Solución: creación de máquina de salto - 4 puntos**
### Montar una máquina de salto para poder acceder a nuestro servidor web. Esta máquina se debería encender y apagar cada vez que se quiera modificar algo del servidor web.
### Exponer únicamente en ambos servidores lo mínimo indispensable con reglas de firewall (firewall capa 4).

#### 1) Para ello debemos crear una VPC Network, con su respectiva subnet. En este caso le he asignado el rango IP 10.0.0.0/24. Toda la práctica se hará en la región europe-west1.

![VPC_network1](https://github.com/janeirosanti/ASR/assets/47990780/12066672-c9f2-4371-8c49-72d3f5126326)

#### 2) Tras ello para llevar a cabo la práctica nos harán falta dos instancias de máquina virtual (VM Instances). La máquina donde se alojará el servidor web y una máquina de salto usada para no exponer el puerto 22 de nuestro servidor a IP externas a las de nuestra intranet. En este caso ambas tendrán una dirección IP privada dentro de la subred creada anteriormente y también hará falta que tengan una IP pública. La IP pública del servidor será a la que se harán las llamadas http que quieran hacer uso del servicio y la IP pública de la máquina de salto servirá para acceder mediante ssh desde local y así poder después saltar también por ssh a la máquina servidor.
    ![maquina_servidor1](https://github.com/janeirosanti/ASR/assets/47990780/70cfe17d-625f-46f8-aad8-6e5061e9bb1f)
![maquina_salto1](https://github.com/janeirosanti/ASR/assets/47990780/61b24290-52d2-4377-9f6e-9519c84cbca8)

#### Al no haber usado la opción de tener una IP fija cada vez que apagas las máquinas dejan de ocupar una IP pública y por lo tanto esta puede cambiar (lo más normal es que lo haga) a la hora de volver a iniciarla.

![vm_instances](https://github.com/janeirosanti/ASR/assets/47990780/d3ed0898-60b8-4bab-b7d3-9cabc3697b3d)


#### 3) Ahora toca configurar las reglas de firewall para permitir los accesos mencionados anteriormente. Lo primero que debemos hacer es quitar cualquier Firewall Rule que venga por defecto y agregaremos 3 regas de firewall.
   ##### 3.1) La primera regla será la de ssh externo, que permite a equipos de fuera de la intranet conectarse al puerto 22 de nuestra máquina de salto. Pero no a cualquiera, solo las IP en este caso de la Universidad y de mi casa. ![firewall_rule_ssh_externo_maquina_salto](https://github.com/janeirosanti/ASR/assets/47990780/db1a59d1-26f3-421c-8c99-407367653502)
  ##### 3.2) La segunda regla será la de ssh interno, esta regla es la que nos permite escalar del servidor de salto a la máquina del servidor mediante ssh. Como aquí no queremos exponer el puerto 22 de la máquina donde está el servidor a IPs públicas, únicamente damos acceso a las IPs de nuestra subred privada. ![firewall_rule_ssh_interno_server](https://github.com/janeirosanti/ASR/assets/47990780/27a8197c-e7d9-4ff8-855e-1798c0b34380)
  ##### 3.3) La última regla de firewall a configurar es la regla http mediante la cual podremos desde local conectarnos en el puerto 80 de tcp a la máquina del servidor para poder llevar a cabo peticiones http. Aquí de nuevo vuelvo a introducir que solamente puedan realizar estas peticiones mis IP del NAT de mi casa y de la Universidad.![firewall_rule_http_conection_server](https://github.com/janeirosanti/ASR/assets/47990780/4ccc19cd-e451-48de-b98e-ce233f337f60)
#### 4) Además de lo anterior debemos añadir nuestras claves ssh para poder conectarnos. ![claves_ssh](https://github.com/janeirosanti/ASR/assets/47990780/f4a4555d-0610-42c3-ad32-d2757573fc30)

#### 5) Una vez hecho esto debemos comprobar que podemos conectarnos a la máquina del servidor para poder correr el servidor nginx allí.
   ##### 5.1) Para ello comenzamos conectándonos al jump-server mediante ssh. ![ssh_servidor_salto](https://github.com/janeirosanti/ASR/assets/47990780/42fe214f-3df2-472b-a2de-9c01023e7853)
   ##### 5.2) Una vez en la máquina de salto podemos conectarnos directamente por ssh a la máquina server usando la IP privada de esta.![ssh-servidor-web](https://github.com/janeirosanti/ASR/assets/47990780/8d1d4a47-0d1e-4713-b43e-bb3deb30ba85)
#### 6) Dentro de la máquina del servidor ejecutamos un servidor nginx con el código que podemos encontrar en el startup.sh de la práctica anterior.
   ![nginx_google_cloud](https://github.com/janeirosanti/ASR/assets/47990780/9ff6e136-9b30-4ff4-9c2d-2c003f1baad5)
#### 7) Si hubiésemos dejado el html por defecto debería habernos salido algo así.![conexion_nginx_1](https://github.com/janeirosanti/ASR/assets/47990780/d63ef984-40c1-44b0-b533-b5d834c7f049)


## 2da mejora solución: introducción a los WAF - Web Application Firewall (firewall capa 7) - 4 puntos

### Convertir nuestro servidor web para que no tenga ip pública, y montar un balanceador con servicio de WAF haciendo HTTPS offloading. ¿Qué ventajas e incovenientes tiene hacer https offloading en el balanceador? ¿Qué pasos adicionales has tenido que hacer para que la máquina pueda salir a internet para poder instalar el servidor nginx?
### Proteger nuestra máquina de ataques SQL Injection, Cross Syte Scripting y restringir el tráfico sólo a paises de confianza de la UE implantando un WAF a nuestro balanceador.

#### 1) Para esta mejora vamos a cambiar ciertas cosas de nuestra arquitectura. Lo primero que haremos será crear una nueva instancia de máquina virtual donde alojar el servidor. Esto es necesario ya que la nueva máquina debe no tener IP pública ya que para hacerle llegar la petición http esto se hará mediante un balanceador de carga con servicio WAF haciendo HTTPS offloading.
##### 1.1) **Ventajas del https offloading**
   ###### 1.1.1) Quita carga computacional de cifrado y descifrado de las peticiones https al servidor.   
   ###### 1.1.2) Al descifrar las peticiones en el balanceador esto puede permitir un mayor rendimiento haciendo uso de compresión de datos o caching.
   ###### 1.1.3) Obtenemos una mayor escalabilidad. Se puede redirigir inteligentemente el tráfico a las diferentes máquinas en las que podamos tener un servicio y de esta manera podemos manejar un mayor número de solicitudes concurrentes.
##### 1.2) **Desventajas del https offloading**
   ###### 1.2.1) Si algún atacante consigue entrar en la subred privada conseguirá acceso a toda la información limpia sin cifrar.
   ###### 1.2.2) Mayor coste de hardware, esto puede implicar que el balanceador de carga necesite ciertas funciones específicas para manejar el cifrado y descifrado (sobretodo la sobrecarga computacional que supone).
#### 2) Configuramos la nueva máquina donde irá alojado el servidor, en este caso solo tendrá la IP privada de nuestra subnet, la 10.0.0.4.    ![nuevo_server](https://github.com/janeirosanti/ASR/assets/47990780/4965f9d0-5078-4f1d-be08-99cf77e1a1c8)

#### 3) En cuanto a las reglas de firewall, al contrario que antes, ahora no debemos permitir que el tráfico http de fuera de nuestra intranet llegue al puerto 80 de nuestro servidor por lo que únicamente añadiremos el tag de **ssh interno** visto anteriormente a nuestra máquina servidor.
#### 4) Como el servidor no tiene IP pública y no posee acceso a internet debemos habilitar el NAT.
![cloud_nat](https://github.com/janeirosanti/ASR/assets/47990780/e792a575-fbcf-414b-bc74-bae2c0538e43)
#### 5) Ahora debemos generar los certificados para poder firmar peticiones https. Para ello debemos generar nuestro propio .csr y nuestra clave -key. ![clave_privada y csr](https://github.com/janeirosanti/ASR/assets/47990780/2c2e71db-6dae-410a-b26a-2dffcf5cbd0d)
##### 5.1) En este momento creamos nuestra propia CA. ![clave_privada_CA](https://github.com/janeirosanti/ASR/assets/47990780/b1e292dd-2bbe-4ab3-ba80-fabc2541fd44)
##### 5.2) Con la que firmamos nuestro propio certificado para que sea válido. Aunque los navegadores no confíen porque saben que no es una entidad de certificación registrada.![cert_autosigned_rootCA crt](https://github.com/janeirosanti/ASR/assets/47990780/621ca912-4a69-4ddb-9622-3f580f862db1)
#### 6) Ahora debemos crear un grupo de instancias a las que se dirija el balanceador.![instance-group](https://github.com/janeirosanti/ASR/assets/47990780/5b2ac160-d21b-4171-8842-0158a09a7991)
##### 6.1) Debemos también crear un health check en la conexión por el puerto 80 como regla de firewall y además en el backend. ![healthcheck_firewall_policies](https://github.com/janeirosanti/ASR/assets/47990780/9b8cb651-ad86-4476-93b7-cd7aad708b05)
#### 7) En este momento ya podemos tratar de acceder al servidor apuntando a la IP pública que tiene el Load Balancer. ![advertencia_navegador](https://github.com/janeirosanti/ASR/assets/47990780/0612b4ab-b45a-4e56-91d8-a7a28368c6ff)
#### 8) En un principio pone que no es seguro por lo que se explicó en el punto 5.2 pero si seguimos hasta llegar a la web encontramos la pantalla de bienvenida de nginx. ![resultado_final](https://github.com/janeirosanti/ASR/assets/47990780/a848374f-666d-4a8f-9b8c-3d472d81a78d)
##### 8.1) Lo que hemos conseguido es dirigir tráfico https cifrado con nuestro própio certificado al puerto 443 del balanceador de carga. Este descifra el contenido y lo reenvía también por http al puerto 80 del servidor.
#### 9) Para poder protegernos de ataques de SQL Injection, Cross Syte Scripting y restringir el tráfico sólo a paises de confianza de la UE implantando un WAF a nuestro balanceador debemos añadir una serie de security policies para que se cumplan estas condiciones. Esto se configura en Cloud Armor para Google Cloud. (En la policy de los países e la captura se restringe únicamente a España pero para añadir a todos los países de la UE no haría falta más que añadir sus respectivos códigos de país de 2 letras como 'ES' para el caso de España.) ![security_policy](https://github.com/janeirosanti/ASR/assets/47990780/90e2999e-627e-46bb-90bb-e5c653f04b4a)

## 3ra mejora solución: zero trust - 1 punto
### Cifrar el contenido web también dentro del cloud y quitar el HTTPS offloading.

#### Para esta última parte buscamos tráfico https dentro de la subred, esto soluciona una de los inconvenientes del HTTPs offloading que vimos en el apartado anterior ya que aunque las conexiones dentro de la subred estén siendo sniffed se toparían con tráfico cifrado. Para ello el tráfico entre el balanceador y el servidor debe ser obviamente por el puerto 443 y debemos también cambiar la configuración de nginx para que trabaje en el puerto 443.
#### 1) Debemos modificar el instance group para enviar tráfico https al tráfico 443 de la máquina server. ![Modificacion_instance_group](https://github.com/janeirosanti/ASR/assets/47990780/79796992-a7c4-46de-9ab9-e6dc2c6215f4)
#### 2) Ahora nos toca cambiar la configuración del Load Balancer en su parte de backend para que se pueda comunicar por el puerto 443. ![cambio_back_balancer_allow_443](https://github.com/janeirosanti/ASR/assets/47990780/5afa349d-4f78-4070-966e-8757fe702d85)
#### 3) Hacemos lo propio con el Health Check del balanceador.![cambiar_health_check_back_balancer](https://github.com/janeirosanti/ASR/assets/47990780/43fda7fc-d347-4f25-89e2-77f7f3cbe11d)

#### 3) Por último en lo respectivo a la configuración del entorno cloud debemos cambiar la Firewall policy de Health Check también cambiando por el puerto 443. ![firewall_healthchecks_443](https://github.com/janeirosanti/ASR/assets/47990780/f20f43c8-7495-4ec9-b6a7-05708eb99fd7)

#### 4) Solo faltaría cambiar el fichero de configuración de nginx. Pero antes debmos enviar el certificado y la clave ssl a la máquina servidor. Para eso hice uso del comando scp primero llevándolos de local a la máquina de salto. ![scp_archivos](https://github.com/janeirosanti/ASR/assets/47990780/30631902-7f5d-4a09-b21b-7f5562e283f4)![recepcion_archivos](https://github.com/janeirosanti/ASR/assets/47990780/94cf2281-c566-409b-8a69-ad834fdb6b20)

#### Y posteriormente de la máquina de salto a la máquina servidor. ![archivos_enviados_server](https://github.com/janeirosanti/ASR/assets/47990780/e1ca9423-a11f-4787-8097-a9cb847b7ac0)![llegan_server](https://github.com/janeirosanti/ASR/assets/47990780/28a54418-7043-46e3-a7f0-42b9dae2fc8d)

#### 5) Para terminar faltaría cambiar la configuración de nginx por la siguiente. 

![config_nginx](https://github.com/janeirosanti/ASR/assets/47990780/5e2bfd36-3884-4012-abc4-ea1e002c93a5)
#### 6) Comprobamos que funciona. ![resultado_final](https://github.com/janeirosanti/ASR/assets/47990780/fe9f7fe2-c1bc-45b2-82e2-fcb9f0a4e3a8)

#### 7) Otra manera de comprobarlo. ![funciona_final](https://github.com/janeirosanti/ASR/assets/47990780/7eae569b-c2ee-4bac-a2fa-736a9c80661a)

#### Ahí podemos ver cómo ya se nos indica que curl no puede verificar la identidad de la aturidad del certificado y por lo tanto no nos muestra la página.

## 4ta mejora solución - 1 punto
## ¿Qué otras mejoras se te ocurrirían para mejorar la seguridad o disponibilidad del servidor web? (No hace falta implementarlas)

### 1) Que el balanceador pueda abrir o cerrar más de una máquina (servicios de escalabilidad automática) corriendo el servidor según la demanda. De esta manera nos adaptamos al posible tráfico que tenga el servidor, ya sea en momentos de alta demanda como en momentos con una demanda casi nula donde solo haga falta una instancia.
### 2) Configuración de HSTS: Permite HSTS en tu servidor para forzar a los navegadores a utilizar siempre conexiones HTTPS, reduciendo así la exposición a ataques.
### 3) Geodistribución: Distribuir los diferentes servidores entre diferentes zonas geográficas de manera que se reduzca la latencia.















