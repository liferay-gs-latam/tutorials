# Configuración de logs personalizados y persistentes en Liferay PaaS

**Referencia:** LRHC-147544
**Entorno:** Liferay PaaS (Liferay DXP 2025.Q1)

---

## 1. Objetivo de este documento

Este documento describe cómo generar archivos de log (`.log`) personalizados en Liferay PaaS y almacenarlos en una ruta definida por usted, de manera que la información no se pierda durante despliegues (deploy) o reinicios del servicio Liferay. También incluye la alternativa recomendada por Liferay y las buenas prácticas a considerar en entornos productivos.

---

## 2. Aclaración importante antes de comenzar

> ⚠️ **Esta configuración se considera una personalización (customization).** Por lo tanto se ofrece como *best effort* y queda **fuera del alcance del soporte estándar** de Liferay. Su implementación, mantenimiento y validación son responsabilidad del equipo del cliente.

Puede consultar el alcance de las personalizaciones en el artículo oficial: *Customizations* (`https://support.liferay.com/v/25988430`). Si desean avanzar por este camino con acompañamiento, su Account Executive puede orientarles sobre las opciones disponibles.

---

## 3. Opción recomendada por Liferay

En Liferay PaaS, los logs del portal **no se eliminan** tras un reinicio o despliegue del servicio: la plataforma los centraliza automáticamente y quedan disponibles en la Consola Cloud. Antes de implementar una solución personalizada, recomendamos evaluar este mecanismo nativo, que no requiere mantenimiento adicional:

- Descargue los logs directamente desde la **Consola Cloud**, seleccionando el rango de días y horas que necesite (hasta aproximadamente un mes de historial en línea).
- Utilice la herramienta de línea de comandos `lcp log` para extraer los registros hacia un entorno local.
- Para descargas de más de 10.000 líneas, siga el artículo *"Not able to download the logs of more than 10.000 lines"* en learn.liferay.com.

Para la mayoría de los casos de investigación de incidentes, este enfoque es más simple, más robusto y evita impactos en el rendimiento del ambiente.

---

## 4. Esquema de logs personalizado (best effort)

Si aún así requieren generar archivos `.log` propios en una ruta específica (por ejemplo, `/opt/liferay/app_logs` o `/mnt/custom-logs`), el procedimiento consta de tres pasos: definir un volumen persistente, configurar Log4j2 para escribir en ese volumen y desplegar la configuración.

### Paso 1 — Definir un volumen persistente (LCP.json)

El sistema de archivos del contenedor es efímero y se reinicia con cada despliegue. Para conservar los archivos, edite el archivo `liferay/LCP.json` del repositorio de su proyecto y agregue una sección `volumes` que mapee un directorio del contenedor a un almacenamiento persistente:

```json
{
   "id": "liferay",
   "memory": 8192,
   "cpu": 8,
   "volumes": {
      "custom-logs": "/mnt/custom-logs"
   }
}
```

En este ejemplo, el directorio `/mnt/custom-logs` persistirá a través de reinicios y despliegues. La clave (`custom-logs`) es un identificador único del volumen; el valor es la ruta absoluta dentro del contenedor.

### Paso 2 — Configurar Log4j2 (portal-log4j-ext.xml)

Indique al framework de logging de Liferay (Log4j2) que escriba en ese directorio mediante un archivo `portal-log4j-ext.xml` que defina un nuevo appender rotativo. Ejemplo:

```xml
<?xml version="1.0"?>
<Configuration strict="true">
    <Appenders>
        <Appender name="CUSTOM_FILE" type="RollingFile"
                  fileName="/mnt/custom-logs/custom-app.log"
                  filePattern="/mnt/custom-logs/custom-app.%d{yyyy-MM-dd}.log">
            <Layout type="PatternLayout">
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5p [%t][%c{1}:%L] %m%n</Pattern>
            </Layout>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="100 MB" />
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </Appender>
    </Appenders>
    <Loggers>
        <!-- Enviar solo los logs de un paquete/clase especifico -->
        <Logger name="com.mi.paquete.custom" level="DEBUG" additivity="false">
            <AppenderRef ref="CUSTOM_FILE"/>
        </Logger>
        <!-- (Opcional) Enviar TODOS los logs del portal a este archivo -->
        <Root level="INFO">
            <AppenderRef ref="CUSTOM_FILE"/>
        </Root>
    </Loggers>
</Configuration>
```

> 💡 **Recomendación:** limite el appender a los paquetes/clases que realmente necesita (bloque `<Logger>`). Enviar todo el portal al archivo (bloque `<Root>`) genera un volumen muy alto de datos y puede afectar el rendimiento.

### Paso 3 — Desplegar la configuración

Coloque el archivo `portal-log4j-ext.xml` en el repositorio Git del proyecto, en la ruta correspondiente a su entorno:

```
liferay/configs/{env}/files/web/WEB-INF/classes/META-INF/portal-log4j-ext.xml
```

Reemplace `{env}` por su entorno (por ejemplo `dev` o `prd`). Al desplegar el proyecto, el archivo se copiará a la ubicación correcta dentro del servicio Liferay y anulará el comportamiento de logging por defecto.

---

## 5. Buenas prácticas y consideraciones de rendimiento

- **Niveles de log:** en producción mantenga `INFO` o `WARN`. Use `DEBUG`/`TRACE` solo de forma temporal para diagnóstico, ya que generan gran volumen de datos.
- **Rotación obligatoria:** configure siempre políticas de rotación (por tamaño y/o por tiempo) para evitar que el archivo crezca sin límite y consuma todo el almacenamiento.
- **Monitoreo del almacenamiento:** el volumen persistente está sujeto a la cuota de su plan. Revise periódicamente el uso para evitar interrupciones del servicio.
- **Impacto en performance:** escribir de forma constante una copia de los logs a disco puede degradar el rendimiento del ambiente cloud. Acote los loggers al mínimo necesario.
- **Extracción de datos:** aun generando estos archivos, para analizarlos deberá descargarlos a una máquina local; el mecanismo nativo de la Consola Cloud sigue siendo la vía recomendada.

---

## 6. Resumen de decisión

| Opción | Descripción | Consideración |
|---|---|---|
| **Recomendada** | Logging nativo de Liferay Cloud (descarga desde la Consola Cloud o `lcp log`) | Sin mantenimiento ni riesgo de rendimiento |
| **Alternativa (best effort)** | Volumen persistente en `LCP.json` + `portal-log4j-ext.xml` | Personalización fuera del alcance de soporte; la implementa y mantiene el cliente |

---

*Documento de referencia basado en el ticket LRHC-147544 (Liferay Help Center). Para acompañamiento sobre personalizaciones, contacte a su Account Executive.*
