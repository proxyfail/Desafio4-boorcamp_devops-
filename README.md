1. Preparación del Entorno AWS

Antes de proceder con el despliegue de la infraestructura, es fundamental asegurar una configuración inicial adecuada y segura.
1.1. Gestión de Identidad y Acceso (IAM)

Como buena práctica de seguridad en AWS, no se recomienda utilizar la cuenta raíz (root) para operaciones diarias. Es preferible operar con un usuario IAM que posea los permisos necesarios. Para este tutorial, asumiremos un usuario con permisos de administrador para simplificar el proceso.

Acción: Verificar o configurar un usuario IAM en la AWS CLI con los permisos adecuados.

Script de Verificación/Configuración:
Bash

#!/bin/bash

# --- Variables de configuración inicial ---
AWS_REGION="us-west-2" # Defina su región de AWS preferida, ej. us-east-1, sa-east-1
IAM_USER_NAME="devops-rds-admin" # Nombre de su usuario IAM

echo "### 1.1. Configuración de IAM ###"

# Comprobamos si el usuario IAM ya existe. Si no, lo creamos.
if aws iam get-user --user-name "$IAM_USER_NAME" &> /dev/null; then
    echo "Usuario IAM '$IAM_USER_NAME' ya existe. Continuando."
else
    echo "Creando usuario IAM: $IAM_USER_NAME"
    aws iam create-user --user-name "$IAM_USER_NAME"
    echo "Usuario '$IAM_USER_NAME' creado."

    echo "Creando clave de acceso para '$IAM_USER_NAME'. Guarde estas credenciales de forma segura."
    aws iam create-access-key --user-name "$IAM_USER_NAME"

    echo "Adjuntando la política 'AdministratorAccess' al usuario '$IAM_USER_NAME'."
    aws iam attach-user-policy \
        --user-name "$IAM_USER_NAME" \
        --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
    echo "Política 'AdministratorAccess' adjuntada."
fi

echo -e "\n### Nota Importante ###"
echo "Asegúrese de que su AWS CLI esté configurada para utilizar este usuario."
echo "Puede configurarlo con: aws configure --profile $IAM_USER_NAME"
echo "O si desea que sea el perfil por defecto: aws configure."
echo "La región configurada para este script es: $AWS_REGION"
aws configure set default.region $AWS_REGION # Configura la región por defecto para este script
echo "Verificando la región actual de la CLI: $(aws configure get region)"

1.2. Selección de Región

La consistencia regional es un aspecto crítico. Todos los recursos que se desplegarán (VPC, subredes, RDS) deben residir en la misma región para garantizar su correcta interconexión y funcionamiento.

Acción: Confirmar que la región de AWS definida en sus scripts es la deseada.

Script de Verificación: (Ya incluido en el script anterior como parte de la configuración global de la CLI).
2. Diseño e Implementación de la Infraestructura de Red (VPC)

La VPC constituye el entorno de red aislado dentro de AWS, esencial para alojar y controlar el acceso a la instancia RDS. Para este escenario, se requiere una VPC con al menos dos subredes públicas distribuidas en diferentes Zonas de Disponibilidad.
2.1. Creación de la VPC Principal

Se establecerá una VPC con un bloque CIDR amplio (/16), lo que permite flexibilidad para futuras expansiones y la creación de múltiples subredes internas.

Acción: Crear la VPC denominada tutorial-vpc.

Script:
Bash

#!/bin/bash

# --- Parámetros de configuración de la VPC ---
VPC_CIDR="10.0.0.0/16" # Bloque CIDR para la VPC
VPC_NAME="tutorial-vpc" # Nombre de la VPC

echo "### 2.1. Creación de la VPC Principal ###"

# Creación de la VPC
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block "$VPC_CIDR" \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$VPC_NAME}]" \
    --query 'Vpc.VpcId' --output text)

if [ -z "$VPC_ID" ]; then
    echo "ERROR: Fallo al crear la VPC."
    exit 1
fi

echo "VPC '$VPC_NAME' creada con ID: $VPC_ID"

# Se guarda el VPC_ID en un archivo para su uso en scripts posteriores.
echo "$VPC_ID" > vpc_id.txt

2.2. Configuración de Acceso a Internet (Internet Gateway y Tabla de Enrutamiento)

Para permitir que el tráfico de internet alcance la base de datos, la VPC necesita un Internet Gateway (IGW) que actúe como punto de entrada/salida, y una tabla de enrutamiento que dirija el tráfico por defecto hacia este IGW.

Acción: Crear el Internet Gateway, asociarlo a la VPC y configurar la tabla de enrutamiento principal.

Script:
Bash

#!/bin/bash

# --- Dependencias de ID de VPC ---
VPC_ID=$(cat vpc_id.txt) # Obtiene el ID de la VPC del archivo

IGW_NAME="tutorial-igw" # Nombre para el Internet Gateway

echo "### 2.2. Configuración de Acceso a Internet (Internet Gateway y Tabla de Enrutamiento) ###"

if [ -z "$VPC_ID" ]; then
    echo "ERROR: El ID de la VPC no se encontró. Asegúrese de haber ejecutado el script de creación de VPC previamente."
    exit 1
fi

# Creación del Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=$IGW_NAME}]" \
    --query 'InternetGateway.InternetGatewayId' --output text)

if [ -z "$IGW_ID" ]; then
    echo "ERROR: Fallo al crear el Internet Gateway."
    exit 1
fi

echo "Internet Gateway '$IGW_NAME' creado con ID: $IGW_ID"

# Asociación del Internet Gateway a la VPC
echo "Asociando el Internet Gateway $IGW_ID a la VPC $VPC_ID..."
aws ec2 attach-internet-gateway \
    --vpc-id "$VPC_ID" \
    --internet-gateway-id "$IGW_ID"
echo "Internet Gateway asociado correctamente."

# Obtención del ID de la tabla de enrutamiento principal de la VPC
MAIN_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' --output text)

if [ -z "$MAIN_RT_ID" ]; then
    echo "ERROR: No se pudo encontrar la tabla de enrutamiento principal para la VPC $VPC_ID."
    exit 1
fi

echo "ID de la tabla de enrutamiento principal de la VPC: $MAIN_RT_ID"

# Creación de una ruta por defecto (0.0.0.0/0) apuntando al Internet Gateway
echo "Creando ruta por defecto (0.0.0.0/0) hacia el Internet Gateway $IGW_ID en la tabla $MAIN_RT_ID..."
aws ec2 create-route \
    --route-table-id "$MAIN_RT_ID" \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id "$IGW_ID"
echo "Ruta creada con éxito."

# Guardar los IDs del IGW y la tabla de enrutamiento para pasos posteriores.
echo "$IGW_ID" > igw_id.txt
echo "$MAIN_RT_ID" > main_rt_id.txt

2.3. Creación de Subredes Públicas Multi-AZ

Para garantizar la alta disponibilidad de la instancia RDS, es necesario contar con al menos dos subredes ubicadas en diferentes Zonas de Disponibilidad. Estas subredes se configurarán como públicas para permitir el acceso desde internet en el contexto de este tutorial.

Acción: Crear dos subredes con bloques CIDR /24 en Zonas de Disponibilidad distintas y habilitar la asignación automática de IPs públicas.

Script:
Bash

#!/bin/bash

# --- Parámetros de configuración de las subredes ---
VPC_ID=$(cat vpc_id.txt) # ID de la VPC
MAIN_RT_ID=$(cat main_rt_id.txt) # ID de la tabla de enrutamiento principal

SUBNET_CIDR_1="10.0.0.0/24"
SUBNET_AZ_1="us-west-2a" # Seleccione una Zona de Disponibilidad para su región (ej. us-east-1a)
SUBNET_NAME_1="Tutorial-public-1"

SUBNET_CIDR_2="10.0.1.0/24"
SUBNET_AZ_2="us-west-2b" # Debe ser una Zona de Disponibilidad DIFERENTE a la anterior
SUBNET_NAME_2="Tutorial-public-2"

echo "### 2.3. Creación de Subredes Públicas Multi-AZ ###"

if [ -z "$VPC_ID" ] || [ -z "$MAIN_RT_ID" ]; then
    echo "ERROR: Faltan los IDs de la VPC o la Route Table. Asegúrese de ejecutar los scripts anteriores."
    exit 1
fi

# Creación de la primera Subred Pública
SUBNET_ID_1=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$SUBNET_CIDR_1" \
    --availability-zone "$SUBNET_AZ_1" \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$SUBNET_NAME_1}]" \
    --query 'Subnet.SubnetId' --output text)

if [ -z "$SUBNET_ID_1" ]; then
    echo "ERROR: Fallo al crear la Subred 1."
    exit 1
fi
echo "Subred 1 '$SUBNET_NAME_1' ($SUBNET_CIDR_1) creada con ID: $SUBNET_ID_1 en AZ: $SUBNET_AZ_1"

# Habilitar la asignación automática de IP pública para la Subred 1
echo "Habilitando la asignación automática de IP pública para Subred 1..."
aws ec2 modify-subnet-attribute \
    --subnet-id "$SUBNET_ID_1" \
    --map-public-ip-on-launch
echo "Asignación de IP pública habilitada para Subred 1."


# Creación de la segunda Subred Pública
SUBNET_ID_2=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$SUBNET_CIDR_2" \
    --availability-zone "$SUBNET_AZ_2" \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$SUBNET_NAME_2}]" \
    --query 'Subnet.SubnetId' --output text)

if [ -z "$SUBNET_ID_2" ]; then
    echo "ERROR: Fallo al crear la Subred 2."
    exit 1
fi
echo "Subred 2 '$SUBNET_NAME_2' ($SUBNET_CIDR_2) creada con ID: $SUBNET_ID_2 en AZ: $SUBNET_AZ_2"

# Habilitar la asignación automática de IP pública para la Subred 2
echo "Habilitando la asignación automática de IP pública para Subred 2..."
aws ec2 modify-subnet-attribute \
    --subnet-id "$SUBNET_ID_2" \
    --map-public-ip-on-launch
echo "Asignación de IP pública habilitada para Subred 2."

# Las nuevas subredes se asocian por defecto a la tabla de enrutamiento principal.
# Puede verificar la asociación con:
# aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$SUBNET_ID_1"

# Se guardan los IDs de las subredes para el siguiente paso.
echo "$SUBNET_ID_1" > subnet_id_1.txt
echo "$SUBNET_ID_2" > subnet_id_2.txt

2.4. Configuración del Security Group para RDS

El Security Group actúa como un firewall de estado a nivel de instancia, controlando el tráfico de entrada y salida. Para este escenario, se configurará una regla de entrada específica para el puerto de MariaDB (3306) permitiendo el acceso desde cualquier dirección IP (0.0.0.0/0).

Acción: Identificar el Security Group por defecto de la VPC y añadir la regla de entrada necesaria.

Script:
Bash

#!/bin/bash

# --- Parámetros de configuración del Security Group ---
VPC_ID=$(cat vpc_id.txt) # ID de la VPC
DB_PORT="3306" # Puerto estándar para MariaDB/MySQL
SOURCE_CIDR="0.0.0.0/0" # Permite acceso desde cualquier IP (solo para fines de tutorial)

echo "### 2.4. Configuración del Security Group para RDS ###"

if [ -z "$VPC_ID" ]; then
    echo "ERROR: El ID de la VPC no se encontró. Asegúrese de ejecutar el script de creación de VPC previamente."
    exit 1
fi

# Obtención del ID del Security Group por defecto de la VPC
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" \
    --query 'SecurityGroups[0].GroupId' --output text)

if [ -z "$SECURITY_GROUP_ID" ]; then
    echo "ERROR: No se pudo encontrar el Security Group por defecto para la VPC $VPC_ID."
    exit 1
fi

echo "ID del Security Group por defecto de la VPC: $SECURITY_GROUP_ID"

# Adición de la regla de entrada para MariaDB (Puerto 3306) desde cualquier origen
echo "Añadiendo regla de entrada para TCP/$DB_PORT desde $SOURCE_CIDR al Security Group $SECURITY_GROUP_ID..."
aws ec2 authorize-security-group-ingress \
    --group-id "$SECURITY_GROUP_ID" \
    --protocol tcp \
    --port "$DB_PORT" \
    --cidr "$SOURCE_CIDR" \
    --description "Permitir acceso a MariaDB desde internet para tutorial"

echo "Regla de entrada añadida con éxito."

# Se guarda el ID del Security Group para pasos posteriores.
echo "$SECURITY_GROUP_ID" > security_group_id.txt

3. Configuración de Amazon RDS

Con la infraestructura de red ya establecida y configurada, se procede con la creación y configuración de la instancia de base de datos relacional gestionada.
3.1. Creación del Grupo de Subredes de Base de Datos (DB Subnet Group)

Un DB Subnet Group es un requisito para el despliegue de instancias RDS dentro de una VPC. Permite a RDS seleccionar subredes en diferentes Zonas de Disponibilidad, lo que es fundamental para la alta disponibilidad y resiliencia de la base de datos.

Acción: Crear un DB Subnet Group que incluya las dos subredes públicas previamente configuradas.

Script:
Bash

#!/bin/bash

# --- Dependencias de IDs de Subredes ---
SUBNET_ID_1=$(cat subnet_id_1.txt) # IDs de las subredes
SUBNET_ID_2=$(cat subnet_id_2.txt)
DB_SUBNET_GROUP_NAME="tutorial-db-subnet-group"
DB_SUBNET_GROUP_DESCRIPTION="Grupo de Subredes para el tutorial de RDS con acceso público"

echo "### 3.1. Creación del Grupo de Subredes de Base de Datos (DB Subnet Group) ###"

if [ -z "$SUBNET_ID_1" ] || [ -z "$SUBNET_ID_2" ]; then
    echo "ERROR: Los IDs de las subredes no se encontraron. Asegúrese de ejecutar el script de creación de subredes previamente."
    exit 1
fi

# Creación del DB Subnet Group
echo "Creando DB Subnet Group '$DB_SUBNET_GROUP_NAME' con subredes $SUBNET_ID_1 y $SUBNET_ID_2..."
aws rds create-db-subnet-group \
    --db-subnet-group-name "$DB_SUBNET_GROUP_NAME" \
    --db-subnet-group-description "$DB_SUBNET_GROUP_DESCRIPTION" \
    --subnet-ids "$SUBNET_ID_1" "$SUBNET_ID_2"

echo "DB Subnet Group '$DB_SUBNET_GROUP_NAME' creado exitosamente."

# Se guarda el nombre del DB Subnet Group para el script de creación de la instancia RDS.
echo "$DB_SUBNET_GROUP_NAME" > db_subnet_group_name.txt

3.2. Lanzamiento de la Instancia Amazon RDS

Este paso finaliza el despliegue de la base de datos MariaDB. Se vinculará con la VPC, el Security Group y el DB Subnet Group previamente configurados, y se habilitará el acceso público según los requisitos del tutorial.

Acción: Crear la instancia RDS tutorial-mariadb-instance con la configuración especificada.

Script:
Bash

#!/bin/bash

# --- Parámetros de configuración de la instancia RDS ---
DB_INSTANCE_IDENTIFIER="tutorial-mariadb-instance" # Nombre de la instancia de base de datos
DB_INSTANCE_CLASS="db.t2.micro" # Clase de instancia (compatible con la capa gratuita)
DB_ENGINE="mariadb"
DB_ENGINE_VERSION="10.1.34" # Versión del motor de MariaDB
MASTER_USERNAME="admin" # Nombre de usuario principal de la base de datos
ALLOCATED_STORAGE_GB=20 # Espacio de almacenamiento asignado en GB
DB_SUBNET_GROUP_NAME=$(cat db_subnet_group_name.txt) # Nombre del DB Subnet Group
SECURITY_GROUP_ID=$(cat security_group_id.txt) # ID del Security Group
AVAILABILITY_ZONE="us-west-2a" # Una de las AZs donde se encuentran sus subredes públicas

echo "### 3.2. Lanzamiento de la Instancia Amazon RDS ###"

if [ -z "$DB_SUBNET_GROUP_NAME" ] || [ -z "$SECURITY_GROUP_ID" ]; then
    echo "ERROR: Faltan los datos necesarios (DB Subnet Group o Security Group ID). Asegúrese de que los scripts anteriores se ejecutaron correctamente."
    exit 1
fi

# Generación de una contraseña segura para la base de datos.
DB_PASSWORD=$(openssl rand -base64 12 | tr -dc A-Za-z0-9) # Genera una contraseña alfanumérica
echo -e "\n### IMPORTANTE: Por favor, guarde esta contraseña de forma segura: $DB_PASSWORD ###\n"

# Creación de la instancia de base de datos MariaDB
echo "Iniciando la creación de la instancia RDS '$DB_INSTANCE_IDENTIFIER'..."
aws rds create-db-instance \
    --db-instance-identifier "$DB_INSTANCE_IDENTIFIER" \
    --db-instance-class "$DB_INSTANCE_CLASS" \
    --engine "$DB_ENGINE" \
    --engine-version "$DB_ENGINE_VERSION" \
    --master-username "$MASTER_USERNAME" \
    --master-user-password "$DB_PASSWORD" \
    --allocated-storage "$ALLOCATED_STORAGE_GB" \
    --db-subnet-group-name "$DB_SUBNET_GROUP_NAME" \
    --vpc-security-group-ids "$SECURITY_GROUP_ID" \
    --publicly-accessible \
    --availability-zone "$AVAILABILITY_ZONE" \
    --no-multi-az \
    --license-model general-public-license \
    --tags Key=Name,Value="$DB_INSTANCE_IDENTIFIER"

echo "La instancia RDS '$DB_INSTANCE_IDENTIFIER' está en proceso de creación. Este proceso puede tardar varios minutos."

# Esperar a que la instancia alcance el estado 'available'.
echo "Esperando que la instancia RDS esté 'available' (disponible)..."
aws rds wait db-instance-available \
    --db-instance-identifier "$DB_INSTANCE_IDENTIFIER"
echo "La instancia RDS '$DB_INSTANCE_IDENTIFIER' ahora está disponible."

# Obtención del Endpoint de la instancia RDS (la dirección para la conexión).
RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier "$DB_INSTANCE_IDENTIFIER" \
    --query 'DBInstances[0].Endpoint.Address' --output text)

if [ -z "$RDS_ENDPOINT" ]; then
    echo "ERROR: No se pudo obtener el Endpoint de la instancia RDS. Verifique el estado de la instancia."
    exit 1
fi

echo -e "\n### Detalles de Conexión de la Instancia RDS ###"
echo "Endpoint de la base de datos: $RDS_ENDPOINT"
echo "Usuario Maestro: $MASTER_USERNAME"
echo "Contraseña Maestro: $DB_PASSWORD" # Conserve esta contraseña de forma segura.

# Se guardan el Endpoint y la contraseña para el script de verificación de conectividad.
echo "$RDS_ENDPOINT" > rds_endpoint.txt
echo "$DB_PASSWORD" > db_password.txt

4. Verificación de Conectividad

Una vez que la instancia RDS se encuentre en estado available, es fundamental verificar la conectividad desde una máquina local para asegurar que la configuración es correcta.

Acción: Intentar establecer una conexión a la instancia RDS utilizando un cliente MariaDB/MySQL.

Script:
Bash

#!/bin/bash

# --- Dependencias de conexión ---
RDS_ENDPOINT=$(cat rds_endpoint.txt) # Endpoint de la base de datos
MASTER_USERNAME="admin" # Nombre de usuario maestro
DB_PASSWORD=$(cat db_password.txt) # Contraseña del usuario maestro

echo "### 4. Verificación de Conectividad ###"

if [ -z "$RDS_ENDPOINT" ] || [ -z "$DB_PASSWORD" ]; then
    echo "ERROR: No se encontraron el Endpoint o la contraseña. Asegúrese de que la instancia RDS se creó y está disponible."
    exit 1
fi

echo "Intentando conectar a la instancia MariaDB..."
echo "Comando de conexión a ejecutar: mariadb -h $RDS_ENDPOINT -u $MASTER_USERNAME -p"
echo "(La contraseña se pasará automáticamente desde el script)"

# Intento de conexión y ejecución de un comando simple para verificar.
# Asegúrese de tener el cliente 'mariadb' o 'mysql' instalado en su sistema local.
# Ej. en sistemas basados en Debian/Ubuntu: sudo apt install mariadb-client
mariadb -h "$RDS_ENDPOINT" -u "$MASTER_USERNAME" -p"$DB_PASSWORD" -e "SHOW DATABASES;"

if [ $? -eq 0 ]; then
    echo -e "\n¡Conexión a MariaDB establecida con éxito! La base de datos es accesible."
else
    echo -e "\nERROR: Fallo al conectar a MariaDB. Revise los siguientes puntos:"
    echo "  - Reglas del Security Group (puerto 3306 permitido desde su IP o 0.0.0.0/0)."
    echo "  - Configuración de la tabla de enrutamiento y el Internet Gateway para las subredes públicas."
    echo "  - Credenciales (usuario y contraseña) de la base de datos."
fi

5. Limpieza de Recursos (¡Crucial para evitar costos!)

Para prevenir cargos inesperados en su cuenta de AWS, es imperativo eliminar todos los recursos creados una vez que haya completado y verificado el tutorial. Este script revertirá todas las acciones de despliegue.

Acción: Eliminar la instancia RDS, el DB Subnet Group, las subredes, el Internet Gateway y la VPC en el orden correcto.

Script de Limpieza (¡Ejecute con extrema precaución!):
Bash

#!/bin/bash

echo "### 5. Limpieza de Recursos (¡Crucial para evitar costos!) ###"
echo "ADVERTENCIA: Este script procederá a eliminar todos los recursos de AWS creados durante este tutorial."
read -p "¿Está seguro de que desea continuar con la eliminación de recursos? (s/N): " -n 1 -r
echo # Salto de línea
if [[ ! $REPLY =~ ^[Ss]$ ]]
then
    echo "Operación de limpieza cancelada."
    exit 1
fi

echo -e "\nIniciando la eliminación de recursos. Este proceso puede tomar algún tiempo..."

# Recuperar IDs de los archivos temporales generados
DB_INSTANCE_IDENTIFIER="tutorial-mariadb-instance"
DB_SUBNET_GROUP_NAME=$(cat db_subnet_group_name.txt 2>/dev/null)
SECURITY_GROUP_ID=$(cat security_group_id.txt 2>/dev/null)
SUBNET_ID_1=$(cat subnet_id_1.txt 2>/dev/null)
SUBNET_ID_2=$(cat subnet_id_2.txt 2>/dev/null)
MAIN_RT_ID=$(cat main_rt_id.txt 2>/dev/null)
IGW_ID=$(cat igw_id.txt 2>/dev/null)
VPC_ID=$(cat vpc_id.txt 2>/dev/null)


# 1. Eliminar la instancia RDS
if [ -n "$DB_INSTANCE_IDENTIFIER" ]; then
    echo "Eliminando instancia RDS '$DB_INSTANCE_IDENTIFIER'..."
    aws rds delete-db-instance \
        --db-instance-identifier "$DB_INSTANCE_IDENTIFIER" \
        --skip-final-snapshot \
        --delete-automated-backups || true # El '|| true' evita que el script falle si el recurso ya no existe
    echo "Esperando que la instancia RDS termine de eliminarse..."
    aws rds wait db-instance-deleted \
        --db-instance-identifier "$DB_INSTANCE_IDENTIFIER" || true
    echo "Instancia RDS eliminada."
fi

# 2. Eliminar el DB Subnet Group
if [ -n "$DB_SUBNET_GROUP_NAME" ]; then
    echo "Eliminando DB Subnet Group '$DB_SUBNET_GROUP_NAME'..."
    aws rds delete-db-subnet-group \
        --db-subnet-group-name "$DB_SUBNET_GROUP_NAME" || true
    echo "DB Subnet Group eliminado."
fi

# 3. Revocar la regla de entrada del Security Group (solo la regla específica)
if [ -n "$SECURITY_GROUP_ID" ]; then
    echo "Revocando regla de entrada del Security Group $SECURITY_GROUP_ID (TCP/3306 desde 0.0.0.0/0)..."
    aws ec2 revoke-security-group-ingress \
        --group-id "$SECURITY_GROUP_ID" \
        --protocol tcp \
        --port 3306 \
        --cidr 0.0.0.0/0 || true
    echo "Regla de Security Group revocada (si existía)."
fi
# Nota: No se elimina el Security Group por defecto de la VPC, solo la regla añadida.

# 4. Eliminar las subredes
if [ -n "$SUBNET_ID_1" ]; then
    echo "Eliminando Subred 1 ($SUBNET_ID_1)..."
    aws ec2 delete-subnet --subnet-id "$SUBNET_ID_1" || true
    echo "Subred 1 eliminada."
fi

if [ -n "$SUBNET_ID_2" ]; then
    echo "Eliminando Subred 2 ($SUBNET_ID_2)..."
    aws ec2 delete-subnet --subnet-id "$SUBNET_ID_2" || true
    echo "Subred 2 eliminada."
fi

# 5. Eliminar la ruta del Internet Gateway de la tabla de enrutamiento principal
if [ -n "$MAIN_RT_ID" ] && [ -n "$IGW_ID" ]; then
    echo "Eliminando ruta del Internet Gateway de la tabla de enrutamiento $MAIN_RT_ID..."
    aws ec2 delete-route \
        --route-table-id "$MAIN_RT_ID" \
        --destination-cidr-block 0.0.0.0/0 || true
    echo "Ruta eliminada."
fi

# 6. Desvincular y eliminar el Internet Gateway
if [ -n "$IGW_ID" ] && [ -n "$VPC_ID" ]; then
    echo "Desvinculando Internet Gateway $IGW_ID de la VPC $VPC_ID..."
    aws ec2 detach-internet-gateway \
        --internet-gateway-id "$IGW_ID" \
        --vpc-id "$VPC_ID" || true
    echo "Internet Gateway desvinculado."

    echo "Eliminando Internet Gateway $IGW_ID..."
    aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID" || true
    echo "Internet Gateway eliminado."
fi

# 7. Finalmente, eliminar la VPC
if [ -n "$VPC_ID" ]; then
    echo "Eliminando VPC $VPC_ID..."
    # La eliminación de la VPC puede tomar un momento si aún tiene interfaces de red u otros recursos asociados.
    aws ec2 delete-vpc --vpc-id "$VPC_ID" || true
    echo "VPC eliminada."
fi

echo -e "\nProceso de limpieza de recursos completado. Se recomienda verificar su consola AWS para confirmar la eliminación."

# Eliminar archivos temporales con IDs
rm -f vpc_id.txt igw_id.txt main_rt_id.txt subnet_id_1.txt subnet_id_2.txt security_group_id.txt db_subnet_group_name.txt rds_endpoint.txt db_password.txt
echo "Archivos temporales con IDs eliminados."

6. Diagrama de Arquitectura de la Solución

A continuación, se presenta un diagrama que ilustra la arquitectura implementada, facilitando la comprensión de la interconexión de los componentes.
Fragmento de código

graph TD
    subgraph Internet
        A[Cliente de Internet] --> |TCP/3306| B(AWS Internet Gateway)
    end

    subgraph AWS Cloud
        subgraph Region (ej. us-west-2)
            subgraph VPC: tutorial-vpc (10.0.0.0/16)
                direction LR
                subgraph "DB Subnet Group"
                    subnet1["Subred Pública 1 (10.0.0.0/24)<br>AZ: us-west-2a"]
                    subnet2["Subred Pública 2 (10.0.1.0/24)<br>AZ: us-west-2b"]
                end

                RT["Route Table (Principal)<br>Ruta por defecto: 0.0.0.0/0 -> IGW"]
                SG["Security Group<br>(Regla de entrada: TCP/3306 desde 0.0.0.0/0)"]

                subnet1 --- RT
                subnet2 --- RT
                RT --> B

                RDS_DB["Amazon RDS (Instancia MariaDB)<br>tutorial-mariadb-instance"]
                RDS_DB --- db_subnet_group
                RDS_DB --- SG
            end
        end
    end

    B --> SG
    SG --> RDS_DB

Elementos Clave del Diagrama:

    Cliente de Internet: Representa cualquier sistema externo que inicia la conexión.
    AWS Internet Gateway (IGW): El punto de entrada y salida para el tráfico entre la VPC e internet.
    VPC (tutorial-vpc): El entorno de red virtual aislado que contiene todos los recursos desplegados.
    Subredes Públicas (Subred 1, Subred 2): Segmentos de red dentro de la VPC, ubicados en diferentes Zonas de Disponibilidad para asegurar redundancia y alta disponibilidad. Están configuradas para permitir comunicación con el IGW.
    Route Table (Principal): Define las reglas de enrutamiento del tráfico dentro de la VPC, incluyendo una ruta por defecto que dirige el tráfico de internet hacia el IGW.
    Security Group: Actúa como un firewall virtual que controla el tráfico a nivel de instancia. Es crucial que permita el tráfico TCP en el puerto 3306 desde el origen deseado (en este caso, 0.0.0.0/0).
    DB Subnet Group: Una colección de subredes que Amazon RDS utiliza para desplegar la instancia de base de datos, garantizando que se pueda ubicar en múltiples AZs.
    Amazon RDS (Instancia MariaDB): La base de datos relacional gestionada por AWS. Su despliegue aprovecha la configuración de red y seguridad de la VPC para ser accesible públicamente.