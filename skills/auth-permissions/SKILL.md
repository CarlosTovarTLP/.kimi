---
name: auth-permissions
description: Maneja todo lo relacionado con autenticación, permisos, roles y acceso a rutas. Úsalo cuando necesites agregar nuevos permisos, crear roles, proteger rutas nuevas, ajustar la lógica de guards, modificar el interceptor HTTP, o depurar problemas de acceso en la aplicación.
---

Eres un experto en el sistema de autenticación y permisos del proyecto de sostenibilidad UNIBÁN en Angular 21.

## Archivos Clave del Sistema de Auth/Permisos

```
src/app/core/
├── auth/
│   ├── uniban-auth.service.ts        ← Servicio principal de auth (JWT + Firebase)
│   ├── uniban-auth-interceptor.ts    ← Añade Bearer token a requests HTTP
│   └── permission.helper.ts          ← Funciones hasAnyPermission(), isValidAction()
├── guards/
│   ├── uniban-auth.guard.ts          ← Valida JWT válido y no expirado
│   └── permission.guard.ts           ← Valida permiso del usuario vs ruta
└── constants/
    └── permisos.constants.ts         ← PERMISOS y ACCIONES_PERMISOS
```

## Sistema de Permisos

### Constantes definidas en `permisos.constants.ts`

```typescript
export const PERMISOS = {
  SALARIO_DIGNO: 'salario_digno',
  HUELLA_CARBONO: 'huella_carbono',
  // ... más permisos
};

export const ACCIONES_PERMISOS = {
  VER: 'ver',
  CREAR: 'crear',
  EDITAR: 'editar',
  ELIMINAR: 'eliminar',
  // ...
};
```

### Cómo verificar permisos en componentes

```typescript
import { hasAnyPermission, isValidAction } from '../../core/auth/permission.helper';
import { UnibanAuthService } from '../../core/auth/uniban-auth.service';
import { PERMISOS, ACCIONES_PERMISOS } from '../../core/constants/permisos.constants';

@Component({...})
export class MiComponente {
  private authService = inject(UnibanAuthService);

  get puedeEditar(): boolean {
    const user = this.authService.currentUser;
    return isValidAction(user, PERMISOS.MI_MODULO, ACCIONES_PERMISOS.EDITAR);
  }
}
```

### Cómo proteger una ruta

```typescript
// En app.routes.ts o en el archivo de rutas del módulo
{
  path: 'mi-ruta',
  component: MiComponent,
  data: { permission: PERMISOS.MI_PERMISO },
  canActivate: [PermissionGuard],
}
```

## Flujo de Autenticación

1. Usuario hace login en Firebase (`src/app/auth/login/`)
2. Se obtiene `idToken` de Firebase
3. Se llama al backend con el `idToken` → backend retorna JWT propio
4. `UnibanAuthService` guarda JWT en `localStorage` con clave `uniban_jwt`
5. `UnibanAuthInterceptor` añade el JWT como `Bearer` en cada request HTTP
6. `UnibanAuthGuard` valida que el JWT no esté expirado en cada navegación

## Agregar nuevo permiso

1. Agregar constante en `PERMISOS` dentro de `permisos.constants.ts`
2. Agregar al backend (coordinación con equipo backend)
3. Asignar permiso al rol correspondiente en Administración → Roles
4. Proteger rutas con `data: { permission: PERMISOS.NUEVO }` + `PermissionGuard`
5. Filtrar navegación: agregar `permission` al `NavItem` en `navigation.config.ts`

## UnibanAuthService - API pública

```typescript
user$: Observable<UnibanUser | null>         // Observable del usuario actual
currentUser: UnibanUser | null               // Getter síncrono del usuario
isAuthenticated: boolean                     // Si hay sesión activa
loginWithFirebaseIdToken(idToken): Promise   // Login con Firebase token
logout(): void                               // Cierra sesión
restoreSession(): void                       // Restaura sesión desde localStorage
isJwtExpired(): boolean                      // Verifica si el JWT expiró
```

## Estructura UnibanUser

Leer `uniban-auth.service.ts` para ver la interfaz exacta `UnibanUser`. Generalmente incluye:
- `id`, `nombre`, `email`
- `permisos: string[]` — lista de claves de permisos asignados
- `rol`, `fincas` — fincas asociadas al usuario

## Depuración de problemas de acceso

1. Verificar que el JWT no esté expirado: `authService.isJwtExpired()`
2. Verificar que el usuario tiene el permiso: `hasAnyPermission(user, PERMISOS.X)`
3. Verificar que la ruta tiene `data.permission` configurado correctamente
4. Verificar que `PermissionGuard` está en el array `canActivate` de la ruta
5. Revisar la consola del navegador para errores 401/403 del backend

## Antes de modificar

Siempre leer primero:
- `permisos.constants.ts` — para ver permisos existentes y evitar duplicados
- `permission.helper.ts` — para entender la lógica de validación
- `uniban-auth.guard.ts` y `permission.guard.ts` — antes de modificar guards
