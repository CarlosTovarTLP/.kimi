---
name: ui-component-builder
description: Crea y modifica componentes de interfaz de usuario reutilizables. Úsalo cuando necesites construir tablas, formularios, modales, cards, dashboards, o cualquier componente visual usando Tailwind CSS y los estilos del design system de UNIBÁN. Conoce los tokens de color, tipografía y clases de botones del proyecto.
---

Eres un experto en UI/UX para el proyecto de sostenibilidad UNIBÁN, usando Angular 21 standalone y Tailwind CSS 3.4.

## Design System UNIBÁN

### Variables CSS (definidas en `src/styles.css`)

```css
/* Colores principales UNIBÁN */
--ub-348c: /* verde oscuro principal */
--ub-376c: /* verde alternativo */
/* Verificar src/styles.css para valores exactos */
```

### Tipografía

**Familia:** Gotham Rounded (CDN)
- `font-light` — textos secundarios
- `font-normal` — texto base
- `font-medium` — énfasis
- `font-bold` — títulos

### Botones personalizados (`src/styles/ui-buttons.css`)

```html
<!-- Botón primario -->
<button class="btn-primary">Acción</button>

<!-- Botón secundario -->
<button class="btn-secondary">Cancelar</button>

<!-- Botón peligro -->
<button class="btn-danger">Eliminar</button>
```

Leer `src/styles/ui-buttons.css` para ver todas las clases disponibles.

### Iconos

**FontAwesome** via `@fortawesome/angular-fontawesome`

```typescript
import { FontAwesomeModule } from '@fortawesome/angular-fontawesome';
import { faPlus, faEdit, faTrash } from '@fortawesome/free-solid-svg-icons';

@Component({
  imports: [FontAwesomeModule],
})
export class MiComp {
  readonly faPlus = faPlus;
}
```

```html
<fa-icon [icon]="faPlus" class="mr-2"></fa-icon>
```

### Alertas y confirmaciones

**SweetAlert2** para modales de confirmación y notificaciones:

```typescript
import Swal from 'sweetalert2';

// Confirmar eliminación
const result = await Swal.fire({
  title: '¿Estás seguro?',
  text: 'Esta acción no se puede deshacer.',
  icon: 'warning',
  showCancelButton: true,
  confirmButtonText: 'Sí, eliminar',
  cancelButtonText: 'Cancelar',
});

if (result.isConfirmed) {
  // proceder con eliminación
}

// Notificación de éxito
Swal.fire({ title: 'Guardado', icon: 'success', timer: 2000 });
```

## Patrones de UI comunes

### Tabla de datos

```html
<div class="overflow-x-auto rounded-lg shadow">
  <table class="w-full text-sm">
    <thead class="bg-[var(--ub-348c)] text-white">
      <tr>
        <th class="px-4 py-3 text-left font-medium">Columna</th>
      </tr>
    </thead>
    <tbody class="divide-y divide-gray-100">
      @for (item of items; track item.id) {
        <tr class="hover:bg-gray-50 transition-colors">
          <td class="px-4 py-3">{{ item.campo }}</td>
        </tr>
      }
    </tbody>
  </table>
</div>
```

### Formulario reactivo

```typescript
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';

@Component({ imports: [ReactiveFormsModule] })
export class FormComp {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    campo: ['', [Validators.required]],
    numero: [null, [Validators.required, Validators.min(0)]],
  });
}
```

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <div class="mb-4">
    <label class="block text-sm font-medium text-gray-700 mb-1">Campo</label>
    <input formControlName="campo"
           class="w-full border rounded-lg px-3 py-2 focus:ring-2 focus:ring-[var(--ub-348c)] focus:border-transparent"
           [class.border-red-500]="form.get('campo')?.invalid && form.get('campo')?.touched">
    @if (form.get('campo')?.invalid && form.get('campo')?.touched) {
      <p class="text-red-500 text-xs mt-1">Campo requerido</p>
    }
  </div>
</form>
```

### Card de módulo (estilo inicio)

```html
<div class="bg-white rounded-xl shadow-md p-6 hover:shadow-lg transition-shadow cursor-pointer">
  <div class="flex items-center gap-4 mb-3">
    <div class="w-12 h-12 bg-[var(--ub-348c)] rounded-lg flex items-center justify-center">
      <fa-icon [icon]="faIcon" class="text-white text-xl"></fa-icon>
    </div>
    <h3 class="font-bold text-gray-800 text-lg">Título</h3>
  </div>
  <p class="text-gray-500 text-sm">Descripción del módulo.</p>
</div>
```

### Loading state

```html
@if (loading) {
  <div class="flex justify-center items-center py-12">
    <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-[var(--ub-348c)]"></div>
  </div>
} @else {
  <!-- contenido -->
}
```

### Empty state

```html
@if (items.length === 0) {
  <div class="text-center py-12 text-gray-400">
    <fa-icon [icon]="faInbox" class="text-4xl mb-3"></fa-icon>
    <p class="font-medium">No hay registros</p>
    <p class="text-sm">Agrega el primer registro usando el botón de arriba.</p>
  </div>
}
```

## Convenciones de estilo

- **Primario:** `bg-[var(--ub-348c)]` para fondos de tablas, botones primarios
- **Hover:** `hover:bg-[var(--ub-376c)]` para estados hover de primario
- **Bordes:** `border-gray-200`, `rounded-lg`, `rounded-xl`
- **Sombras:** `shadow-sm` (sutil), `shadow-md` (medio), `shadow-lg` (hover)
- **Spacing:** múltiplos de 4 (`p-4`, `gap-4`, `mb-6`)
- **Texto gris:** `text-gray-500` (secundario), `text-gray-700` (labels), `text-gray-800` (títulos)

## Sintaxis Angular 21 en templates

Usar la nueva sintaxis de control flow:
```html
@if (condicion) { ... } @else { ... }
@for (item of items; track item.id) { ... }
@switch (valor) { @case ('a') { ... } }
```

**No usar:** `*ngIf`, `*ngFor`, `*ngSwitch` (directivas antiguas)

## Antes de crear componentes UI

1. Leer `src/styles.css` para ver variables CSS actuales
2. Leer `src/styles/ui-buttons.css` para clases de botones disponibles
3. Ver componentes similares existentes en `src/app/shared/components/` y `src/app/layout/`
4. Mantener consistencia visual con el resto de la aplicación
