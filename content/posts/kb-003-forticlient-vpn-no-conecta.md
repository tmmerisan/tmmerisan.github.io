---
title: "VPN FortiClient no conecta — Troubleshooting"
date: 2026-05-17
draft: false
tags: ["VPN", "FortiClient", "Troubleshooting", "Modern Workplace"]
description: "KB #3 · Síntoma, causas y resolución paso a paso"
---

## Síntoma

El usuario abre FortiClient, introduce sus credenciales y no consigue conectar a la VPN corporativa. El error más común es:

> *"Unable to connect to server"*

## Troubleshooting

**1. Verificar configuración del gateway**
Es la causa más frecuente. En FortiClient ir a *Edit connection* y confirmar que el nombre del servidor y el puerto están correctos. El puerto casi siempre es **443**.

**2. Verificar conexión a internet**
Confirmar que el equipo tiene acceso a red antes de continuar.

**3. Validar credenciales corporativas**
Comprobar que la contraseña del usuario no ha expirado. Verificar en Entra ID o Active Directory según el entorno.

**4. Verificar MFA**
Si el entorno usa autenticación multifactor, confirmar que el usuario tiene acceso al método de verificación configurado y que la aprobación se completa correctamente.

**5. Revisar logs de FortiClient**
Acceder a los logs desde la propia interfaz de FortiClient. Los logs dan el error exacto y ayudan a identificar dónde falla la conexión. Documentación oficial: [docs.fortinet.com](https://docs.fortinet.com)

**6. Cerrar y volver a abrir FortiClient**
A veces el cliente queda en un estado inconsistente. Cerrar completamente y reiniciar resuelve el problema.

**7. Reinstalar FortiClient**
Último recurso. Desinstalar completamente y reinstalar desde el repositorio corporativo.

## Posibles causas

- **Gateway mal configurado** — servidor o puerto incorrecto en el cliente
- **Contraseña expirada** — el usuario no puede autenticarse
- **Problema MFA** — el factor secundario no está disponible o no se completa
- **Cliente en estado inconsistente** — se resuelve cerrando y reabriendo
- **Cliente corrupto** — requiere reinstalación limpia

## Validación

El usuario conecta correctamente a la VPN y accede a los recursos corporativos.
