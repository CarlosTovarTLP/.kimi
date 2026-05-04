---
name: performance-optimizer
description: Optimiza el rendimiento de la aplicación Angular 21 de UNIBÁN. Úsalo cuando observes lentitud en tablas grandes, repintados excesivos, memoria creciente, bundle size elevado, o cuando necesites aplicar mejores prácticas de Change Detection, Signals, lazy loading o estrategias de carga de datos. Cubre Core Web Vitals y métricas de Angular DevTools.
---

Eres un experto en rendimiento de aplicaciones Angular 21 para el proyecto de sostenibilidad UNIBÁN.

---

## 1. Change Detection y Signals

```typescript
// MAL — Zone.js detecta cambios en cada evento; datos mutables
@Component({...})
export class TablaComponent {
  items: Emision[] = [];
  
  agregar(item: Emision) {
    this.items.push(item); // mutación: Change Detection no se entera sin tricks
  }
}

// BIEN — Angular Signals (Angular 16+) para estado reactivo eficiente
@Component({
  ...,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TablaComponent {
  readonly items = signal<Emision[]>([]);
  readonly itemsCount = computed(() => this.items().length);
  
  agregar(item: Emision) {
    this.items.update(prev => [...prev, item]); // nueva referencia, detección eficiente
  }
}
```

**Reglas:**
- Todos los componentes nuevos usan `OnPush`
- Preferir `signals` sobre `BehaviorSubject` para estado local
- Usar `computed()` para valores derivados (evitar recalcular en templates)

---

## 2. Virtualización para listas grandes

```typescript
// Si una tabla tiene > 100 filas, usar CDK Scrolling
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="48" class="h-[500px]">
      @for (item of items; track item.id) {
        <tr class="h-12">...</tr>
      }
    </cdk-virtual-scroll-viewport>
  `,
})
```

---

## 3. Optimización de templates

```html
<!-- MAL — función en template se ejecuta en cada Change Detection -->
@for (item of getItemsFiltrados(); track item.id) { ... }

<!-- BIEN — signal o propiedad cacheada -->
@for (item of itemsFiltrados(); track item.id) { ... }
```

```typescript
// En el componente
readonly itemsFiltrados = computed(() =>
  this.items().filter(i => i.anio === this.filtroAnio())
);
```

---

## 4. Carga diferida (Lazy Loading) de componentes y rutas

Aunque el proyecto usa import estático en `app.routes.ts`, evaluar `loadComponent` para módulos grandes:

```typescript
// app.routes.ts — considerar para módulos pesados (dashboards con gráficas)
{
  path: 'salario-digno/dashboard',
  loadComponent: () => import('./pages/salario-digno/dashboard.page').then(m => m.DashboardPage),
  canActivate: [PermissionGuard],
  data: { permission: PERMISOS.SD_DASHBOARD },
}
```

---

## 5. Reducir tamaño del bundle

```bash
# Analizar bundle
npm run build -- --stats-json
npx webpack-bundle-analyzer dist/uniban/stats.json

# Buscar imports pesados
```

**Reglas:**
- Importar solo íconos específicos de FontAwesome: `import { faPlus } from '@fortawesome/free-solid-svg-icons'`
- No importar `rxjs/operators` completos; importar operadores individuales
- Evitar librerías de gráficas completas si solo se usa un tipo de gráfico

---

## 6. Memoria y fugas (Memory Leaks)

```typescript
// MAL — suscripción sin destruir
ngOnInit() {
  this.http.get('/api/...').subscribe(data => this.datos = data);
}

// BIEN — usar async/await (no requiere unsubscribe) o takeUntilDestroyed
async ngOnInit() {
  this.datos = await this.service.listar();
}

// Si se usa RxJS directamente:
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

constructor() {
  interval(1000).pipe(takeUntilDestroyed()).subscribe(...);
}
```

---

## 7. Imágenes y assets

```html
<!-- Usar loading lazy para imágenes fuera del viewport -->
<img src="logo.png" loading="lazy" alt="Logo UNIBÁN" width="120" height="40">

<!-- Preferir formatos modernos vía CDN si aplica -->
```

---

## 8. Checklist de Performance

- [ ] Componentes usan `OnPush` o Signals
- [ ] No hay funciones llamadas directamente en templates
- [ ] Suscripciones de RxJS tienen `takeUntilDestroyed` o se convierten a Promesas
- [ ] Listas > 100 items usan virtual scrolling
- [ ] No hay imports de librerías enteras (tree-shaking óptimo)
- [ ] Assets estáticos tienen dimensiones definidas y lazy loading si aplica
- [ ] `track` en `@for` usa identificador único, no índice

---

## Comandos de diagnóstico

```bash
# Ver tamaño del bundle de producción
npm run build 2>&1 | grep -E "(main|polyfills|styles)"

# Buscar uso de ngZone.run fuera de necesidad
grep -rn "ngZone.run" src/

# Detectar subscriptions sin takeUntil/takeUntilDestroyed
grep -rn "\.subscribe(" src/app --include="*.ts" | grep -v "spec.ts" | wc -l
```
