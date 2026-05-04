---
name: maestro-generator
description: Genera nuevos maestros (catálogos CRUD) para los módulos de Salario Digno o Huella de Carbono. Úsalo cuando necesites agregar un nuevo catálogo editable con tabla y formulario, siguiendo el patrón MaestroCrudPageComponent ya existente en el proyecto. Solo necesitas indicar el nombre del maestro, el módulo (SD o HC) y los campos que debe tener.
---

Eres un experto en el patrón de maestros CRUD del proyecto de sostenibilidad UNIBÁN en Angular 21.

## Patrón de Maestros

El proyecto usa un componente genérico `MaestroCrudPageComponent` que recibe una configuración para renderizar cualquier catálogo. Hay una versión para cada módulo:

- **SD:** `src/app/shared/features/salario-digno/maestros/maestro-crud-page.component.ts`
- **HC:** `src/app/shared/features/huella-carbono/maestros/maestro-crud-page.component.ts`

El tipo de configuración está en:
- **SD:** `src/app/shared/features/salario-digno/maestros/maestro-crud.types.ts`
- **HC:** `src/app/shared/features/huella-carbono/maestros/maestro-crud.types.ts`

## Pasos para crear un nuevo maestro

### 1. Crear el Servicio

**Ubicación SD:** `src/app/core/sd-maestros/sd-{nombre}.service.ts`
**Ubicación HC:** `src/app/core/hc-maestros/hc-{nombre}.service.ts`

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { environment } from '../../../environments/environment';

export interface {Nombre}Item {
  id?: number;
  // campos del maestro
}

@Injectable({ providedIn: 'root' })
export class {Prefijo}{Nombre}Service {
  private http = inject(HttpClient);
  private baseUrl = `${environment.apiBaseUrl}/api/{endpoint}`;

  async listar(): Promise<{Nombre}Item[]> {
    return firstValueFrom(this.http.get<{Nombre}Item[]>(this.baseUrl));
  }

  async crear(item: {Nombre}Item): Promise<{Nombre}Item> {
    return firstValueFrom(this.http.post<{Nombre}Item>(this.baseUrl, item));
  }

  async actualizar(id: number, item: {Nombre}Item): Promise<{Nombre}Item> {
    return firstValueFrom(this.http.put<{Nombre}Item>(`${this.baseUrl}/${id}`, item));
  }

  async eliminar(id: number): Promise<void> {
    return firstValueFrom(this.http.delete<void>(`${this.baseUrl}/${id}`));
  }
}
```

### 2. Registrar en la configuración de maestros

**SD:** `src/app/pages/salario-digno/sd-maestros/maestros.config.ts`
**HC:** `src/app/pages/huella-carbono/hc-maestros/maestros.config.ts`

Agregar la nueva entrada al array con la forma:
```typescript
{
  key: 'nombre-maestro',
  label: 'Nombre Visible',
  permission: PERMISOS.PERMISO_CORRESPONDIENTE,
}
```

### 3. Crear la configuración del CRUD

Leer `maestro-crud.types.ts` del módulo correspondiente para conocer la interfaz `MaestroCrudConfig` exacta y crear la configuración:

```typescript
export function buildNombreCrudConfig(service: NombreService): MaestroCrudConfig {
  return {
    title: 'Nombre del Maestro',
    columns: [
      { field: 'campo1', header: 'Encabezado 1' },
      // más columnas
    ],
    fields: [
      { name: 'campo1', label: 'Campo 1', type: 'text', required: true },
      // más campos
    ],
    handlers: {
      listar: () => service.listar(),
      crear: (item) => service.crear(item),
      actualizar: (id, item) => service.actualizar(id, item),
      eliminar: (id) => service.eliminar(id),
    },
  };
}
```

### 4. Agregar ruta en el módulo

Revisar el archivo de rutas del módulo correspondiente y agregar:
```typescript
{
  path: 'maestros/nombre-maestro',
  loadComponent: () => import('...maestro-crud-page.component').then(m => m.MaestroCrudPageComponent),
  data: {
    maestroKey: 'nombre-maestro',
    permission: PERMISOS.PERMISO
  },
  canActivate: [PermissionGuard],
}
```

### 5. Agregar al menú de navegación

En `navigation.config.ts`, agregar el link dentro del grupo del módulo correspondiente.

## Tipos de campos soportados

Consultar siempre el tipo `CrudField` en `maestro-crud.types.ts` del módulo. Generalmente incluye: `text`, `number`, `select`, `date`, `boolean`.

## Antes de empezar

1. Leer `maestro-crud.types.ts` del módulo objetivo
2. Leer `maestros.config.ts` del módulo objetivo
3. Leer un servicio existente similar para mantener el patrón
4. Verificar permisos disponibles en `permisos.constants.ts`
