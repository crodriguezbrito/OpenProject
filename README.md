# Laboratorio de Integración: OpenLDAP + OpenProject en Docker
POC de conexión entre OpenLdap y OpenProject para permitir loguear usuarios usando LDAP.
Este repositorio contiene la configuración necesaria para desplegar un entorno local de pruebas con autenticación centralizada. El laboratorio incluye un servidor OpenLDAP, la interfaz de administración web phpLDAPadmin y la plataforma de gestión de proyectos OpenProject operando conjuntamente dentro de una red aislada de Docker.

## Requisitos Previos

Antes de comenzar, asegúrate de tener instalado en tu sistema operativo local:
* Docker Engine v20.10+
* Docker Compose v2.0+
* Un navegador web actualizado.

## Estructura del Proyecto

openproject-ldap/
 - docker-compose.yml   # Orquestación de contenedores y volúmenes
 -  README.md            # Documentación del entorno (este archivo)

## Despliegue Rápido

1. Crea el directorio del proyecto y ubícate dentro de él:
   mkdir openproject-ldap && cd openproject-ldap

2. Guarda tu archivo docker-compose.yml en esta carpeta y levanta la infraestructura en segundo plano (modo detached):
   docker compose up -d

3. Verifica que los tres contenedores estén corriendo de manera correcta:
   docker compose ps

NOTA: El contenedor de OpenProject puede demorar entre 2 y 4 minutos en su primer inicio debido a la creación y migración automática de su base de datos interna. Puedes monitorizar el estado del arranque ejecutando el comando: docker logs -f openproject_local

## Matriz de URLs y Credenciales de Acceso

- OpenProject:
  * URL: http://localhost:8000
  * Usuario: admin
  * Contraseña: admin (cambiar en primer inicio a: admin123456789)

- phpLDAPadmin:
  * URL: http://localhost:8080
  * Login DN: cn=admin,dc=ejemplo,dc=local
  * Contraseña: adminpassword

-------------------------------------------------------------------------------

## Pasos de Verificación de Extremo a Extremo

Sigue esta secuencia exacta para validar que todo el ecosistema de autenticación federada funcione correctamente.

### Paso 1: Configurar la estructura base en LDAP
1. Accede a http://localhost:8080 e inicia sesión en phpLDAPadmin utilizando el Login DN y clave indicados en la tabla de arriba.
2. En el árbol izquierdo, haz clic en la raíz dc=ejemplo,dc=local -> "Create a child entry" -> "Generic: Organizational Unit". Escribe el nombre "usuarios", pulsa en Create Object y confirma con Commit.
3. Haz clic de nuevo en la raíz dc=ejemplo,dc=local -> "Create a child entry" -> "Generic: Posix Group". Escribe el nombre "usuarios-grupo". El sistema le dará de forma automática el número 500. Confirma con Create Object y Commit.

### Paso 2: Crear el usuario de prueba (Carlos)
1. Selecciona la carpeta ou=usuarios en el árbol de la izquierda.
2. Haz clic en "Create a child entry" -> "Generic: User Account".
3. Rellena los campos obligatorios del perfil con los siguientes datos:
   * First name / Last name: Carlos Laboratorio
   * User ID (uid): Carlos (Atención: Respeta las mayúsculas/minúsculas de cómo lo escribas aquí)
   * Password: carlos12345
   * GID Number: Escribe 500 (asociado a tu nuevo usuarios-grupo)
   * UID Number: Escribe un ID de usuario único libre (ej: 1002)
   * Home directory: /home/carlos
4. Guarda los cambios (Create Object y Commit).
5. Añadir Email (Obligatorio): Selecciona al usuario uid=Carlos a la izquierda, haz clic arriba en "Add attribute", selecciona "Email", escribe carlos@ejemplo.local y dale a Save Changes.

### Paso 3: Enlazar OpenProject con el Servidor LDAP
1. Entra a http://localhost:8000 en tu navegador con la cuenta de administrador global actualizada (admin / admin123456789).
2. Ve al menú lateral en Administración -> Autenticación -> Conexiones LDAP y pulsa el botón verde + LDAP connection.
3. Introduce la configuración técnica del servidor interno:
   * Name: LDAP Local
   * Host: openldap (gracias a Docker se resuelven mediante el nombre del servicio)
   * Port: 389
   * Account: cn=admin,dc=ejemplo,dc=local
   * Password: adminpassword
   * Base DN: dc=ejemplo,dc=local
   * Login attribute: uid
4. En la sección Mapeo de atributos (un poco más abajo):
   * Firstname: givenName
   * Lastname: sn
   * Email: mail
5. Crucial: Marca la casilla "On-the-fly user creation" (Creación de usuarios al vuelo).
6. Guarda con el botón Save y presiona la opción "Test connection" para verificar el indicador verde de éxito.

### Paso 4: Test final de Login
1. Cierra tu sesión de administrador en OpenProject o abre una pestaña del navegador en modo de incógnito.
2. Ve a la URL de acceso: http://localhost:8000.
3. Introduce las credenciales creadas en tu directorio LDAP:
   * Usuario: Carlos
   * Contraseña: carlos12345

Si todo ha ido bien, OpenProject se conectará internamente con el contenedor de LDAP, verificará el password, tomará de forma dinámica el email carlos@ejemplo.local y le aprovisionará una cuenta limpia en el sistema con acceso automático.

-------------------------------------------------------------------------------

## Comandos Útiles de Mantenimiento

Para detener el laboratorio de forma temporal reteniendo la información de la base de datos:
docker compose stop

Para encender de nuevo el laboratorio tras una parada temporal:
docker compose start

Para destruir el laboratorio por completo, liberando y eliminando todos los datos almacenados en los volúmenes persistentes:
docker compose down -v

## Nota
Se trata de un POC para probar que el sistema funciona correctamente. Se recomienda no utilizar en entornos de producción.
