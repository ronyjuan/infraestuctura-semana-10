 # Informe de Laboratorio: Observabilidad

## 13. Preguntas a responder

### 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?
Prometheus es genial para los números. Nos dice "qué" está fallando y cuándo (por ejemplo, si el CPU subió al 90% o si hay muchos errores HTTP). Pero los números no te dicen el motivo exacto del error dentro del código. 

Ahí es donde entra Loki. Loki guarda los textos de los logs del sistema en tiempo real. Entonces, si Prometheus te avisa con una alerta que algo anda mal, vas a Loki para ver la línea exacta de código o el mensaje de error que causó el problema. Se complementan: uno te avisa del problema y el otro te muestra el porqué.

### 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
La gran ventaja es que no dependes de la memoria de nadie ni de configurar las cosas a mano haciendo clics en la web, donde cualquiera se puede equivocar. 

Al dejar todo configurado en archivos de texto (como archivos YAML), si el servidor se borra o si queremos levantar el laboratorio en otra computadora (`docker compose down -v` y luego subirlo), todo vuelve a aparecer configurado exactamente igual de forma automática. Además, queda un registro guardado en Git de cualquier cambio que se haga.

### 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
Muestran valores distintos porque miden cosas diferentes:
* El **CPU del host** mide el esfuerzo de toda la máquina virtual en general (incluyendo el sistema operativo y cualquier otro proceso que esté corriendo de fondo).
* El **CPU del contenedor** mide única y exclusivamente los recursos que está gastando nuestro contenedor específico de la aplicación (el backend).

Para alertar sobre una aplicación concreta, se debe usar obligatoriamente el del **contenedor**. Si la máquina en general está libre (p. ej. al 15%), pero nuestra aplicación se trabó y está consumiendo el 100% de lo que tiene permitido, la alerta del host nunca se enteraría y la app se caería sin que nos diéramos cuenta.

### 4. ¿Qué diferencia hay entre el *evaluation interval* y el *pending period* de una alarma?
* El **Evaluation interval** es cada cuánto tiempo Grafana va a revisar si se cumple la condición de la alerta (por ejemplo, configuramos que revise cada 10 segundos el estado del CPU).
* El **Pending period** es un tiempo de espera o "de gracia" antes de que la alerta se ponga en rojo (Firing). Si pusimos 30 segundos, significa que el CPU tiene que estar alto de forma continua durante esos 30 segundos. Esto evita que si el CPU sube un solo segundo por un proceso rápido, se dispare una alarma falsa por molestar.

---

##  Instrucciones para validar el trabajo desarrollado

Para comprobar que todo el sistema de observabilidad, el mapeo de métricas y el ciclo cerrado de alertas e infraestructura mediante Webhooks funcionan correctamente, siga estos pasos:

1. Estado inicial en reposo
Abra la interfaz web de Grafana y diríjase a Alerting ➔ Alert rules.

Podrá observar que la regla llamada CPU backend > 50% dentro del grupo evaluacion-backend se encuentra inicialmente en estado verde (Normal), ya que el sistema no está recibiendo carga de trabajo.

2. Ejecutar la prueba de estrés
Para forzar y simular una saturación real en el backend, abra la terminal del entorno de desarrollo y ejecute el siguiente comando para generar carga continua por 60 segundos:

curl "http://localhost:3001/load?seconds=60"

3. Monitoreo de la alerta en tiempo real
Regrese de inmediato a la sección de Alert rules en Grafana.

Verá que la alerta cambia rápidamente a color amarillo (Pending) al detectar la subida de CPU.

Una vez transcurrido el tiempo de gracia configurado (Pending period), la alerta cambiará de forma automática a color rojo (Firing), indicando que se ha disparado formalmente.

4. Verificación del ciclo cerrado (Alerta ➔ Webhook ➔ Log)
Diríjase al Dashboard de Infraestructura Global que se encuentra configurado en Grafana.

Observe fijamente el panel inferior titulado "Logs de infraestructura" (mapeado mediante Loki).

Verá que acaba de ingresar un registro de manera automática enviado por el Webhook de la aplicación que confirma la recepción del evento desde Grafana:

Alerta recibida: [Firing] CPU backend > 50%

Al cabo de los 60 segundos, cuando el comando curl finalice su ejecución, el uso de CPU caerá por sí solo y la regla de alerta regresará automáticamente a su estado original verde (Normal), cerrando el flujo completo de monitoreo.