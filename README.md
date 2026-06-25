# 🐔 Api Boreal

Proceso automatizado de extracción de datos desde la API de Boreal (Asimetrix) hacia el datalake en Snowflake. Se ejecuta diariamente en Saturn Cloud y notifica por correo el resultado de cada ejecución.

---

## 📁 Estructura del proyecto

```
api_boreal/
├── auth/
│   └── token_gen.py
├── db/
│   └── database.py
├── logs/
│   └── boreal.log
├── models/
│   └── operadora.py
├── services/
│   ├── mailer.py
│   └── operadora_service.py
├── config.py                  
├── main.py
├── requirements.txt
└── README.md
```

---

## 🔄 Flujo general

```text
main.py
   ↓
get_token()
   ↓
create_session()
   ↓
operadora_service.run()
   ↓
Consulta API Boreal
   ↓
Deduplicación y construcción de variable_id
   ↓
Delete/insert en Snowflake
   ↓
Correo de éxito o error
```

---

## 🗂️ Descripción de archivos

### 🚀 `main.py`

Archivo principal del proceso.

Responsabilidades:

* Configurar el logger del proyecto.
* Obtener el token de autenticación.
* Crear la sesión de conexión a Snowflake.
* Ejecutar el servicio de Operadora Avícola.
* Cerrar la sesión al finalizar.
* Lanzar error si el proceso falla, para que la ejecución quede marcada como fallida.

---

### ⚙️ `config.py`

Archivo central de configuración.

Contiene:

* Configuración de logging.
* URL base de la API Boreal.
* URL de autenticación.
* Usuario y contraseña de la API.
* Timeout de peticiones HTTP.
* Configuración de conexión a Snowflake.
* Nombre de database, schema y tabla destino.
* Configuración del correo de notificaciones.

---

### 🔑 `auth/token_gen.py`

Se encarga de generar el token de autenticación.

Responsabilidades:

* Hacer una petición POST al endpoint de login.
* Enviar usuario y contraseña de la API.
* Extraer el `access_token` de la respuesta.
* Construir el header de autorización.

---

### ❄️ `db/database.py`

Se encarga de crear la sesión hacia Snowflake.

Responsabilidades:

* Cargar la llave privada del usuario de servicio.
* Convertir la llave al formato requerido por Snowflake.
* Crear el engine de conexión.
* Ubicar la sesión en la database y schema configurados.
* Retornar una sesión SQLAlchemy lista para usar.

> 🔒 La conexión usa **Key Pair Authentication** con usuario de servicio.

---

### 🗃️ `models/operadora.py`

Define el modelo ORM de la tabla `OPERADORAAPI`.

Cada atributo del modelo representa una columna en Snowflake.

La llave primaria del modelo es:

```python
variable_id
```

Esta llave se construye durante el proceso combinando:

```text
farm + time + variable_ubidots_id
```

---

### 🔧 `services/operadora_service.py`

Contiene la lógica principal del ETL.

Responsabilidades:

* Calcular el rango de fechas a consultar.
* Consultar la API Boreal.
* Manejar la paginación de resultados.
* Eliminar duplicados.
* Construir objetos del modelo `Operadora`.
* Construir la lista de `variable_id`.
* Eliminar en Snowflake los registros existentes con esos mismos IDs.
* Insertar los nuevos registros en lotes.
* Enviar correo de éxito o error.

---

### 📧 `services/mailer.py`

Contiene las funciones de envío de correo.

Responsabilidades:

* Enviar correo de error con detalle en archivo adjunto.
* Enviar correo de éxito con fecha, duración y cantidad de registros insertados.
* Usar configuración SMTP definida en `config.py`.

---

## 🌐 Fuente de datos

Endpoint principal:

```text
https://cerberus.asimetrix.co/v2/boreal/OPERADORAAVICOLA
```

Endpoint de autenticación:

```text
https://login.asimetrix.co/
```

---

## 🎯 Destino de datos

> ⚠️ Para pruebas, el proceso apunta a:
> ```text
> RAW.API_BOREAL.OPERADORAAPI_PRUEBA
> ```

La tabla productiva es:

```text
RAW.API_BOREAL.OPERADORAAPI
```

---

## 📥 Tipo de carga

El proceso no borra toda la tabla. La estrategia es un **reemplazo por IDs**:

```text
1. Consulta datos del rango de fechas configurado.
2. Elimina duplicados en memoria.
3. Construye variable_id para cada registro.
4. Borra en Snowflake los registros que tengan esos mismos variable_id.
5. Inserta los registros nuevos.
```

---

## ⚙️ Variables de entorno

El proyecto requiere las siguientes variables de entorno configuradas en Saturn Cloud:

### 🔑 API Boreal
| Variable | Descripción |
|---|---|
| `APIUSERNAME` | Usuario de autenticación en la API de Asimetrix |
| `APIPASSWORD` | Contraseña de autenticación en la API de Asimetrix |

### ❄️ Snowflake (usuario de servicio)
| Variable | Descripción |
|---|---|
| `SNOWFLAKE_ACCOUNT` | Identificador de la cuenta Snowflake |
| `SRV_SNOWFLAKEUSER_TI_ANALITICA` | Usuario de servicio |
| `SRV_ROLE` | Rol del usuario de servicio |
| `SNOWFLAKE_WAREHOUSE` | Warehouse de cómputo |
| `SRV_KEY_SNOWFLAKE_TI_ANALITICA` | Llave privada en formato PEM |
| `SRV_PASSPHRASE_SNOWFLAKE_TI_ANALITICA` | Passphrase de la llave privada |

### 📧 Correo
| Variable | Descripción |
|---|---|
| `USERMAIL` | Correo remitente (Gmail) |
| `USERPASSWORD` | Contraseña de aplicación de Gmail |
| `SENDTOEMAILS` | Lista de destinatarios en formato `["correo1@x.com", "correo2@x.com"]` |

---

## 📦 Instalación de dependencias

```bash
pip install -r requirements.txt
```

---

## ▶️ Ejecución

```bash
python main.py
```

---

## 📝 Logs

El proceso genera logs en consola y en el archivo:

```text
logs/boreal.log
```

Los logs registran:

* 🚀 Inicio del proceso
* 🔑 Obtención del token
* 🔌 Conexión a Snowflake
* 🔌 Rango de fechas consultado
* 📦 Cantidad de registros obtenidos
* 📦 Cantidad de registros únicos
* 📥 Inserción por lotes
* 📧 Envío de correos
* ❌ Errores del proceso

---

## 📧 Notificaciones por correo

El proceso envía correo en dos casos:

| Evento | Asunto |
|---|---|
| ✅ Ejecución exitosa | `✅ API BOREAL - Ejecución exitosa` |
| ❌ Error en ejecución | `❌ API BOREAL - Errores detectados` |
