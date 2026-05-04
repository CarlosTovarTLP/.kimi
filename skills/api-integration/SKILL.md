---
name: api-integration
description: Integra y conecta el frontend con el backend de UNIBÁN. Úsalo cuando necesites crear nuevos servicios HTTP, manejar errores de API correctamente, tipar respuestas del backend, depurar llamadas que fallan, o ajustar el interceptor/proxy. Conoce la URL base del backend, el patrón de servicios del proyecto y cómo manejar estados de carga y error en los componentes.
---

Eres un experto en integración frontend-backend para el proyecto de sostenibilidad UNIBÁN.

## Configuración del Backend

**URL producción:** `https://uniban-backend-971613212358.us-east1.run.app`
**URL desarrollo:** proxy local configurado en `proxy.conf.json`

```json
// proxy.conf.json
{ "/api": { "target": "https://uniban-backend-971613212358.us-east1.run.app" } }
```

**En servicios, siempre usar:**
```typescript
import { environment } from '../../../environments/environment';
private baseUrl = `${environment.apiBaseUrl}/api/mi-endpoint`;
// En dev: apiBaseUrl = '' → queda '/api/mi-endpoint' (usa el proxy)
// En prod: apiBaseUrl = 'https://...' → URL completa
```

---

## Patrón estándar de servicio API

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { environment } from '../../../environments/environment';

export interface MiRecursoItem {
  id?: number;
  nombre: string;
  // más campos tipados
}

@Injectable({ providedIn: 'root' })
export class MiRecursoService {
  private http = inject(HttpClient);
  private baseUrl = `${environment.apiBaseUrl}/api/mi-recurso`;

  async listar(): Promise<MiRecursoItem[]> {
    return firstValueFrom(this.http.get<MiRecursoItem[]>(this.baseUrl));
  }

  async listarConFiltros(anio: number, fincaId: number): Promise<MiRecursoItem[]> {
    const params = new HttpParams()
      .set('anio', anio.toString())
      .set('finca_id', fincaId.toString());
    return firstValueFrom(this.http.get<MiRecursoItem[]>(this.baseUrl, { params }));
  }

  async obtener(id: number): Promise<MiRecursoItem> {
    return firstValueFrom(this.http.get<MiRecursoItem>(`${this.baseUrl}/${id}`));
  }

  async crear(item: MiRecursoItem): Promise<MiRecursoItem> {
    return firstValueFrom(this.http.post<MiRecursoItem>(this.baseUrl, item));
  }

  async actualizar(id: number, item: Partial<MiRecursoItem>): Promise<MiRecursoItem> {
    return firstValueFrom(this.http.put<MiRecursoItem>(`${this.baseUrl}/${id}`, item));
  }

  async eliminar(id: number): Promise<void> {
    return firstValueFrom(this.http.delete<void>(`${this.baseUrl}/${id}`));
  }
}
```

---

## Manejo de errores en componentes

```typescript
@Component({...})
export class MiComponent {
  items: MiItem[] = [];
  loading = false;
  error: string | null = null;

  async cargarDatos() {
    this.loading = true;
    this.error = null;
    try {
      this.items = await this.service.listar();
    } catch (err) {
      this.error = 'No se pudieron cargar los datos. Intenta de nuevo.';
      console.error('[MiComponent] Error al cargar:', err);
    } finally {
      this.loading = false;
    }
  }

  async guardar(item: MiItem) {
    try {
      await this.service.crear(item);
      Swal.fire({ title: 'Guardado', icon: 'success', timer: 2000 });
      await this.cargarDatos();
    } catch (err: any) {
      const mensaje = err?.error?.message ?? 'Error al guardar.';
      Swal.fire({ title: 'Error', text: mensaje, icon: 'error' });
    }
  }
}
```

---

## Manejo de errores HTTP (tipos comunes del backend)

```typescript
import { HttpErrorResponse } from '@angular/common/http';

function manejarError(err: unknown): string {
  if (err instanceof HttpErrorResponse) {
    switch (err.status) {
      case 400: return err.error?.message ?? 'Datos inválidos.';
      case 401: return 'Sesión expirada. Inicia sesión de nuevo.';
      case 403: return 'No tienes permiso para esta acción.';
      case 404: return 'El recurso no fue encontrado.';
      case 409: return err.error?.message ?? 'Conflicto con datos existentes.';
      case 422: return err.error?.message ?? 'Error de validación.';
      case 500: return 'Error del servidor. Contacta al administrador.';
      default:  return 'Error inesperado. Intenta de nuevo.';
    }
  }
  return 'Error de conexión.';
}
```

---

## Carga de archivos (upload)

Referencia: `src/app/core/api/archivo.service.ts`

```typescript
async subirArchivo(archivo: File, tipo: string): Promise<{ url: string }> {
  const formData = new FormData();
  formData.append('file', archivo);
  formData.append('tipo', tipo);
  return firstValueFrom(
    this.http.post<{ url: string }>(`${this.baseUrl}/upload`, formData)
    // NO setear Content-Type — HttpClient lo hace automáticamente con boundary
  );
}
```

---

## Interceptor de autenticación

Ya está implementado en `uniban-auth-interceptor.ts`. Agrega automáticamente:
```
Authorization: Bearer {JWT}
```

Si el backend retorna `401`, el interceptor redirige a `/login`. No necesitas manejar esto en cada servicio.

---

## Estados de carga en templates

```html
<!-- Patrón recomendado para tablas/listas -->
@if (loading) {
  <div class="flex justify-center py-12">
    <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-[var(--ub-348c)]"></div>
  </div>
} @else if (error) {
  <div class="bg-red-50 border border-red-200 rounded-lg p-4 text-red-700">
    <p class="font-medium">Error al cargar datos</p>
    <p class="text-sm">{{ error }}</p>
    <button class="btn-secondary mt-2" (click)="cargarDatos()">Reintentar</button>
  </div>
} @else if (items.length === 0) {
  <div class="text-center py-12 text-gray-400">Sin registros.</div>
} @else {
  <!-- tabla de datos -->
}
```

---

## Depuración de llamadas HTTP

```bash
# Ver las requests en la terminal durante desarrollo
# En el navegador: DevTools → Network → filtrar por /api

# Verificar que el proxy está activo
npm start  # debe usar proxy.conf.json

# Si una ruta da 404 en dev, verificar proxy.conf.json
# Si da 401, verificar que el JWT no expiró
# Si da 403, verificar que el usuario tiene el permiso
# Si da 500, revisar los logs del backend
```

---

## Antes de crear un servicio

1. Verificar si ya existe un servicio similar en `src/app/core/api/`, `core/hc-maestros/` o `core/sd-maestros/`
2. Confirmar con el equipo backend el endpoint exacto y los tipos de request/response
3. Revisar `proxy.conf.json` para confirmar que `/api` está proxificado
4. Usar `HttpParams` para query strings — nunca concatenar strings con datos del usuario
