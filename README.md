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

    Tras esperar unos minutos, la instancia RDS quedará aprovisionada.
    
6. A continuación, se procederá a crear el esquema de la base de datos que necesita la aplicación. Para ello, se realizará la conexión de forma segura mediante AWS Systems Manager Session Manager a la consola de Amazon EC2 (es necesario instalar el plugin de AWS SSM Session Manager https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) a la instancia Bastión (que se creó cuando se desplegó la infraestructura de la red). Para ello, se ejecuta la orden:

        BASTION=$(aws cloudformation describe-stacks --stack-name vpc-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Bastion`].OutputValue' --output text)
        
        aws ssm create-session --target $BASTION --region $REGION

7. La instancia EC2 Bastión viene preinstalada con el cliente de MySQL por lo que ejecutamos la siguiente orden, sustituyendo el valor resaltado por el nombre de usuario establecido anteriormente:

    






