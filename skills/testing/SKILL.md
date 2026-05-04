---
name: testing
description: Escribe y corrige tests para el proyecto usando Vitest y Angular Testing Library. Úsalo cuando necesites crear tests unitarios para servicios, tests de componentes Angular standalone, o tests de integración. Conoce la configuración de Vitest del proyecto y los patrones de testing con HttpClient, formularios reactivos y RxJS.
---

Eres un experto en testing para Angular 21 con Vitest en el proyecto de sostenibilidad UNIBÁN.

## Configuración del proyecto

**Framework:** Vitest 4.0.8
**Config:** `vitest.config.ts` en la raíz (o en `angular.json`)
**Archivos de test:** `*.spec.ts` junto al archivo que testean
**Tipos globales:** `vitest/globals` (configurado en `tsconfig.spec.json`)
**Ejecutar tests:**
```bash
npm test           # modo watch
npm run test:ci    # una sola vez (verificar script en package.json)
```

---

## Tests de Servicios

### Servicio HTTP con HttpClient

```typescript
// hc-fertilizantes.service.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { HcFertilizantesService } from './hc-fertilizantes.service';

describe('HcFertilizantesService', () => {
  let service: HcFertilizantesService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        HcFertilizantesService,
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });
    service = TestBed.inject(HcFertilizantesService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('debe listar fertilizantes', async () => {
    const mockData = [{ id: 1, nombre: 'Urea' }];

    const promise = service.listar();
    const req = httpMock.expectOne('/api/hc/fertilizantes');
    expect(req.request.method).toBe('GET');
    req.flush(mockData);

    const result = await promise;
    expect(result).toEqual(mockData);
  });

  it('debe crear un fertilizante', async () => {
    const nuevo = { nombre: 'DAP' };
    const creado = { id: 2, nombre: 'DAP' };

    const promise = service.crear(nuevo);
    const req = httpMock.expectOne('/api/hc/fertilizantes');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(nuevo);
    req.flush(creado);

    const result = await promise;
    expect(result).toEqual(creado);
  });
});
```

---

## Tests de Componentes Standalone

### Componente con servicio inyectado

```typescript
// maestro-crud-page.component.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { MaestroCrudPageComponent } from './maestro-crud-page.component';
import { MiServicio } from '../../core/mi.service';

describe('MaestroCrudPageComponent', () => {
  let component: MaestroCrudPageComponent;
  let fixture: ComponentFixture<MaestroCrudPageComponent>;
  let serviceSpy: { listar: ReturnType<typeof vi.fn> };

  beforeEach(async () => {
    serviceSpy = { listar: vi.fn().mockResolvedValue([]) };

    await TestBed.configureTestingModule({
      imports: [MaestroCrudPageComponent],  // standalone: importar directamente
      providers: [
        { provide: MiServicio, useValue: serviceSpy },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(MaestroCrudPageComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debe crearse', () => {
    expect(component).toBeTruthy();
  });

  it('debe cargar datos al iniciar', async () => {
    serviceSpy.listar.mockResolvedValue([{ id: 1, nombre: 'Test' }]);
    await component.cargarDatos();
    expect(component.items.length).toBe(1);
  });
});
```

---

## Tests de Guards

```typescript
// permission.guard.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { Router } from '@angular/router';
import { PermissionGuard } from './permission.guard';
import { UnibanAuthService } from '../auth/uniban-auth.service';
import { PERMISOS } from '../constants/permisos.constants';

describe('PermissionGuard', () => {
  let guard: PermissionGuard;
  let authServiceMock: { currentUser: unknown };
  let routerMock: { navigate: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    authServiceMock = { currentUser: null };
    routerMock = { navigate: vi.fn() };

    TestBed.configureTestingModule({
      providers: [
        PermissionGuard,
        { provide: UnibanAuthService, useValue: authServiceMock },
        { provide: Router, useValue: routerMock },
      ],
    });
    guard = TestBed.inject(PermissionGuard);
  });

  it('debe denegar acceso si no tiene permiso', () => {
    authServiceMock.currentUser = { permisos: [] };
    const route = { data: { permission: PERMISOS.ADMINISTRACION } } as any;
    const result = guard.canActivate(route, {} as any);
    expect(result).toBe(false);
    expect(routerMock.navigate).toHaveBeenCalledWith(['/inicio']);
  });
});
```

---

## Tests de Helpers/Funciones puras

```typescript
// permission.helper.spec.ts
import { describe, it, expect } from 'vitest';
import { hasAnyPermission, isValidAction } from './permission.helper';
import { PERMISOS, ACCIONES_PERMISOS } from '../constants/permisos.constants';

describe('hasAnyPermission', () => {
  it('retorna true si el usuario tiene el permiso', () => {
    const user = { permisos: [PERMISOS.SALARIO_DIGNO] } as any;
    expect(hasAnyPermission(user, PERMISOS.SALARIO_DIGNO)).toBe(true);
  });

  it('retorna false si el usuario no tiene el permiso', () => {
    const user = { permisos: [] } as any;
    expect(hasAnyPermission(user, PERMISOS.SALARIO_DIGNO)).toBe(false);
  });

  it('retorna false si el usuario es null', () => {
    expect(hasAnyPermission(null, PERMISOS.SALARIO_DIGNO)).toBe(false);
  });
});
```

---

## Patrones clave para este proyecto

### Mock de UnibanAuthService
```typescript
const authServiceMock = {
  currentUser: { id: 1, nombre: 'Test User', permisos: [PERMISOS.SALARIO_DIGNO] },
  user$: of({ id: 1, nombre: 'Test User' }),
  isAuthenticated: true,
  logout: vi.fn(),
};
```

### Mock de Router
```typescript
const routerMock = {
  navigate: vi.fn(),
  navigateByUrl: vi.fn(),
};
```

### Esperar operaciones async en componentes
```typescript
it('debe cargar datos', async () => {
  fixture.detectChanges(); // ngOnInit
  await fixture.whenStable(); // esperar promesas
  fixture.detectChanges(); // actualizar DOM
  expect(component.items.length).toBeGreaterThan(0);
});
```

---

## Qué testear primero (prioridad)

1. **Helpers y funciones puras** — `permission.helper.ts`, `navigation.helper.ts`
2. **Guards** — `UnibanAuthGuard`, `PermissionGuard`
3. **Servicios de API** — listar, crear, actualizar, eliminar
4. **Lógica de componentes** — métodos públicos, no detalles de implementación

## Antes de escribir tests

1. Leer el archivo a testear completamente
2. Identificar dependencias inyectadas (para hacer mocks)
3. Revisar si hay tests similares en el proyecto como referencia
4. Ejecutar `npm test` para confirmar que el entorno funciona
