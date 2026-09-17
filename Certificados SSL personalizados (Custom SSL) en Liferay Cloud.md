# Certificados SSL personalizados (Custom SSL) en Liferay Cloud

Tutorial paso a paso para agregar tu propio certificado SSL a los dominios
personalizados en Liferay Cloud (Liferay PaaS), con ejemplos de configuración y
codificación.

---

## ¿Cuándo necesitas un certificado propio?

Por defecto, Liferay Cloud ya gestiona certificados por ti:

- Los dominios de infraestructura (`.lfr.cloud`) están cubiertos por un
  certificado **wildcard** de la plataforma.
- Para los dominios personalizados agregados por consola o `LCP.json`, Liferay
  Cloud solicita automáticamente un certificado de **Let's Encrypt** que se
  renueva solo.

Usa un **certificado SSL personalizado** solo cuando necesites algo que
Let's Encrypt no cubre: por ejemplo, un certificado emitido por una CA
corporativa específica, un certificado wildcard propio, o requisitos de
cumplimiento que exijan un emisor concreto.

> **Precedencia:** si existen certificados en ambos lugares (Let's Encrypt vía
> consola y certificados propios en el `LCP.json`), **los certificados
> personalizados definidos en el `LCP.json` tienen prioridad**.

---

## Conceptos clave antes de empezar

- Los certificados personalizados se configuran en el `LCP.json` del servicio
  **`webserver`**, dentro del array `certs` del objeto `loadBalancer`.
- Los valores del certificado (`key` y `crt`) **no se escriben en texto plano**
  en el `LCP.json`: se almacenan como **secrets** del entorno y se referencian
  desde el archivo.
- Liferay Cloud solo acepta claves y certificados en **formato PEM con
  codificación Base64**, incluyendo las líneas de encapsulación
  (`-----BEGIN...-----` / `-----END...-----`).
- Cada certificado se agrupa junto con los dominios personalizados a los que
  aplica.

> **Requisito previo:** los dominios ya deben estar configurados como custom
> domains (registro DNS tipo A apuntando a la IP del Load Balancer del
> entorno). Consulta la guía de *Custom Domains* si aún no lo hiciste.

---

## Paso 1: Generar / preparar el certificado y la clave

Al generar la clave, debes usar los algoritmos de cifrado **RSA-2048** o
**ECDSA P-256**, y **evitar claves protegidas por passphrase**.

Los archivos de certificado y clave contienen texto en este formato (con
`CERTIFICATE` o `KEY` en las etiquetas de inicio/fin, respectivamente):

```
-----BEGIN CERTIFICATE-----
base64encodedcertificate
-----END CERTIFICATE-----
```

> **Nota:** una vez creados los certificados personalizados, tú eres
> responsable de gestionarlos (actualizarlos al agregar nuevos dominios,
> renovarlos cuando expiren). A diferencia de Let's Encrypt, **no se renuevan
> automáticamente**.

---

## Paso 2: Codificar la clave y el certificado en Base64

Liferay Cloud requiere los valores en Base64 (con las etiquetas de
encapsulación incluidas). Si aún no están codificados, sigue estos pasos:

1. Crea un archivo para la clave y otro para el certificado:

   ```bash
   touch originalkeyfile.key
   touch originalcertfile.crt
   ```

2. Copia **todo** el contenido de la clave —**incluyendo las etiquetas begin/end**—
   en `originalkeyfile.key` y guárdalo.

3. Copia **todo** el contenido del certificado —**incluyendo las etiquetas
   begin/end**— en `originalcertfile.crt` y guárdalo.

4. Convierte ambos archivos a Base64 (o usa tu método de codificación
   preferido):

   ```bash
   openssl base64 -A -in originalkeyfile.key -out base64keyfile.key
   openssl base64 -A -in originalcertfile.crt -out base64certfile.crt
   ```

Ahora los archivos `base64keyfile.key` y `base64certfile.crt` contienen los
valores codificados listos para usar.

> **Tip:** puedes incluir varios certificados en el campo `crt` concatenando los
> certificados en una sola cadena y codificando el resultado en Base64 (útil
> para cadenas de certificados / intermediate certs).

---

## Paso 3: Crear los secrets en el entorno

Almacena los valores codificados como **secret variables** del entorno elegido
(uno para la clave y otro para el certificado). Copia el contenido de
`base64keyfile.key` y `base64certfile.crt` en sus respectivos secrets.

> Los secrets se crean por entorno. Consulta *Managing Secure Environment
> Variables with Secrets* en la documentación oficial para el procedimiento de
> creación de secrets.

Por convención, este tutorial usa los nombres de secret:

- `ssl-key-secret` → contenido de la clave codificada
- `ssl-crt-secret` → contenido del certificado codificado

---

## Paso 4: Referenciar los secrets en el `LCP.json` del `webserver`

En `webserver/LCP.json`, usa la propiedad `certs` dentro del objeto
`loadBalancer`. Cada entrada del array agrupa `key`, `crt` y los
`customDomains` a los que aplica. Los valores `key` y `crt` referencian los
secrets usando el prefijo **`@`**:

```json
{
  "id": "webserver",
  "loadBalancer": {
    "certs": [
      {
        "customDomains": ["acme.liferay.cloud"],
        "key": "@ssl-key-secret",
        "crt": "@ssl-crt-secret"
      }
    ]
  }
}
```

### Múltiples certificados

Para mapear varios certificados a distintos dominios, agrega más objetos al
array `certs`, cada uno con su propio par de secrets:

```json
{
  "id": "webserver",
  "loadBalancer": {
    "certs": [
      {
        "customDomains": ["acme.liferay.cloud"],
        "key": "@ssl-key-secret",
        "crt": "@ssl-crt-secret"
      },
      {
        "customDomains": ["acme2.liferay.cloud", "acme3.liferay.cloud"],
        "key": "@ssl-key-secret-2",
        "crt": "@ssl-crt-secret-2"
      }
    ]
  }
}
```

> **Consola vs. LCP.json:** mapear múltiples certificados a tus dominios
> requiere usar la propiedad `certs` en el `LCP.json` del `webserver`. Agregar
> dominios por la consola, en cambio, mapea **todos** los dominios a un único
> certificado.

---

## Paso 5: Desplegar y reiniciar el web server

> **Importante:** al agregar o actualizar un certificado SSL para dominios
> personalizados, debes **desplegar un build al servicio `webserver` y
> reiniciarlo** para que los cambios surtan efecto.

---

## Paso 6: Verificar el estado del certificado

La página **Network** muestra tanto los certificados que Liferay provee como
los certificados personalizados que agregaste. En la pestaña **SSL
Certificates**:

- Haz clic en cada entrada de la tabla para ver más información del certificado.
- Al renovar certificados, vuelve a esta pestaña para confirmar que se
  desplegaron correctamente.
- Si un certificado está **expirado, por expirar o mal configurado**, recibirás
  una alerta.

---

## Ejemplo por entorno

Combinando la regla de dominios por entorno con certificados propios en `prd` y
manteniendo Let's Encrypt (automático) en `uat`:

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
            "customDomains": ["acme.com", "www.acme.com"],
            "key": "@ssl-key-secret",
            "crt": "@ssl-crt-secret"
          }
        ]
      }
    }
  }
}
```

En este ejemplo:

- `uat.acme.com` usa el certificado automático de Let's Encrypt (entrada sin
  `key`/`crt`).
- `acme.com` y `www.acme.com` usan un certificado propio, referenciado desde
  secrets, con CDN activada.

---

## Referencia de propiedades (`certs`)

| Propiedad       | Descripción                                                                 |
| --------------- | --------------------------------------------------------------------------- |
| `customDomains` | Lista de dominios personalizados que cubre este certificado.                |
| `key`           | Clave del certificado SSL en Base64. Referencia a un secret (prefijo `@`).  |
| `crt`           | Certificado firmado en Base64. Referencia a un secret (prefijo `@`).        |

---

## Resumen del flujo

1. Genera la clave (RSA-2048 o ECDSA P-256, sin passphrase) y el certificado.
2. Codifica ambos en Base64 con `openssl base64`, conservando las etiquetas
   begin/end.
3. Guarda los valores como secrets del entorno.
4. Referencia los secrets en el array `certs` del `loadBalancer`, en el
   `webserver/LCP.json`, agrupados con sus `customDomains`.
5. Despliega un build al `webserver` y reinícialo.
6. Verifica el estado en **Network → SSL Certificates**.

---

## Recordatorios importantes

- Los certificados personalizados **no se renuevan automáticamente**: es tu
  responsabilidad renovarlos antes de que expiren.
- Los certificados del `LCP.json` **tienen prioridad** sobre los de Let's
  Encrypt configurados por consola.
- Solo se aceptan claves/certificados en **PEM + Base64** con las etiquetas de
  encapsulación.
- Cualquier cambio de certificado requiere **redeploy + restart** del
  `webserver`.

---

## Referencias oficiales

- Load Balancer (Custom SSL):
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/load-balancer`
- Custom Domains:
  `https://learn.liferay.com/w/dxp/cloud/configuring-the-cloud-network/custom-domains`
- Managing Secure Environment Variables with Secrets:
  `https://learn.liferay.com/w/dxp/cloud/tuning-security-settings/managing-secure-environment-variables-with-secrets`
