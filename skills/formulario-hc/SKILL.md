---
name: formulario-hc
description: Crea o modifica formularios de Información Corporativa del módulo Huella de Carbono. Úsalo cuando necesites agregar un nuevo formulario de captura de datos para HC (combustibles, vuelos, gases, energía, producción, etc.) siguiendo el patrón InfoCorporativaCrudComponent. También útil para corregir o extender formularios existentes.
---

Eres un experto en los formularios de Huella de Carbono del proyecto de sostenibilidad UNIBÁN en Angular 21.

## Contexto del Módulo HC - Información Corporativa

**Ubicación de formularios:** `src/app/shared/features/huella-carbono/info-corporativa/`

**Componente genérico:** `InfoCorporativaCrudComponent` — recibe configuración `InfoCorporativaCrudConfig`

**Tipos:** `src/app/shared/features/huella-carbono/info-corporativa/info-corporativa.types.ts` (verificar siempre)

**Servicios de datos:** `src/app/core/hc-info-corporativa/` — un servicio por formulario

**Configuración de formularios activos:** `src/app/pages/huella-carbono/hc-info-corporativa/info-corporativa.config.ts`

## Formularios HC ya implementados (referencia)

Leer los existentes para entender el patrón antes de crear uno nuevo:
- Combustible (`hc-combustible.service.ts`)
- Vuelos (`hc-vuelos.service.ts`)
- Contenedores (`hc-contenedores.service.ts`)
- Gases refrigerantes (`hc-gases.service.ts`)
- Energía eléctrica
- Producción anual
- Entre otros en `src/app/core/hc-info-corporativa/`

## Proceso para nuevo formulario HC

### 1. Crear interfaz y servicio

**Ubicación:** `src/app/core/hc-info-corporativa/hc-{nombre}.service.ts`

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { environment } from '../../../environments/environment';

export interface Hc{Nombre}Item {
  id?: number;
  anio?: number;
  finca_id?: number;
  // campos específicos del formulario
}

@Injectable({ providedIn: 'root' })
export class Hc{Nombre}Service {
  private http = inject(HttpClient);
  private baseUrl = `${environment.apiBaseUrl}/api/hc/{endpoint}`;

  async listar(anio: number, fincaId: number): Promise<Hc{Nombre}Item[]> {
    return firstValueFrom(
      this.http.get<Hc{Nombre}Item[]>(`${this.baseUrl}?anio=${anio}&finca_id=${fincaId}`)
    );
  }

  async crear(item: Hc{Nombre}Item): Promise<Hc{Nombre}Item> {
    return firstValueFrom(this.http.post<Hc{Nombre}Item>(this.baseUrl, item));
  }

  async actualizar(id: number, item: Hc{Nombre}Item): Promise<Hc{Nombre}Item> {
    return firstValueFrom(this.http.put<Hc{Nombre}Item>(`${this.baseUrl}/${id}`, item));
  }

  async eliminar(id: number): Promise<void> {
    return firstValueFrom(this.http.delete<void>(`${this.baseUrl}/${id}`));
  }
}
```

### 2. Agregar configuración

En `info-corporativa.config.ts`, agregar entrada al mapa de formularios disponibles.

### 3. Registrar ruta

En las rutas de HC, agregar la ruta del formulario con el componente genérico `InfoCorporativaCrudComponent` y la data del formulario configurado.

### 4. Agregar al menú de navegación

En `navigation.config.ts`, dentro del grupo de HC → Información Corporativa.

## Columnas y campos comunes en HC

Los formularios de información corporativa típicamente incluyen:
- `anio` — año de reporte (number, obligatorio)
- `finca_id` — finca asociada (select, obligatorio)
- Campos específicos por tipo de dato (consumo, cantidad, distancia, etc.)
- `observaciones` — campo libre opcional

## Antes de comenzar

1. Leer `info-corporativa.types.ts` para conocer `InfoCorporativaCrudConfig`
2. Leer un servicio similar en `hc-info-corporativa/` para mantener el patrón
3. Leer `info-corporativa.config.ts` para ver cómo se registran los formularios
4. Verificar permisos en `permisos.constants.ts`
5. Revisar si el endpoint del backend ya existe o si es nuevo

## Validaciones importantes

- Siempre verificar que `anio` y `finca_id` estén presentes en formularios de info corporativa
- Los campos numéricos de emisiones/consumo deben tener `min: 0`
- Los selects deben cargar sus opciones de los maestros correspondientes (distancias, factores de emisión, etc.)
