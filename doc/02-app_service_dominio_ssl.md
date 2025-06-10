# Configuración de Dominio Personalizado y Certificado SSL en Azure App Service

## 🧾 Requisitos Previos
- Tener una App Service desplegada.
- Tener un dominio personalizado (comprado desde Azure o externo).

---

## ✅ Parte 1: Añadir un Dominio Personalizado

1. Ir al [Portal de Azure](https://portal.azure.com).
2. Buscar y abrir tu **App Service**.
3. En el menú lateral, ir a **Dominios personalizados**.
4. Hacer clic en **+ Agregar dominio personalizado**.
5. Introducir tu dominio (ej: `www.tudominio.com`).
6. Verificar el dominio agregando un **registro CNAME o TXT** en tu DNS.
7. Esperar a que el estado del dominio diga **"Validado"**.

---

## ✅ Parte 2: Crear un Certificado SSL Gratuito

1. Ir a **Configuración de TLS/SSL** > **Certificados públicos (versión preliminar)**.
2. Hacer clic en **+ Crear certificado gratuito**.
3. Seleccionar el dominio validado.
4. Pulsar en **Crear** y esperar a que se genere el certificado.

---

## ✅ Parte 3: Asociar el Certificado SSL (SSL Binding)

1. Ir a **TLS/SSL bindings**.
2. Hacer clic en **+ Agregar binding**.
3. Elegir el dominio personalizado y seleccionar el certificado creado.
4. Elegir tipo: **SNI SSL**.
5. Guardar los cambios.

---

## 🔐 Resultado

Tu App Service ahora funcionará con HTTPS de forma segura y gratuita, usando un certificado gestionado por Azure que se renueva automáticamente.

---

## ⚠️ Limitaciones

- Solo funciona para dominios personalizados (no `*.azurewebsites.net`).
- No se puede exportar el certificado.
- No es compatible con IP-based SSL.