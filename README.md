# **Despliegue de una aplicación web multicapa altamente disponible y segura mediante AWS Elastic Beanstalk con AWS Academy Learner Labs**

## **Objetivo:**
AWS Elastic Beanstalk es un servicio que permite a los desarrolladores desplegar y administrar rápidamente aplicaciones en la nube de AWS sin necesidad de gestionar la infraestructura en la que se ejecutan las aplicaciones, reduciendo con ello la complejidad y sobrecarga administrativa que conlleva el aprovisionamiento, balanceo de carga, escalado y monitorización de la aplicación.

AWS Elastic Beanstalk permite desplegar aplicaciones desarrolladas en Java, PHP, .NET, Ruby, Python, Go y Node.js bajo instancias de Amazon EC2.

Es posible interactuar con AWS Elastic Beanstalk de la misma forma que con el resto de los servicios; es decir, mediante la Consola de Administración de AWS, mediante la AWS CLI y programáticamente desde los SDKs. Adicionalmente, AWS pone a nuestra disposición una interfaz de línea de comandos de más alto nivel especialmente diseñada para AWS Elastic Beanstalk desde la que podemos desplegar con facilidad y rapidez las aplicaciones.

El objetivo de esta práctica es, precisamente, dar a conocer las posibilidades de esta herramienta desplegando una sencilla aplicación web multicapa sin estado desarrollada en PHP utilizando AWS Elastic Beanstalk, junto con otros servicios para diseñar una infraestructura altamente disponible, escalable y segura en el entorno de AWS Academy Learner Labs.

## **Requerimientos:**

* Un sandbox de <em>AWS Academy Learner Lab</em> 
* Un entorno con sistema operativo Linux y con acceso programático a AWS configurado

## **Arquitectura propuesta:**

![Arquitectura](/images/arch.png)

## **Servicios utilizados:**

La solución que se implementará utilizará los siguientes servicios:
* **AWS Elastic Beanstalk**, para el aprovisionamiento y escalado de la infraestructura computacional, balanceo de carga, despliegue de la aplicación web en PHP. Este servicio aprovisionará instancias de Amazon EC2 y un balanceador de carga.
* **Amazon RDS**, para el aprovisionamiento y administración de una base de datos relacional con un motor MariaDB en alta disponibilidad.
* **Amazon VPC**, para la administración y creación de una infraestructura de red altamente disponible y segura.
* **AWS CloudFormation**, para el despliegue automatizado de la infraestructura de red anterior. Además, AWS Elastic Beanstalk también utiliza AWS CloudFormation tras las bambalinas.
* **Amazon CloudWatch Logs y Metrics**, para obtener visibilidad sobre el rendimiento y desempeño tanto de la infraestructura como de la aplicación desplegada.
* **AWS Systems Manager Parameter Store**, para proteger las credenciales de acceso a la base de datos de la aplicación.
* **Amazon S3**, para alojar las diferentes versiones de las aplicaciones que se despliegan en AWS Elastic Beanstalk.
* **Amazon CloudFront**, para brindar seguridad en el borde y rendimiento a nivel global a nuestra aplicación.

## **Instrucciones:**

### **Preparación del entorno de red**

En primer lugar, es necesario definir el entorno de red donde se lanzarán los recursos computacionales de AWS Elastic Beanstalk. Para ello, debe crearse una VPC que, siguiendo el esquema planteado tendrá asignado el bloque CIDR 10.2.0.0/16 y abarcará dos zonas de disponibilidad. Dentro de dicha VPC se crearán las siguientes subredes:

* **Publica1** (10.2.0.0/24) y **Publica2** (10.2.3.0/24), subredes públicas donde se desplegará el balanceador de carga de aplicación (ALB).
* **App1** (10.2.1.0/24) y **App2** (10.2.4.0/24), subredes privadas donde se desplegarán las instancias EC2 del entorno de AWS Elastic Beanstalk
* **BD1** (10.2.2.0/24) y **BD2** (10.2.5.0/24), subredes privadas donde se desplegarán la instancia RDS en despliegue multi-AZ, tanto la primaria como la secundaria.

Esta arquitectura nos permitirá desplegar la aplicación de tres capas, asegurando los recursos del backend, mientras se expone el balanceador de carga públicamente a través de Internet.

Como el objetivo de este repositorio no es la creación de la infraestructura de red, ésta se facilita mediante una plantilla de AWS CloudFormation llamada `vpc.yaml` incluida en el directorio `network` del repositorio. Dicha plantilla, aparte de la infraestructura de red también creará:

* Una instancia EC2 que se utilizará como bastión para poder realizar la conexión a la instancia RDS y configurar la BD de la aplicación
* Dos dispositivos NAT Gateway, uno en cada zona de disponibilidad, para permitir que los recursos en subredes privadas puedan acceder a Internet

1. Se establece la región donde se desplegará la infraestructura (recuérdese que sólo están disponibles las regiones `us-east-1` y `us-west-2`)

        REGION=us-east-1

2. Para desplegar la pila de recursos, basta con ejecutar la siguiente orden:

        aws cloudformation deploy --template-file network/vpc.yaml --stack-name vpc-stack --region $REGION

    Transcurridos unos minutos, la infraestructura de red ya estará preparada.

### **Creación de la Base de Datos**

El siguiente paso es lanzar una instancia de Amazon RDS con compatibilidad con MariaDB en un despliegue en Multi-AZ que garantice la conmutación por error automática en caso de fallo en una de las zonas de disponibilidad donde se desplegará la infraestructura.

3. Es necesario crear un grupo de subredes en Amazon RDS, compuesto por las subredes BD1 y BD2. Para ello se necesita conocer los IDs de ambas subredes, por lo que se consultan sus IDs a la pila de AWS CloudFormation:

        BD1=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`BD1`].OutputValue' --output text)

        BD2=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`BD2`].OutputValue' --output text)

    A continuación, se crea el grupo de subredes:

        aws rds create-db-subnet-group --db-subnet-group-name workshop-subnet-group --db-subnet-group-description "Grupo de subredes RDS para workshop" --subnet-ids $BD1 $BD2 --region $REGION

4. Se crea ahora el grupo de seguridad que se asignará a la instancia RDS que permita el tráfico por el puerto 3306 TCP. Previamente, se obtiene el ID de la VPC:

        VPC=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[*].Outputs[?OutputKey==`VPC`].OutputValue' --output text)

    Se crea el grupo de seguridad:

        RDS_SG=$(aws ec2 create-security-group --description "Grupo de seguridad RDS workshop" --group-name workshop-rds-sg --vpc-id $VPC --region $REGION --output text)

    Se autoriza el tráfico por el puerto 3306 TCP:

        aws ec2 authorize-security-group-ingress --group-id $RDS_SG --protocol tcp --port 3306 --cidr 0.0.0.0/0 --region $REGION

5. Por último, se crea la instancia RDS, indicando la VPC y el grupo de subredes donde se desplegará, el grupo de seguridad que tendrá asignado y el usuario y contraseña de acceso (en este caso, se ha utilizado `admin` / `adminpass`):

        USUARIO=admin

        PASSWORD=adminpass

        aws rds create-db-instance --db-name workshop --db-instance-identifier workshop-rds --engine mariadb --db-instance-class db.t4g.small --master-username $USUARIO --master-user-password $PASSWORD --vpc-security-group-ids $RDS_SG --db-subnet-group-name workshop-subnet-group --allocated-storage 20 --multi-az --tags Key=Name,Value=workshop-RDS --region $REGION

    Para esperar a que la instancia RDS se aprovisione (tardará unos 10 minutos), se ejecuta la orden:

        aws rds wait db-instance-available --region $REGION
    
6. A continuación, se procederá a crear el esquema de la base de datos que necesita la aplicación. Para ello, se realizará la conexión de forma segura mediante AWS Systems Manager Session Manager a la consola de Amazon EC2 (es necesario instalar el plugin de AWS SSM Session Manager https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) a la instancia Bastión (que se creó cuando se desplegó la infraestructura de la red). Para ello, se ejecuta la orden:

        BASTION=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Bastion`].OutputValue' --output text)
        
        aws ssm start-session --target $BASTION --region $REGION

7. Se obtiene el punto de enlace de la instancia RDS, y se accede a él desde la instancia EC2 Bastión, que viene preinstalada con el cliente de MySQL:

        DATABASE=$(aws rds describe-db-instances --region us-east-1 --query 'DBInstances[?DBInstanceIdentifier==`workshop-rds`].Endpoint.Address' --output text)

        mysql -u admin -p -h $DATABASE


    Se introduce la contraseña y se accede a la base de datos.

8. Desde el cliente `mysql` se crea la tabla necesaria para la aplicación:

        create table workshop.contactos (
            id varchar(10),
            nombre varchar(100),
            email varchar(60),
            telefono varchar(12),
            cargo varchar(30),
            fechaCreacion datetime, 
            primary key (ID)
        );

    Una vez creada el esquema de la base de datos, se cierra la sesión en la instancia bastión.

9. Por motivos de seguridad, no se introducirán las credenciales (contraseña) de la base de datos dentro del código de la aplicación, sino que se permitirá que el servicio de AWS Systems Manager Parameter Store custodie el secreto. Posteriormente, cuando se despliegue la infraestructura de la aplicación se otorgarán a las instancias EC2 los permisos temporales necesarios para que puedan acceder al valor del parámetro que almacena la contraseña. 

    Para ello, se creará un secreto en AWS Systems Manager Parameter Store con la siguiente orden, que contendrá la contraseña del usuario privilegiado de la base de datos RDS:

        NOMBRESECRETO=password
        SECRETO=adminpass

        aws ssm put-parameter --name $NOMBRESECRETO --value $SECRETO --type SecureString --region $REGION

    La sentencia anterior crea un parámetro secreto cifrado mediante la clave predeterminada del servicio AWS Key Management Service (KMS). El nombre de este parámetro se le pasará a la aplicación como una variable de entorno a la que accederá en tiempo de ejecución.

### **Creación de un entorno para la aplicación mediante AWS Elastic Beanstalk**

La aplicación que se desplegará se encuentra dentro de este repositorio. Se trata de una sencilla aplicación en PHP que realiza operaciones CRUD sobre una tabla de contactos. Para garantizar la privacidad y seguridad de la conexión a la base de datos, todos los parámetros de configuración se pasarán mediante las siguientes variables de entorno:
* **AWS_REGION**. Región donde se encuentra almacenado el secreto de AWS Systems Manager Parameter Store que contiene la contraseña de la BD
* **RDS_HOSTNAME**. Contendrá el punto de enlace de la instancia RDS que se ha creado
* **RDS_USER_NAME**. Contendrá el nombre de usuario creado para la conexión a la BD
* **RDS_DB_SECRET**. Contendrá el nombre del secreto de AWS Systems Manager
Parameter Store donde se custodia la contraseña de conexión a la BD
* **RDS_DB_NAME**. Nombre de la base de datos creada en la instancia RDS, en el caso de esta práctica es `workshop`, pero podría ser cualquier otro

En el código de la función `conectar_mariadb()` del archivo `funciones.php` se muestra cómo acceder a las variables de entorno y al secreto de AWS SSM Parameter Store para configurar la cadena de conexión.

#### **AWS Elastic Beanstalk**

AWS Elastic Beanstalk es un servicio PaaS (Platform as a Service) que permite a los desarrolladores desplegar y administrar aplicaciones en la nube de AWS de una forma ágil y sencilla. Los desarrolladores sólo tienen que cargar el código de la aplicación y dejar que el servicio se encargue de forma automática de los detalles del despliegue, como el aprovisionamiento de la infraestructura subyacente y su capacidad, el balanceo de carga, el escalado automático y la monitorización del estado de la aplicación.

AWS Elastic Beanstalk es un servicio compatible con aplicaciones desarrolladas en Go, Java, PHP, .NET, Node.js, Python, Docker e incluso resulta posible utilizar plataformas personalizadas.

AWS Elastic Beanstalk dispone de una CLI personalizada (EB CLI), que se va a utilizar durante este workshop. Para ello debe descargarse e instalarse en el entorno, siguiendo las instrucciones del siguiente enlace:

https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html#eb-cli3-install.scripts

Para comprender cómo funciona AWS Elastic Beanstalk, debemos conocer los siguientes conceptos:

* **Aplicación**. Es una colección lógica de componentes de AWS Elastic Beanstalk, incluyendo entornos, versiones y configuraciones de entornos. Se podría decir que, conceptualmente, es similar a una carpeta.

* **Versión de Aplicación**. Se refiere a una iteración específica y etiquetada de código desplegable para una aplicación web. Una versión de la aplicación apunta a un objeto de un bucket de Amazon S3 que contiene el código desplegable (por ejemplo, un archivo WAR en Java). Las aplicaciones pueden tener múltiples versiones y cada versión es única. Un entorno puede tener desplegada una única versión de la aplicación en un momento determinado.

* **Entorno**. Es una colección de recursos de AWS que ejecutan una versión de una aplicación determinada. Una versión de una aplicación puede ejecutarse a la vez en diferentes entornos. Cuando se crea un entorno, AWS Elastic Beanstalk aprovisiona los recursos e infraestructura necesaria para ejecutar la versión de la aplicación elegida.

* **Configuración del entorno**. Es una colección de parámetros y configuraciones que definen cómo se comporta el entorno y sus recursos asociados. Cuando se actualiza la configuración del entorno, el servicio AWS Elastic Beanstalk aplica automáticamente los cambios sobre los recursos existentes o elimina y despliega nuevos recursos.

10. En primer lugar, se creará la aplicación en AWS Elastic Beanstalk. Para ello, desde el directorio del repositorio se ejecuta la orden:

        eb init --region $REGION -v

    Se iniciará entonces una asistente que preguntará sobre el nombre de la aplicación:

        Enter Application Name
        (default is "aws-academy-elasticbeanstalk"): workshop

    La CLI de AWS Elastic Beanstalk detectará la aplicación escrita en PHP, y se selecciona el entorno de ejecución correspondiente a la versión PHP 8.1:

        It appears you are using PHP. Is this correct?
        (Y/n): Y
        Select a platform branch.
        1) PHP 8.1 running on 64bit Amazon Linux 2
        2) PHP 8.0 running on 64bit Amazon Linux 2
        3) PHP 7.4 running on 64bit Amazon Linux 2 (Deprecated)
        (default is 1): 1

    Se continúa el asistente tal y como se muestra, utilizando el par de claves `vockey` de AWS Academy Learner Lab:

        Do you wish to continue with CodeCommit? (Y/n): n
        Do you want to set up SSH for your instances?
        (Y/n): Y
        Select a keypair.
        1) vockey
        2) [ Create new KeyPair ]
        (default is 1): 1

11.  Una vez concluido el asistente, ya se habrá creado la aplicación llamada `workshop` y ahora se necesitará crear un entorno para desplegarla. El entorno de AWS Elastic Beanstalk que se creará estará compuesto por:

* Un **grupo de autoescalado de instancias EC2**, con un mínimo de 2 instancias, pudiendo escalar hasta 6 instancias, distribuidas entre las dos zonas de disponibilidad habilitadas en nuestra infraestructura de red. Estas instancias EC2 alojarán las versiones de nuestra aplicación y se desplegarán dentro de las subredes `App1` y `App2`
* Un **balanceador de carga de aplicación** (ALB), desplegado dentro de las subredes `Publica1` y `Publica2`, y que se encargará de distribuir el tráfico HTTP entrante entre las diferentes instancias EC2 de la capa de aplicación

Antes de seguir adelante y crear el entorno, es necesario obtener las subredes públicas donde se desplegará el ALB, así como las subredes privadas donde operarán las instancias EC2:

    PUB1=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Publica1`].OutputValue' --output text)

    PUB2=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Publica2`].OutputValue' --output text)

    APP1=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`App1`].OutputValue' --output text)

    APP2=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`App2`].OutputValue' --output text)

Para crear el entorno de la aplicación, se ejecuta las órdenes siguientes:

    DATABASE=$(aws rds describe-db-instances --region $REGION --query 'DBInstances[?DBInstanceIdentifier==`workshop-rds`].Endpoint.Address' --output text)

    ENTORNOCNAME=jevs-workshop-1234

    eb create devel -c $ENTORNOCNAME -sr LabRole -ip LabInstanceProfile --min-instances 2 --max-instances 6 -i t4g.micro -k vockey --envvars AWS_REGION=$REGION,RDS_HOSTNAME=$DATABASE,RDS_USER_NAME=admin,RDS_DB_NAME=workshop,RDS_DB_SECRET=$NOMBRESECRETO --vpc.ec2subnets $APP1,$APP2 --elb-type application --vpc.elbpublic --vpc.id $VPC --vpc.elbsubnets $PUB1,$PUB2 --region $REGION

La instrucción anterior:

* Crea un entorno llamado `devel` cuyo CNAME es `jevs-workshop-1234.elasticbenastalk.com` (como ejemplo, puede elegirse cualquier subdominio de `elasticbeanstalk.com` que no exista).
* Se permite al servicio AWS Elastic Beanstalk que asuma el rol de IAM `LabRole` (generado por AWS Academy Learner Lab) para que pueda aprovisionar los recursos necesarios de la infraestructura (instancias EC2, ALB, ...).
* Por otra parte, las instancias que alojarán la aplicación serán de tipo `t4g.micro` y se les asignará el perfil de instancia `LabInstanceProfile` (generado por AWS Academy Learner Lab) que permitirá a las instancias acceder al secreto almacenado en AWS SSM Parameter Store.
* Los números mínimo y máximo de instancias del grupo de autoescalado están indicados en los parámetros `min-instances` y `max-instances`
* Se permitirá el acceso por SSH a las instancias EC2 utilizando el par de claves `vockey`, generado en el AWS Academy Learner Lab
* El entorno tendrá acceso a las variables indicadas en el parámetro `envvars`, es decir, `AWS_REGION`,`RDS_HOSTNAME`,`RDS_USER_NAME`,`RDS_DB_NAME` y `RDS_DB_SECRET`
* Se indica mediante el parámetro `vpc.elbsubnets` las subredes en las que se desplegará el ALB
* Se indica mediante el parámetro `vpc.ec2subnets` las subredes donde se desplegarán las instancias EC2 del entorno

Tras el lanzamiento de la orden, irán apareciendo en el <em>log</em> de la consola las notificaciones sobre los eventos de creación de la infraestructura del entorno. En algún momento aparecerá la creación de dos grupos de seguridad, tal y como se muestra:

    2022-11-22 10:44:50 INFO Created security group named: sg-02adcb0edfe1c6fd6
    2022-11-22 10:45:06 INFO Created security group named: sg-033fcd0851f1b4a44

El segundo de ellos es el grupo de seguridad que se asignará a las instancias EC2; el primero es el grupo de seguridad asignado al ALB. Se almacena el primero de ellos, que se utilizará a continuación:

    EC2SG=sg-033fcd0851f1b4a44

12. Anteriormente, se creó un grupo de seguridad sobre la instancia RDS,
que permitía el tráfico de entrada por el puerto 3306 TCP desde cualquier ubicación (`0.0.0.0/0`). Siguiendo las buenas prácticas, se limitará ahora el tráfico entrante para que sólo se acepte aquél que provenga desde el
grupo de seguridad de la capa de aplicación de nuestra infraestructura. Para ello, se ejecutan las siguientes órdenes para revocar los permisos anteriores y otorgar los permisos restringidos:

        aws ec2 revoke-security-group-ingress --group-id $RDS_SG --port 3306 --protocol tcp --cidr 0.0.0.0/0 --region $REGION

        aws ec2 authorize-security-group-ingress --group-id $RDS_SG --protocol tcp --port 3306 --source-group $EC2SG --region $REGION

13. Por último, se puede abrir el entorno `devel` mediante la orden:

        eb open devel --region $REGION

    Se abrirá una ventana del navegador donde se podrá acceder a la aplicación.



