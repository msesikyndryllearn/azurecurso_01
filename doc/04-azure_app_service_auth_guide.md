# Autenticación Integrada en Azure App Service

La autenticación integrada en Azure App Service (también conocida como "Easy Auth") es una característica que permite agregar autenticación y autorización a tus aplicaciones web sin necesidad de escribir código adicional.

## Características principales

### Proveedores de identidad soportados
- Microsoft Entra ID (anteriormente Azure AD)
- Facebook
- Google
- Twitter
- Apple
- Cualquier proveedor compatible con OpenID Connect

### Funcionalidades clave
- Autenticación sin código en el nivel de plataforma
- Manejo automático de tokens
- Integración con Azure Key Vault para secretos
- Soporte para múltiples proveedores simultáneamente
- Autorización basada en roles y claims

## Configuración básica

### Desde Azure Portal
1. Navega a tu App Service
2. Ve a "Authentication" en el menú lateral
3. Haz clic en "Add identity provider"
4. Selecciona el proveedor deseado
5. Configura los parámetros específicos del proveedor

### Configuración mediante ARM templates o CLI
```json
{
  "type": "Microsoft.Web/sites/config",
  "apiVersion": "2022-03-01",
  "name": "[concat(parameters('siteName'), '/authsettingsV2')]",
  "properties": {
    "globalValidation": {
      "requireAuthentication": true,
      "unauthenticatedClientAction": "RedirectToLoginPage"
    },
    "identityProviders": {
      "azureActiveDirectory": {
        "enabled": true,
        "registration": {
          "openIdIssuer": "https://sts.windows.net/{tenant-id}/",
          "clientId": "{client-id}"
        }
      }
    }
  }
}
```

## Flujo de autenticación

1. **Usuario no autenticado** accede a la aplicación
2. **App Service** intercepta la solicitud
3. **Redirección** al proveedor de identidad seleccionado
4. **Usuario se autentica** con el proveedor
5. **Proveedor redirige** de vuelta con el token
6. **App Service valida** el token y crea una sesión
7. **Usuario accede** a la aplicación autenticada

## Acceso a información del usuario

Una vez autenticado, puedes acceder a la información del usuario a través de headers HTTP especiales:

```javascript
// En Node.js
const userInfo = {
  name: req.headers['x-ms-client-principal-name'],
  id: req.headers['x-ms-client-principal-id'],
  idp: req.headers['x-ms-client-principal-idp']
};

// Información completa del principal
const principalHeader = req.headers['x-ms-client-principal'];
const principal = JSON.parse(Buffer.from(principalHeader, 'base64').toString());
```

```csharp
// En C# (.NET)
public class AuthController : Controller
{
    public IActionResult GetUserInfo()
    {
        var name = Request.Headers["X-MS-CLIENT-PRINCIPAL-NAME"].FirstOrDefault();
        var id = Request.Headers["X-MS-CLIENT-PRINCIPAL-ID"].FirstOrDefault();
        var idp = Request.Headers["X-MS-CLIENT-PRINCIPAL-IDP"].FirstOrDefault();
        
        return Ok(new { Name = name, Id = id, Provider = idp });
    }
}
```

```python
# En Python (Flask/FastAPI)
from flask import request

@app.route('/user-info')
def get_user_info():
    user_info = {
        'name': request.headers.get('X-MS-CLIENT-PRINCIPAL-NAME'),
        'id': request.headers.get('X-MS-CLIENT-PRINCIPAL-ID'),
        'provider': request.headers.get('X-MS-CLIENT-PRINCIPAL-IDP')
    }
    return user_info
```

## Configuración avanzada

### Autorización personalizada
- Configurar reglas de autorización específicas
- Implementar middleware personalizado
- Integrar con Azure AD roles y groups

### Gestión de tokens
- Configurar tiempo de vida de tokens
- Habilitar refresh automático
- Acceder a tokens para llamadas a APIs externas

### Endpoints especiales
- `/.auth/login/<provider>` - Iniciar login
- `/.auth/logout` - Cerrar sesión
- `/.auth/me` - Información del usuario actual
- `/.auth/refresh` - Renovar tokens

## Configuración de Microsoft Entra ID

### Registro de aplicación en Azure AD
1. Ve a Azure Active Directory > App registrations
2. Crea una nueva aplicación
3. Configura las URLs de redirección:
   - `https://<your-app-name>.azurewebsites.net/.auth/login/aad/callback`
4. Genera un secreto de cliente
5. Configura los permisos necesarios

### Configuración en App Service
```json
{
  "identityProviders": {
    "azureActiveDirectory": {
      "enabled": true,
      "registration": {
        "openIdIssuer": "https://login.microsoftonline.com/{tenant-id}/v2.0",
        "clientId": "{client-id}",
        "clientSecretSettingName": "AAD_CLIENT_SECRET"
      },
      "validation": {
        "allowedAudiences": [
          "{client-id}"
        ]
      }
    }
  }
}
```

## Configuración de Google

### Obtener credenciales de Google
1. Ve a Google Cloud Console
2. Crea un proyecto o selecciona uno existente
3. Habilita Google+ API
4. Crea credenciales OAuth 2.0
5. Configura orígenes autorizados y URLs de redirección

### Configuración en App Service
```json
{
  "identityProviders": {
    "google": {
      "enabled": true,
      "registration": {
        "clientId": "{google-client-id}",
        "clientSecretSettingName": "GOOGLE_CLIENT_SECRET"
      },
      "validation": {
        "allowedAudiences": [
          "{google-client-id}"
        ]
      }
    }
  }
}
```

## Manejo de errores y troubleshooting

### Errores comunes
- **Error 401**: Token expirado o inválido
- **Error 403**: Usuario no autorizado
- **Redirección infinita**: Configuración incorrecta de URLs

### Logs y debugging
```bash
# Habilitar logs de autenticación
az webapp log config --name <app-name> --resource-group <rg-name> --web-server-logging filesystem

# Ver logs en tiempo real
az webapp log tail --name <app-name> --resource-group <rg-name>
```

## Mejores prácticas

### Seguridad
- Utiliza HTTPS siempre
- Configura CORS apropiadamente
- Implementa logout seguro
- Valida tokens en el backend

### Performance
- Configura cache de tokens apropiadamente
- Minimiza llamadas a endpoints de autenticación
- Utiliza refresh tokens cuando sea posible

### Monitoreo
- Configura alertas para fallos de autenticación
- Monitorea métricas de login/logout
- Implementa logging personalizado

## Ejemplos de implementación

### Proteger rutas específicas
```javascript
// Middleware de autenticación personalizado
function requireAuth(req, res, next) {
    const principal = req.headers['x-ms-client-principal'];
    
    if (!principal) {
        return res.redirect('/.auth/login/aad');
    }
    
    try {
        const user = JSON.parse(Buffer.from(principal, 'base64').toString());
        req.user = user;
        next();
    } catch (error) {
        return res.status(401).json({ error: 'Invalid authentication' });
    }
}

// Aplicar middleware a rutas específicas
app.get('/protected', requireAuth, (req, res) => {
    res.json({ message: 'Welcome!', user: req.user });
});
```

### Autorización basada en roles
```csharp
[Authorize]
public class AdminController : Controller
{
    public IActionResult Index()
    {
        var roles = Request.Headers["X-MS-CLIENT-PRINCIPAL"].FirstOrDefault();
        if (roles != null)
        {
            var principal = JsonSerializer.Deserialize<ClaimsPrincipal>(
                Convert.FromBase64String(roles)
            );
            
            if (principal.IsInRole("Admin"))
            {
                return View();
            }
        }
        
        return Forbid();
    }
}
```

## Scripts de automatización

### PowerShell - Configuración automática
```powershell
# Configurar autenticación AAD
$resourceGroup = "myResourceGroup"
$appName = "myWebApp"
$clientId = "your-client-id"
$tenantId = "your-tenant-id"

az webapp auth update `
    --name $appName `
    --resource-group $resourceGroup `
    --enabled true `
    --action RedirectToLoginPage `
    --aad-allowed-token-audiences $clientId `
    --aad-client-id $clientId `
    --aad-tenant-id $tenantId
```

### Azure CLI - Configuración completa
```bash
#!/bin/bash

# Variables
RESOURCE_GROUP="myResourceGroup"
APP_NAME="myWebApp"
CLIENT_ID="your-client-id"
TENANT_ID="your-tenant-id"

# Habilitar autenticación
az webapp auth update \
    --name $APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --enabled true \
    --action RedirectToLoginPage \
    --aad-allowed-token-audiences $CLIENT_ID \
    --aad-client-id $CLIENT_ID \
    --aad-tenant-id $TENANT_ID

echo "Autenticación configurada exitosamente"
```

## Ventajas y consideraciones

### Ventajas
- Implementación rápida sin código
- Mantenimiento simplificado
- Integración nativa con Azure
- Soporte para múltiples proveedores
- Escalabilidad automática
- Seguridad gestionada por Microsoft

### Consideraciones
- Menor flexibilidad comparado con implementaciones custom
- Dependencia del ecosistema Azure
- Limitaciones en personalización de UI
- Costos adicionales por el servicio
- Curva de aprendizaje para configuraciones avanzadas

## Referencias y recursos adicionales

- [Documentación oficial de Azure App Service Authentication](https://docs.microsoft.com/azure/app-service/overview-authentication-authorization)
- [Azure AD App Registration Guide](https://docs.microsoft.com/azure/active-directory/develop/quickstart-register-app)
- [REST API Reference for App Service Auth](https://docs.microsoft.com/rest/api/appservice/webapps/updateauthsettingsv2)
- [Troubleshooting Authentication Issues](https://docs.microsoft.com/azure/app-service/troubleshoot-authentication-authorization)

---

*Última actualización: Junio 2025*