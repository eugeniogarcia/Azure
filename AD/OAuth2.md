# Datos
## Tenant 
```
c2e7c8ec-a7ce-496c-82ae-6dd056f6c099
```

## Resource Apps
- WA
```
ApplicationID:
0eee58b5-825e-4202-92b4-6e155ff734cc
```
- WA1
```
ApplicationID:
489b7587-eee7-45d2-9972-8661fe1f3965
```
## Client Apps
- Test_MP
```
ApplicationID:
a0cd069e-4ce9-4062-a998-1b73a5302aa1

ClientSecret:
n4=cSzqHF:Z?[e1yJh6iAZqd26nYka-I
```
- Test_MP1
```
ApplicationID:
87c2f01a-3bb9-4cf8-a0e1-2c6375dc3482

ClientSecret:
z?bIr[Ji.4sD0szGQ/u0OZaDmKms14p7
```

## Endpoints
### V1
```
https://login.microsoftonline.com/c2e7c8ec-a7ce-496c-82ae-6dd056f6c099/oauth2/token
```
### V2
```
https://login.microsoftonline.com/c2e7c8ec-a7ce-496c-82ae-6dd056f6c099/oauth2/v2.0/token
```
### Configuration
```
https://login.microsoftonline.com/c2e7c8ec-a7ce-496c-82ae-6dd056f6c099/.well-known/openid-configuration
```
### Discovery
Obtenemos las claves publicas usadas por Azure:
```
https://login.microsoftonline.com/common/discovery/keys
```
## Token
En el token podemos encontrar:  
- aud: Respurce Application Id  
- iss: Representa el IDCS  
- azp: Application id de la cliente application - la que obtiene el token  
- azpacr: `enum` que indica de que forma conseguimos el token. 1 significa clientid clientsecret, 2 que se consiguio usando un certificado  

En las validaciones del token se debe comprobar que `oid = sub` para el caso en el que se ha seguido `Client Credentials`. Se debe comprobar tambien que tid = tennantid`

# Conexion al AD
```
Connect-AzureAD -TenantId c2e7c8ec-a7ce-496c-82ae-6dd056f6c099

Get-AzureADDomain

Get-AzureADApplication -SearchString Test_MP

Get-AzureADApplication -ObjectId 1d05e0e1-5929-45bb-931d-db9aa130097f
```

# Fijar el tiempo de vencimiento de los tokens
Podemos definir una politica de tipo `TokenLifetimePolicy` para controlar el vencimiento de los tokens. La politica se puede aplicar de forma globar para toda la `Organizacion`, a nivel de `service principal` o de `aplicacion`. El que nos va a interesar a nosotros, que es el tiempo de vida del token, bien se aplica a nivel de Organizacion o a nivel de service principal.    

Get the service principal:  
```
$sp = Get-AzureADServicePrincipal -Filter "DisplayName eq 'WA1'"
```  

Define the policies. Aqui podemos ver todas las propiedades que podemos aplicar:  

```
$policy = New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00","MaxInactiveTime":"30.00:00:00","MaxAgeMultiFactor":"until-revoked","MaxAgeSingleFactor":"180.00:00:00"}}') -DisplayName "WebApiDefaultPolicyScenario" -IsOrganizationDefault $false -Type "TokenLifetimePolicy"
```  

Para el tiempo de vida el valor minimo son diez minutos:

```
$policy = New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"00:10:00"}}') -DisplayName "WA1" -IsOrganizationDefault $false -Type "TokenLifetimePolicy"
```  

Notese como en las politicas anteriores hemos puesto `-IsOrganizationDefault $false`. En la siguiente politica decidimos aplicar por defecto para todas las aplicaciones 2 horas de duracion:  

```
New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00"}}') -DisplayName "WA" -IsOrganizationDefault $true -Type "TokenLifetimePolicy"
```  

Podemos ver las politicas:  
```
Get-AzureADPolicy -Id $policy.Id
```

```
Get-AzureADPolicy -Id $policy.Id|Format-List  

Id                    : 2b85ad39-0bb9-4a51-8c8f-2654caac6b92
OdataType             :
AlternativeIdentifier :
Definition            : {{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00"}}}
DisplayName           : WA
IsOrganizationDefault : True
KeyCredentials        : {}
Type                  : TokenLifetimePolicy
```

```
Get-AzureADPolicy -Id 2b85ad39-0bb9-4a51-8c8f-2654caac6b92|Format-List

Id                    : 2b85ad39-0bb9-4a51-8c8f-2654caac6b92
OdataType             :
AlternativeIdentifier :
Definition            : {{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00"}}}
DisplayName           : WA
IsOrganizationDefault : True
KeyCredentials        : {}
Type                  : TokenLifetimePolicy
```

Finalmente podemos asignar la plitica a un service principal:  
```
Add-AzureADServicePrincipalPolicy -Id $sp.ObjectId -RefObjectId $policy.Id
```  
Podriamos haberlo hecho tambien a nivel de aplicacion - No estoy seguro de que a nivel aplicacion funcione:  
```
$app = Get-AzureADApplication -Filter "DisplayName eq 'WA1'"

Add-AzureADApplicationPolicy -Id $app.ObjectId -RefObjectId $policy.Id
```  

Podemos ver las politicas, a quien estan aplicadas:  

```
Get-AzureADPolicy -Id 2b85ad39-0bb9-4a51-8c8f-2654caac6b92|Format-List

Id                    : 2b85ad39-0bb9-4a51-8c8f-2654caac6b92
OdataType             :
AlternativeIdentifier :
Definition            : {{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00"}}}
DisplayName           : WA
IsOrganizationDefault : True
KeyCredentials        : {}
Type                  : TokenLifetimePolicy
```

Quien tiene asociada la politica:  
```
Get-AzureADPolicyAppliedObject -Id f9528edd-6757-4614-9f14-7a01135189fa

Id                                   OdataType
--                                   ---------
4c029ec9-83a1-4007-bc7c-fffd64d40bd4 #microsoft.graph.application
a3d65804-3693-427a-940b-1e56c8f3e602 #microsoft.graph.servicePrincipal
```  

Podemos ver quien tiene ese principal:  

```
Get-AzureADServicePrincipal -ObjectId a3d65804-3693-427a-940b-1e56c8f3e602

ObjectId                             AppId                                DisplayName
--------                             -----                                -----------
a3d65804-3693-427a-940b-1e56c8f3e602 489b7587-eee7-45d2-9972-8661fe1f3965 WA1
```

# Especificar Custom claims en los tokens
Para definit ![custom claims](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-claims-mapping) usamos una politica de tipo `ClaimsMappingPolicy`. Aqui estamos indicando que queremos incluir el `IncludeBasicClaimSet` y una serie de claims adhoc `ClaimsSchema`:  
```
$claim=New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true", "ClaimsSchema": [{"Source":"application","ID":"DisplayName","JwtClaimType":"MpId"}]}}') -DisplayName "MpID" -Type "ClaimsMappingPolicy"
```
Otra variante:  
```
$claim=New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true"}}') -DisplayName "ExtraClaimsExample" -Type "ClaimsMappingPolicy"
```
Y una mas, en la que indicamos ademas como figurara el claim en un token SAML:  
```
$claim=New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true", "ClaimsSchema": [{"Source":"application","ID":"appDisplayName","SamlClaimType":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name","JwtClaimType":"name"}]}}') -DisplayName "ExtraClaimsExample" -Type "ClaimsMappingPolicy"
```  
Aplicamos la politica a un service principal:  
```
Add-AzureADServicePrincipalPolicy -Id $sp.ObjectId -RefObjectId $claim.Id
```



# Aplicar una clave privada-publica custom
Para definir una clave privada para un `service principal`, primero debemos obtener un certificado en formato `pfx`. Puede ser un selfsigned. Convertimos el certificado en base 64:  

```
$fileContentBytes = get-content "jwtRS256_1_old.pfx" -Encoding Byte 

[System.Convert]::ToBase64String($fileContentBytes) | Out-File "jwtRS256_1_old.txt"
```
A continuacion, usamos `graph` para aplicar el certificado al `service principal`. Usaremos el siguiente payload:    

```
{"keyCredentials":
[{
                  "startDate": "2019-07-28T22:47:41Z",
                  "endDate": "2019-08-27T22:47:41Z",
                  "type": "X509CertAndPassword",
                  "usage": "Sign",
                  "keyId": "31d78f0d-0b67-4662-90aa-db4ffb0a767e",
                  "value": "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"
            }],
"passwordCredentials":
[{
                  "startDate": "2018-02-22T01:10:00Z",
                  "endDate": "2019-02-22T01:10:00Z",
                  "keyId": "31d78f0d-0b67-4662-90aa-db4ffb0a767e",
                  "value": "Vera1511"
            }]
}
```

Donde, 

- El `keyId` del las entradas `keyCredentials` y `passwordCredentials` tiene que ser el mismo. El valor es un `guid` cualquiera - que podemos generar con cualquier herramienta de creacion de guids
- En `value` tenemos que poner el certificado en formato base64 que hemos creado en `jwtRS256_1_old.txt` en el paso anterior
- `startDate` y `endDate` tienen que coincidir con las fechas de inicio y fin especificadas en el certificado  

Para aplicar el payload usaremos la api de `graph`, y más en concreto un `PATCH` en la api `https://graph.windows.net/myorganization/servicePrincipals/{service principal id}`.  



## Cmslets para manejar claves:
En ![el repositorio](https://github.com/AzureAD/azure-activedirectory-powershell-tokenkey) tenemos una serie de cmdlets que nos permite gestionar las claves publicas y privadas. Solo me funcionaron los que recuperan las claves publicas, no los que actualizan:  

```
Install-Module AADInternals

Connect-AzureAD -TenantId c2e7c8ec-a7ce-496c-82ae-6dd056f6c099

Get-AzureADDomain

Get-AzureADObjectSetting

Get-AADObject -Type servicePrincipals -Query "`&`$filter=appId eq '$ApplicationId'"
```

```
.\Get-AADSigningKey.ps1 | Format-Table
.\Get-AADSigningKey.ps1 -Latest
.\Get-AADSigningKey.ps1 -Latest -DownloadPath C:\Users\Eugenio\Downloads\
```
