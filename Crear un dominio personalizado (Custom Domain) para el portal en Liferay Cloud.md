# Crear un dominio personalizado (Custom Domain) para el portal en Liferay Cloud

Guía paso a paso para conectar un dominio propio al portal (Liferay DXP) en
Liferay Cloud (Liferay PaaS), con ejemplos de configuración.

---

## Concepto general

Con Liferay Cloud puedes conectar dominios personalizados a los servicios de un
entorno usando tu proveedor de DNS. El flujo tiene **dos partes** y el orden
importa:

1. **DNS:** registrar el dominio apuntando a la IP del Load Balancer del
   entorno.
2. **Servicio:** agregar el dominio al servicio correspondiente (vía `LCP.json`
   o consola).

> **Importante:** para el portal, el dominio se agrega al servicio
> **`webserver`** (el Nginx que está delante del DXP y enruta el tráfico
> público), **no** al servicio `liferay` directamente.

> **Aviso:** los cambios o adiciones de dominios personalizados pueden tardar
> una o más horas en propagarse.

---

## Requisitos previos

- Tener acceso al **registrador/proveedor de DNS** del dominio (por ejemplo,
  Cloudflare, GoDaddy, Route 53, etc.).
- Conocer la **IP del Ingress Load Balancer** del entorno donde quieres el
  dominio (ver el paso 1).
- Acceso al repositorio del proyecto (para editar el `LCP.json` del
  `webserver`) o a la consola de Liferay Cloud.

---

## Paso 1: Obtener la IP del Load Balancer del entorno

Cada entorno del proyecto tiene su **propia IP** de Ingress Load Balancer, que
se usa para conectar los dominios personalizados a los servicios del entorno.

Puedes encontrar esta IP en:

- La página **Network** del entorno, o
- La página dedicada de cada servicio, bajo la sección **Custom Domains**.

> Cada entorno (`dev`, `uat`, `prd`, etc.) tiene una IP distinta. Usa la del
> entorno correcto.

---

## Paso 2: Registrar el dominio en el DNS (registro tipo A)

En tu proveedor de DNS, crea un registro de tipo **`A`** que apunte tu dominio
a la IP del Load Balancer del entorno (obtenida en el paso 1). Haz esto **antes**
de agregar el dominio al servicio.

Ejemplo conceptual del registro:

```
Tipo:   A
Nombre: www.acme.com   (o el dominio que quieras usar)
Valor:  <IP-del-Load-Balancer-del-entorno>
```

**Sobre la propagación del DNS:**

- Puede tardar hasta **24–48 horas** en propagarse por completo, aunque en
  muchos casos toma solo unos minutos.
- Durante la propagación, un dispositivo puede alcanzar el dominio y otro no,
  según el servidor DNS que consulte cada uno.
- Cuando esté listo, el dominio será accesible desde cualquier dispositivo y
  devolverá el error estándar **`default backend - 404`** del load balancer de
  Liferay Cloud. **Ese 404 es la señal de que el DNS ya apunta bien**, aún antes
  de asociar el dominio al servicio.

---

## Paso 3: Agregar el dominio al servicio `webserver` (vía LCP.json)

Este es el método recomendado. Agrega la propiedad `customDomains` dentro del
atributo `environments`, para el **entorno específico**, en el `LCP.json` del
servicio `webserver`:

```json
{
  "id": "webserver",
  "environments": {
    "prd": {
      "loadBalancer": {
        "certs": [
          {
            "customDomains": ["acme.com", "www.acme.com"]
          }
        ]
      }
    }
  }
}
```

> **Regla clave — un dominio por entorno.** Debes definir un entorno específico
> para cada dominio personalizado. **No puedes usar el mismo dominio en varios
> entornos** (la única excepción son los entornos de Disaster Recovery en
> regiones distintas). Esto es necesario para que Liferay Cloud genere los
> certificados correctamente y enrute a los usuarios al dominio correcto.

Una vez agregado el dominio y desplegados los cambios, Liferay Cloud se encarga
del enrutamiento.

### Certificado SSL automático (Let's Encrypt)

Para los dominios personalizados, Liferay Cloud solicita automáticamente un
certificado de **Let's Encrypt** que se renueva solo. No necesitas hacer nada
adicional si usas el certificado automático.

Si necesitas usar tu **propio certificado SSL**, debes agregarlo también en el
`LCP.json` del servicio `webserver` (dentro de las entradas de `certs`, con las
claves `key` y `crt` referenciando secrets). Consulta la página *Load Balancer*
de la documentación oficial para el formato vigente.

---

## Paso 3 (alternativo): Agregar el dominio desde la consola

> **Aviso:** este método está **deprecado**. Se recomienda usar el `LCP.json`.

1. Navega al entorno deseado.
2. Selecciona el servicio (`webserver` para el portal).
3. Abre la pestaña **Custom Domains**.
4. Ingresa los dominios registrados en el campo **Domain Names**.
5. Haz clic en **Update Custom Domains** para finalizar.

> Al agregar dominios por consola, se usa un certificado de Let's Encrypt por
> defecto. Si tienes certificados SSL propios, debes agregarlos por el
> `LCP.json` del `webserver` en su lugar.

---

## Paso 4: Verificar el estado del dominio

Puedes verificar el estado de dos formas:

- **En el navegador:** entra al dominio. Cuando el endpoint está listo, deja de
  devolver el error `default backend - 404` y deja de mostrar advertencias de
  seguridad.
- **En la consola:** revisa el estado en la página **Network** del entorno,
  donde se listan todos los endpoints y dominios personalizados.

**Ten paciencia con los procesos de backend.** Tras configurar el dominio,
Liferay Cloud ejecuta varios pasos que toman algún tiempo:

1. Agregar una ruta al load balancer.
2. Solicitar un certificado SSL a Let's Encrypt.
3. Recibir y resolver el *challenge* de Let's Encrypt.
4. Actualizar el load balancer con el certificado una vez superado el
   *challenge*.

> Si intentas acceder al dominio durante el proceso de *challenge*, puedes
> ignorar con seguridad las advertencias de seguridad del navegador. Al
> completarse, el load balancer queda actualizado con el certificado y el
> servicio es accesible y seguro.

---

## Límites a tener en cuenta

Por entorno, Liferay Cloud aplica los siguientes máximos:

| Límite                                   | Valor |
| ---------------------------------------- | ----- |
| Certificados por entorno                 | 14    |
| Dominios personalizados por entorno      | 250   |
| Dominios Let's Encrypt por certificado   | 98    |
| Dominios totales                         | 498   |

> El número de dominios personalizados también puede estar limitado por las
> cuotas definidas durante el aprovisionamiento.

---

## Ejemplo completo de referencia

`LCP.json` del servicio `webserver` con dominios personalizados por entorno
(dominios distintos para `uat` y `prd`, respetando la regla de un dominio por
entorno):

```json
{
  "id": "webserver",
  "environments": {
    "uat": {
      "loadBalancer": {
        "certs": [
          {
            "customDomains": ["uat.acme.com"]
          }
        ]
      }
    },
    "prd": {
      "loadBalancer": {
        "cdn": true,
        "certs": [
          {
            "customDomains": ["acme.com", "www.acme.com"]
          }
        ]
      }
    }
  }
}
```

En este ejemplo:

- El portal responde en `uat.acme.com` en el entorno `uat`.
- El portal responde en `acme.com` y `www.acme.com` en `prd`, con la CDN
  activada.
- Cada dominio está asociado a un único entorno.
- Los certificados se emiten automáticamente vía Let's Encrypt.

---

## Resumen del flujo

1. Obtén la IP del Load Balancer del entorno (página **Network**).
2. Crea un registro **A** en tu DNS apuntando el dominio a esa IP.
3. Espera la propagación (hasta que el dominio devuelva `default backend - 404`).
4. Agrega el dominio al `LCP.json` del servicio **`webserver`**, dentro del
   entorno específico.
5. Despliega los cambios.
6. Verifica el estado en el navegador o en la página **Network**.

---

## Referencias oficiales

- Custom Domains:
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/custom-domains`
- Load Balancer (incluye certificados SSL propios):
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/load-balancer`
- Configuration via LCP.json:
  `https://learn.liferay.com/w/dxp/cloud/reference/configuration-via-lcp-json`
