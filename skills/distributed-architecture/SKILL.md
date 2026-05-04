---
name: distributed-architecture
description: Diseña y audita la interacción entre el frontend UNIBÁN y su ecosistema distribuido (backend Cloud Run, Firebase Auth, APIs REST, almacenamiento de archivos). Úsalo cuando necesites definir contratos de API, manejar fallos de red, implementar retries, circuit breakers, caching de datos, sincronización offline-first, o colas de eventos entre frontend y backend. También para revisar desacoplamiento entre capas.
---

Eres un experto en arquitectura distribuida aplicada al frontend de UNIBÁN. Entiendes cómo una SPA Angular se integra con servicios backend en Google Cloud Run, Firebase y APIs REST.

---

## 1. Contratos de API y Tipado

```typescript
// BIEN — tipado estricto de request/response; contrato compartido
export interface CrearCargaSalarioDignoRequest {
  fincaId: number;
  anio: number;
  archivoUrl: string;
}

export interface CrearCargaSalarioDignoResponse {
  id: number;
  estado: 'pendiente' | 'procesando' | 'completado' | 'error';
  mensaje?: string;
}
```

**Reglas:**
- Nunca usar `any` para payloads de API
- Validar con zod/io-ts si el backend no tiene contrato OpenAPI firme
- Versionar endpoints: `/api/v1/...` si el backend lo soporta

---

## 2. Resiliencia: Retry y Circuit Breaker

```typescript
// Servicio base con retry automático para errores transitorios
import { retry, delay, catchError } from 'rxjs/operators';
import { of, throwError } from 'rxjs';

async listarConResiliencia(): Promise<Item[]> {
  return firstValueFrom(
    this.http.get<Item[]>(this.baseUrl).pipe(
      retry({ count: 2, delay: 1000 }), // reintenta 2 veces con 1s de espera
      catchError(err => {
        // Si falla después de reintentos, propagar como error de negocio
        return throwError(() => new Error('Servicio no disponible. Intenta más tarde.'));
      })
    )
  );
}
```

**Cuándo no reintentar:**
- `400`, `401`, `403`, `404`, `422` → error del cliente, no reintentar
- `500`, `502`, `503`, `504` → error del servidor/transitorio, reintentar

---

## 3. Caching estratégico en el cliente

```typescript
// Cache por dominio con TTL
@Injectable({ providedIn: 'root' })
export class MaestrosCacheService {
  private cache = new Map<string, { data: unknown; timestamp: number }>();
  private readonly TTL_MS = 5 * 60 * 1000; // 5 minutos

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry) return null;
    if (Date.now() - entry.timestamp > this.TTL_MS) {
      this.cache.delete(key);
      return null;
    }
    return entry.data as T;
  }

  set(key: string, data: unknown): void {
    this.cache.set(key, { data, timestamp: Date.now() });
  }

  invalidate(keyPattern: RegExp): void {
    for (const key of this.cache.keys()) {
      if (keyPattern.test(key)) this.cache.delete(key);
    }
  }
}
```

**Reglas:**
- Cachear catálogos/maestros (cambian poco)
- NO cachear datos sensibles ni de salario
- Invalidar cache al crear/actualizar/eliminar

---

## 4. Sincronización Offline-First (si aplica futuro)

```typescript
// Patrón: cola de acciones pendientes
interface AccionPendiente {
  id: string;
  tipo: 'crear' | 'actualizar' | 'eliminar';
  endpoint: string;
  payload: unknown;
  timestamp: number;
}

// Guardar en IndexedDB si no hay conexión
// Reintentar en background cuando vuelva la conexión
```

---

## 5. Manejo de estado de red

```typescript
import { inject, Injectable, signal } from '@angular/core';
import { fromEvent } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ConnectivityService {
  readonly enLinea = signal<boolean>(navigator.onLine);

  constructor() {
    fromEvent(window, 'online').subscribe(() => this.enLinea.set(true));
    fromEvent(window, 'offline').subscribe(() => this.enLinea.set(false));
  }
}
```

---

## 6. Desacoplamiento frontend-backend

```typescript
// MAL — frontend conoce detalles de implementación del backend
if (error.message.includes('duplicate key value violates unique constraint')) {
  // El mensaje es de PostgreSQL; no debería llegar al frontend
}

// BIEN — frontend reacciona a códigos de error del backend
if (error.status === 409) {
  Swal.fire('Conflicto', 'El registro ya existe.', 'warning');
}
```

---

## 7. Checklist de Arquitectura Distribuida

- [ ] Todos los servicios HTTP tienen manejo de errores transitorios
- [ ] Caché tiene TTL e invalidación estratégica
- [ ] El frontend no depende de mensajes de error del motor de base de datos
- [ ] Los contratos de API están tipados en TypeScript
- [ ] Se distingue entre error de red (retry) y error de negocio (no retry)
- [ ] Auth interceptor maneja 401 redirigiendo a login sin bucles infinitos
- [ ] Uploads de archivos usan progress events y manejan cancelación
