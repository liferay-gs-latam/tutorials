# Crear un servicio personalizado (Custom Service) en Liferay Cloud

Guía paso a paso para crear y desplegar un servicio personalizado en Liferay
Cloud (Liferay PaaS), con ejemplos de configuración y de exposición a
internet.

---

## ¿Qué es un servicio personalizado?

En Liferay Cloud no estás limitado al conjunto de servicios estándar
(`liferay`, `database`, `search`, `webserver`, etc.). Puedes crear y desplegar
un **servicio personalizado** para ejecutar cualquier proceso propio dentro de
la infraestructura de Liferay Cloud: un worker, una API auxiliar, un servicio
de integración, un cron, etc.

Cada servicio se basa en una **imagen Docker** y se configura mediante un
archivo `LCP.json`.

---

## Requisitos previos

Antes de empezar, ten en cuenta lo siguiente:

- **Debe habilitarse en tu cuenta.** El uso de servicios personalizados no está
  activo por defecto. Contacta a tu representante de ventas para habilitar la
  funcionalidad y asegurar que tus entornos estén aprovisionados
  correctamente.
- **Necesitas recursos de hardware suficientes.** Un servicio personalizado
  consume CPU y memoria del *pool* compartido del entorno. Si no hay recursos
  provisionados para él, puede interferir con los recursos asignados a los
  demás servicios. La asignación adicional se define durante el
  aprovisionamiento.
- **Docker.** Los servicios se basan en imágenes Docker. Si quieres ejecutarlos
  localmente, instala Docker en tu máquina.

> Consulta también las *limitaciones de los servicios personalizados* en la
> documentación oficial de Liferay Cloud antes de avanzar.

---

## Pasos para crear el servicio

### 1. Prepara la imagen Docker

Crea o localiza tu servicio como una imagen Docker. Tienes dos opciones:

- Usar un **Dockerfile** que agregas directamente al workspace del proyecto, o
- Usar una **imagen de un repositorio público** (por ejemplo, Docker Hub).

### 2. Crea un directorio para el servicio

Agrega un directorio nuevo junto a los demás servicios, con un archivo
`LCP.json` dentro:

```
├── backup
├── ci
├── database
├── liferay
├── search
├── webserver
└── miservicio
    └── LCP.json
```

### 3. Configura el `LCP.json`

Un `LCP.json` mínimo se ve así:

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1
}
```

- **`id`**: identificador del servicio.
- **`image`**: nombre de la imagen en un repositorio público. Ver la nota sobre
  Dockerfile más abajo.
- **`memory`** / **`cpu`**: recursos asignados (ver la sección siguiente).

### 4. Asigna la imagen Docker

El método depende del origen de la imagen:

- **Imagen de un repositorio público:** define la propiedad `image` con el
  nombre de la imagen:

  ```json
  "image": "midockerhub/miservicio:1.0.0"
  ```

- **Dockerfile local:** coloca el `Dockerfile` dentro del directorio del
  servicio. Al construir el servicio, la imagen del `Dockerfile` se toma
  automáticamente.

  > **Importante:** si hay un `Dockerfile` en el directorio, se usa como imagen
  > del servicio de forma automática y **cualquier propiedad `image` en el
  > `LCP.json` se ignora**. Es un error común cambiar la etiqueta en `image`,
  > reconstruir y no ver cambios porque en realidad manda el `Dockerfile`.

### 5. Commitea los cambios

```bash
git add miservicio/
git commit -m "Agregar servicio personalizado"
```

### 6. Haz push y dispara un build

Sube tu rama e inicia un nuevo build en Liferay Cloud para desplegar.

Tras el build, en la pantalla **Builds** de la consola, el nuevo servicio
aparece en la columna **Services** junto a los demás. Al desplegar el build en
un entorno (**Deploy Build to** en el menú *Actions*), el servicio aparece
también en la página **Services** de ese entorno.

---

## Recursos: memoria y CPU

Declarar `memory` y `cpu` **no es obligatorio**: si los omites, el servicio
arranca con los valores por defecto.

| Propiedad | Valor por defecto | Descripción            |
| --------- | ----------------- | ---------------------- |
| `cpu`     | `1`               | Número de CPUs         |
| `memory`  | `512`             | Memoria en MB          |
| `scale`   | `1`               | Número de instancias   |

Aunque son opcionales, **conviene declararlos de forma consciente** en un
servicio personalizado:

- **512 MB suele ser poco** para muchas aplicaciones (por ejemplo, cualquier
  app basada en JVM), lo que puede provocar reinicios o errores de tipo
  *out-of-memory* difíciles de diagnosticar.
- **Sobredimensionar** también es un problema: le quitas recursos a los demás
  servicios del mismo entorno.

### Override por entorno

Puedes definir valores base en la raíz y ajustarlos por entorno dentro de
`environments`. El valor del entorno tiene prioridad sobre el de la raíz:

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1,
  "environments": {
    "dev": {
      "memory": 512
    },
    "prd": {
      "memory": 2048,
      "cpu": 2
    }
  }
}
```

---

## Controlar en qué entornos se despliega: `deploy`

La propiedad `deploy` es un booleano que controla si el servicio se despliega
en un entorno determinado.

- **Valor por defecto: `true`.** Sin configurar nada, el servicio se despliega
  en **todos** los entornos.
- **Solo se usa dentro de `environments`**, nunca en la raíz del `LCP.json`.

### Cómo restringir a un solo entorno (importante)

Como el valor por defecto ya es `true`, marcar un entorno con `"deploy": true`
**no restringe nada**: los demás entornos siguen heredando el `true` por
defecto y también se despliegan.

Para que el servicio corra **solo en producción**, hay que hacer lo contrario:
poner `"deploy": false` en los entornos que quieres excluir.

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1,
  "environments": {
    "dev": {
      "deploy": false
    },
    "uat": {
      "deploy": false
    },
    "prd": {
      "memory": 2048,
      "cpu": 2
    }
  }
}
```

En este ejemplo el servicio solo se despliega en `prd` (los demás entornos se
desactivaron explícitamente) y, de paso, `prd` recibe más recursos.

**Casos de uso típicos de `deploy: false`:**

- Un servicio pesado que solo quieres en `prd` para no consumir recursos en
  `dev`/`uat`.
- Un servicio auxiliar que solo tiene sentido en producción.
- Servicios que deben correr solo en un entorno concreto (por ejemplo, tareas
  que corresponden a un único entorno).

---

## Exponer el servicio a internet

El acceso web se configura con el bloque **`loadBalancer`** dentro del
`LCP.json` del servicio. El Ingress Load Balancer de Liferay Cloud da acceso a
internet a los servicios del entorno mediante conexiones HTTP(S) con TLS.

### Endpoint automático `.lfr.cloud`

Con solo declarar el bloque `loadBalancer`, la plataforma crea automáticamente
un endpoint público siguiendo este patrón de nombre:

```
<NOMBRE-SERVICIO>-<NOMBRE-PROYECTO>-<NOMBRE-ENTORNO>.lfr.cloud
```

Estos dominios `.lfr.cloud` están cubiertos por un certificado *wildcard* de la
propia infraestructura (no necesitas configurar SSL).

### Configuración mínima

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1,
  "loadBalancer": {
    "targetPort": 3000
  }
}
```

> **`targetPort` es la pieza clave.** Debe apuntar al puerto donde escucha tu
> aplicación **dentro del contenedor** (no al puerto público). Si tu servicio
> arranca en el puerto 3000, aquí va `3000`.

### Con dominio propio y CDN

Para usar tu propio dominio en lugar del `.lfr.cloud`, el flujo tiene dos
partes y **el orden importa**:

1. **Lado DNS:** registra tu dominio como un registro tipo **A** apuntando a la
   IP del Ingress Load Balancer de ese entorno. Cada entorno tiene su propia IP
   fija, visible en la página **Network** del entorno y en la página del
   servicio, bajo **Custom Domains**.
2. **Lado servicio:** una vez propagado el dominio, agrégalo al servicio (vía
   consola o vía `LCP.json`). Para los dominios personalizados, Liferay Cloud
   emite automáticamente un certificado de **Let's Encrypt** que se renueva
   solo.

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1,
  "loadBalancer": {
    "cdn": true,
    "targetPort": 3000,
    "customDomains": ["api.acme.com"]
  }
}
```

- **`cdn: true`**: activa la CDN integrada de Liferay Cloud delante del
  endpoint (cachea contenido en POPs cercanos al usuario). Útil para contenido
  cacheable; prescindible para una API dinámica.
- **`customDomains`**: lista de dominios propios a mapear al servicio.

> **Propagación:** los cambios en dominios personalizados pueden tardar una o
> más horas en propagarse (en algunos casos hasta 24–48 h).

### Certificado SSL propio (opcional)

Si necesitas usar tu propio certificado (en lugar del de Let's Encrypt), las
versiones recientes de la documentación usan un array `certs` en el `LCP.json`
del servicio `webserver`, con `customDomains`, `key` y `crt` en cada entrada.
Si solo usas el certificado automático de Let's Encrypt, no necesitas esta
configuración. Verifica siempre el formato vigente en la página *Load Balancer*
de la documentación oficial.

---

## Nota importante: autenticación básica en entornos no productivos

Por defecto, todos los entornos **no productivos** (`dev`, `uat`, etc.) en
Liferay PaaS exigen **autenticación básica** (vía navegador) para acceder. Esto
protege los entornos de desarrollo.

En la práctica: al exponer tu servicio en `dev` o `uat` y abrir la URL, verás
un prompt de *basic auth* delante. Es el comportamiento esperado del web
server, **no un error de tu servicio**. En `prd` esto no ocurre.

---

## Ejemplo completo de referencia

`LCP.json` que combina imagen pública, recursos con override por entorno,
control de despliegue por entorno y exposición a internet:

```json
{
  "id": "miservicio",
  "image": "midockerhub/miservicio:1.0.0",
  "memory": 1024,
  "cpu": 1,
  "loadBalancer": {
    "cdn": true,
    "targetPort": 3000,
    "customDomains": ["api.acme.com"]
  },
  "environments": {
    "dev": {
      "deploy": false
    },
    "uat": {
      "memory": 512
    },
    "prd": {
      "memory": 2048,
      "cpu": 2
    }
  }
}
```

En este ejemplo:

- El servicio usa una imagen pública de Docker Hub.
- Recibe 1 GB de memoria y 1 CPU por defecto, con ajustes por entorno.
- No se despliega en `dev`.
- Se expone a internet en `api.acme.com` con CDN activada, apuntando al puerto
  interno 3000.

---

## Referencias oficiales

- Using a Custom Service:
  `https://learn.liferay.com/w/dxp/cloud/platform-services/using-a-custom-service`
- Configuration via LCP.json:
  `https://learn.liferay.com/w/dxp/cloud/reference/configuration-via-lcp-json`
- Load Balancer:
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/load-balancer`
- Custom Domains:
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/custom-domains`
