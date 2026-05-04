---
name: refactoring
description: Refactoriza y simplifica código existente del proyecto sin cambiar su comportamiento. Úsalo cuando necesites reducir duplicación entre componentes o servicios, extraer lógica repetida a helpers o servicios compartidos, mejorar la legibilidad de un componente complejo, migrar código a patrones más modernos de Angular 21 (signals, nueva sintaxis de templates), o limpiar código muerto.
---

Eres un experto en refactorización de código Angular 21 para el proyecto de sostenibilidad UNIBÁN.

## Principio fundamental

**No cambiar el comportamiento observable.** Un refactor exitoso pasa los mismos tests antes y después. Si no hay tests, escríbelos antes de refactorizar.

---

## Patrones de refactoring más frecuentes en este proyecto

### 1. Extraer lógica duplicada a un helper

Si dos o más componentes hacen la misma transformación de datos:

```typescript
// ANTES — lógica duplicada en dos componentes
// componente-a.component.ts
get opcionesAnio(): number[] {
  const actual = new Date().getFullYear();
  return [actual - 2, actual - 1, actual];
}

// componente-b.component.ts
get aniosDisponibles(): number[] {
  const hoy = new Date();
  return [hoy.getFullYear() - 2, hoy.getFullYear() - 1, hoy.getFullYear()];
}

// DESPUÉS — extraer a helper compartido
// src/app/core/helpers/anio.helper.ts
export function getAniosRecientes(cantidad = 3): number[] {
  const actual = new Date().getFullYear();
  return Array.from({ length: cantidad }, (_, i) => actual - (cantidad - 1 - i));
}
```

### 2. Extraer componente repetido

Si el mismo bloque HTML aparece en múltiples templates:

```typescript
// ANTES — tabla de loading/error/empty repetida en cada componente
// DESPUÉS — componente compartido en src/app/shared/components/

@Component({
  selector: 'app-tabla-estado',
  standalone: true,
  template: `
    @if (loading) { <div class="spinner..."></div> }
    @else if (error) { <div class="error...">{{ error }}</div> }
    @else if (vacio) { <div class="empty..."><ng-content select="[empty]"/></div> }
    @else { <ng-content /> }
  `,
})
export class TablaEstadoComponent {
  @Input() loading = false;
  @Input() error: string | null = null;
  @Input() vacio = false;
}
```

### 3. Migrar a nueva sintaxis de Angular 17+

```html
<!-- ANTES — directivas antiguas -->
<div *ngIf="loading; else content">Cargando...</div>
<ng-template #content>
  <tr *ngFor="let item of items; trackBy: trackById">

<!-- DESPUÉS — nueva sintaxis de control flow -->
@if (loading) {
  <div>Cargando...</div>
} @else {
  @for (item of items; track item.id) {
    <tr>
  }
}
```

### 4. Consolidar servicios pequeños similares

Si hay múltiples servicios con la misma estructura CRUD pero para endpoints distintos, evaluar si se puede usar un servicio genérico con parámetro de endpoint. Ver cómo lo hace el proyecto con `MaestroCrudPageComponent` que ya usa configuración en lugar de herencia.

### 5. Eliminar código muerto

```bash
# Buscar componentes no importados en ningún lado
grep -r "MiComponente" src/ --include="*.ts" | grep -v "spec.ts" | grep -v "mi-componente.component.ts"

# Buscar servicios no inyectados
grep -r "MiServicio" src/ --include="*.ts" | grep -v "spec.ts" | grep -v "mi.service.ts"
```

### 6. Simplificar observables innecesarios

```typescript
// ANTES — Subject innecesario para dato simple
private _loading = new BehaviorSubject(false);
loading$ = this._loading.asObservable();
// usado como: this._loading.next(true)

// DESPUÉS — señal simple si no hay suscriptores externos
loading = false;
// o Angular Signal si se adopta en el proyecto:
loading = signal(false);
```

---

## Proceso de refactoring seguro

1. **Identificar el alcance** — leer todos los archivos afectados antes de tocar nada
2. **Buscar todos los usos** con Grep antes de mover o renombrar
3. **Escribir tests** si el código a refactorizar no los tiene
4. **Un cambio a la vez** — no mezclar refactor con nuevas features
5. **Verificar** que la app compila y los tests pasan después de cada paso

```bash
# Verificar compilación sin errores
npx ng build --configuration=development 2>&1 | tail -20

# Ejecutar tests
npm test
```

---

## Qué NO refactorizar sin coordinación

- **`app.routes.ts`** — cambios de rutas afectan bookmarks y links externos
- **`uniban-auth.service.ts`** — lógica crítica de autenticación
- **Interfaces de tipos** usadas en múltiples archivos — verificar todos los usos primero
- **`MaestroCrudConfig`** y tipos en `maestro-crud.types.ts` — múltiples maestros dependen de esto

---

## Señales de que algo necesita refactoring

- Componente con más de ~300 líneas de TypeScript
- Template con más de ~150 líneas de HTML
- Mismo bloque copiado en 3+ lugares
- Servicio con método de más de 30 líneas
- Lógica de negocio en el template (`(click)="items = items.filter(..."`)
- `any` en tipos donde se puede ser más específico

---

## Antes de empezar

1. Leer el archivo objetivo completamente
2. Grep para encontrar todos los archivos que lo usan/importan
3. Confirmar que los cambios son solo de estructura, no de comportamiento
4. Si el alcance es grande (más de 5 archivos), describir el plan antes de ejecutar
