# Configuración de Dominio Personalizado y Certificado SSL en Azure Static Web Apps

## 🧾 Requisitos Previos
- Tener una Static Web App desplegada.
- Tener un dominio personalizado registrado.

---

## ✅ Parte 1: Añadir un Dominio Personalizado

1. Ir al [Portal de Azure](https://portal.azure.com).
2. Buscar y abrir tu **Static Web App**.
3. Ir a **Custom domains** en el menú lateral.
4. Hacer clic en **+ Add**.
5. Escribir tu dominio personalizado (ej: `www.tudominio.com`).
6. Seguir las instrucciones para añadir un **registro CNAME o TXT** en tu proveedor DNS.
7. Esperar a que Azure valide el dominio.

---

## ✅ Parte 2: Certificado SSL Gratuito (Automático)

- Una vez que el dominio esté validado, Azure **genera automáticamente un certificado SSL gratuito**.
- No es necesario realizar ninguna acción adicional.
- El certificado se renueva automáticamente.

---

## 🔐 Resultado

Tu sitio web funcionará automáticamente con **HTTPS**, usando un certificado gratuito gestionado por Azure.

---

## ⚠️ Limitaciones

- No se puede cargar un certificado propio.
- El certificado solo es válido para el dominio exacto configurado.
- Los dominios raíz (`midominio.com`) requieren soporte especial en DNS (ALIAS o flattening).