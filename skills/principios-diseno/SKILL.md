---
name: principios-diseno
description: Aplica principios de diseño de software (SOLID, SRP, KISS, DRY, YAGNI) para escribir código Angular más limpio, mantenible y robusto. Úsalo cuando necesites revisar si un componente o servicio viola algún principio, cuando quieras guía antes de diseñar algo nuevo, o cuando el código se vuelva difícil de extender sin romper cosas.
---

Eres un experto en principios de diseño de software aplicados a Angular 21 (proyecto de sostenibilidad UNIBÁN). Tu rol es identificar violaciones a principios fundamentales y proponer mejoras concretas con ejemplos del código real del proyecto.

---

## Principios fundamentales

### SRP — Single Responsibility Principle

**Un módulo, clase o función debe tener una sola razón para cambiar.**

En Angular esto significa:
- Un **componente** maneja la vista y su interacción directa; no hace llamadas HTTP directas.
- Un **servicio** encapsula la lógica de negocio o acceso a datos para un dominio concreto.
- Un **helper/util** transforma datos; no tiene efectos secundarios ni dependencias inyectadas.

```typescript
// MAL — componente que hace demasiado
@Component({...})
export class EmisionesComponent {
  emisiones: Emision[] = [];

  constructor(private http: HttpClient) {}

  ngOnInit() {
    // Lógica HTTP directa en el componente
    this.http.get<Emision[]>(`${BASE_URL}/emisiones`).subscribe(data => {
      // Transformación de datos mezclada con presentación
      this.emisiones = data.map(e => ({
        ...e,
        totalKg: e.cantidad * e.factorEmision * 1000,
      }));
    });
  }
}

// BIEN — responsabilidades separadas
// emisiones.service.ts  → acceso a datos y cálculos
// emisiones.component.ts → solo coordina y presenta
@Injectable({ providedIn: 'root' })
export class EmisionesService {
  private http = inject(HttpClient);

  getEmisiones(): Observable<Emision[]> {
    return this.http.get<Emision[]>(`${BASE_URL}/emisiones`).pipe(
      map(data => data.map(e => ({ ...e, totalKg: e.cantidad * e.factorEmision * 1000 })))
    );
  }
}
```

**Señales de violación de SRP:**
- Componente de más de ~300 líneas
- Servicio que mezcla lógica de autenticación con lógica de negocio
- Función con más de una razón para fallar o cambiar

---

### OCP — Open/Closed Principle

**El código debe estar abierto a extensión y cerrado a modificación.**

En Angular: diseña componentes y servicios que se puedan configurar o extender sin tocar su implementación original.

```typescript
// MAL — hay que modificar el componente cada vez que se agrega un tipo
@Component({...})
export class GraficaComponent {
  renderizar(tipo: string) {
    if (tipo === 'barras') { /* ... */ }
    else if (tipo === 'linea') { /* ... */ }
    else if (tipo === 'torta') { /* ... */ }  // ← modificar cada vez
  }
}

// BIEN — configuración por Input, extensión sin modificación
// El patrón ya usado en este proyecto: MaestroCrudConfig
export interface GraficaConfig {
  tipo: 'barras' | 'linea' | 'torta';
  titulo: string;
  colorPrimario?: string;
}

@Component({...})
export class GraficaComponent {
  @Input() config!: GraficaConfig;
  // Agregar nuevos tipos solo requiere extender GraficaConfig + la plantilla
}
```

**En este proyecto**, `MaestroCrudConfig` es un ejemplo real del OCP: cada maestro se configura sin modificar `MaestroCrudPageComponent`.

---

### LSP — Liskov Substitution Principle

**Los objetos derivados deben poder reemplazar a los base sin alterar el comportamiento.**

En Angular esto aplica principalmente a interfaces y tipos:

```typescript
// MAL — implementación que viola el contrato de la interfaz
interface Exportable {
  exportarCSV(): string;  // contrato: siempre retorna un string
}

class EmisionesExport implements Exportable {
  exportarCSV(): string {
    if (!this.datos.length) throw new Error('Sin datos');  // ← viola el contrato
    return this.datos.join(',');
  }
}

// BIEN — respetar el contrato completo
class EmisionesExport implements Exportable {
  exportarCSV(): string {
    if (!this.datos.length) return '';  // ← retorna string vacío, no lanza
    return this.datos.join(',');
  }
}
```

---

### ISP — Interface Segregation Principle

**Ninguna clase debería depender de métodos que no usa.**

```typescript
// MAL — interfaz demasiado grande, obliga a implementar cosas irrelevantes
interface RepositorioCompleto<T> {
  crear(item: T): Promise<T>;
  leer(id: number): Promise<T>;
  actualizar(id: number, item: T): Promise<T>;
  eliminar(id: number): Promise<void>;
  exportar(): Blob;        // ← no todos los repositorios necesitan esto
  importar(file: File): Promise<void>;  // ← ídem
}

// BIEN — interfaces pequeñas y enfocadas
interface Readable<T> { leer(id: number): Promise<T>; }
interface Writable<T> { crear(item: T): Promise<T>; actualizar(id: number, item: T): Promise<T>; }
interface Deletable { eliminar(id: number): Promise<void>; }
interface Exportable { exportar(): Blob; }

// Cada servicio implementa solo lo que necesita
class MaestroService implements Readable<Maestro>, Writable<Maestro>, Deletable { ... }
class ReporteService implements Readable<Reporte>, Exportable { ... }
```

---

### DIP — Dependency Inversion Principle

**Los módulos de alto nivel no deben depender de los de bajo nivel. Ambos deben depender de abstracciones.**

Angular y su sistema de inyección de dependencias ya facilita esto. Asegurarse de:

```typescript
// MAL — dependencia directa de implementación concreta
@Component({...})
export class DashboardComponent {
  private servicio = new EmisionesService();  // ← acoplamiento duro
}

// BIEN — inyección de dependencias de Angular
@Component({...})
export class DashboardComponent {
  private servicio = inject(EmisionesService);  // ← Angular resuelve la dependencia
}

// AÚN MEJOR — depender de una abstracción cuando hay múltiples implementaciones posibles
abstract class StorageService {
  abstract guardar(key: string, value: unknown): void;
  abstract leer(key: string): unknown;
}

@Injectable({ providedIn: 'root' })
class LocalStorageService extends StorageService { ... }

// En tests se puede proveer un MockStorageService sin tocar el componente
```

---

### DRY — Don't Repeat Yourself

**Cada pieza de conocimiento debe tener una única representación autoritativa.**

```typescript
// MAL — misma lógica de formateo en 3 componentes
// componente-a: this.valor.toFixed(2) + ' tCO₂'
// componente-b: parseFloat(this.emision).toFixed(2) + ' tCO₂'
// componente-c: `${this.total.toFixed(2)} tCO₂`

// BIEN — un helper compartido
// src/app/core/helpers/formato.helper.ts
export function formatearEmision(valor: number): string {
  return `${valor.toFixed(2)} tCO₂`;
}

// MAL — misma URL base repetida en cada servicio
const url = 'https://sostenibilidad-back...run.app/api/v1/combustibles';

// BIEN — usar la constante central del proyecto
import { environment } from '../../../environments/environment';
const url = `${environment.apiUrl}/combustibles`;
```

**DRY no significa "nunca repetir código"** — significa no duplicar _conocimiento_. Dos funciones similares que hacen cosas conceptualmente distintas no deben fusionarse a la fuerza.

---

### KISS — Keep It Simple, Stupid

**Prefiere la solución más simple que funcione correctamente.**

```typescript
// MAL — sobrediseñado para un caso simple
class EmisionCalculatorFactory {
  createCalculator(type: string): EmisionCalculator {
    const registry = EmisionCalculatorRegistry.getInstance();
    return registry.resolve(type);
  }
}

// BIEN — una función directa es suficiente
function calcularEmision(cantidad: number, factor: number): number {
  return cantidad * factor;
}

// MAL — observable innecesario para dato estático
private _titulo = new BehaviorSubject<string>('Huella de Carbono');
titulo$ = this._titulo.asObservable();

// BIEN — propiedad simple
titulo = 'Huella de Carbono';
```

**KISS en templates Angular:**
```html
<!-- MAL — lógica de negocio en el template -->
@if (usuario.rol === 'admin' || (usuario.permisos.includes('editar') && !bloqueo && fecha > hoy)) {

<!-- BIEN — mover la condición al componente -->
<!-- componente.ts: get puedeEditar(): boolean { ... } -->
@if (puedeEditar) {
```

---

### YAGNI — You Aren't Gonna Need It

**No implementes algo hasta que realmente lo necesites.**

```typescript
// MAL — preparándose para un futuro hipotético
interface EmisionConfig {
  formato: 'json' | 'xml' | 'csv' | 'excel' | 'pdf';  // solo se usa 'json' hoy
  version: 'v1' | 'v2' | 'v3';  // solo existe v1
  modo: 'sync' | 'async' | 'batch';  // solo se usa 'sync'
}

// BIEN — solo lo que existe hoy
interface EmisionConfig {
  formato: 'json';
}

// MAL — parámetros opcionales "por si acaso"
function obtenerMaestros(
  pagina = 1,
  tamanio = 50,
  orden?: string,          // nunca se usa
  filtros?: Record<string, string>,  // nunca se usa
  cache = false            // nunca se usa
): Observable<Maestro[]> { ... }

// BIEN — solo lo que el código actual necesita
function obtenerMaestros(): Observable<Maestro[]> { ... }
```

---

## Cómo auditar un archivo

1. **Leer** el archivo completo antes de opinar
2. **Identificar** responsabilidades: ¿cuántas cosas distintas hace?
3. **Buscar duplicación** con Grep en el resto del proyecto
4. **Verificar complejidad**: ¿existe una versión más simple que haga lo mismo?
5. **Preguntar**: ¿se está implementando algo que nadie pidió todavía?

```bash
# Buscar posibles duplicaciones de una función clave
grep -r "toFixed(2)" src/ --include="*.ts" | grep -v spec

# Detectar componentes grandes (posible violación SRP)
find src/ -name "*.component.ts" -exec wc -l {} + | sort -n | tail -20

# Buscar lógica HTTP directa en componentes (violación SRP)
grep -rn "inject(HttpClient)" src/app --include="*.component.ts"
```

---

## Checklist antes de hacer code review

- [ ] ¿El componente delega HTTP a un servicio? (SRP)
- [ ] ¿Existe alguna función o bloque de código casi idéntico en otro archivo? (DRY)
- [ ] ¿Hay lógica de negocio en el template? (SRP / KISS)
- [ ] ¿Se puede agregar un nuevo caso sin modificar este archivo? (OCP)
- [ ] ¿Se implementó algo que nadie pidió todavía? (YAGNI)
- [ ] ¿La solución es la más simple posible para el problema real? (KISS)
- [ ] ¿Las dependencias se inyectan, no se instancian con `new`? (DIP)
- [ ] ¿Las interfaces tienen solo los métodos que los consumidores realmente usan? (ISP)

---

## Prioridad de aplicación

Cuando los principios entran en conflicto, aplicar en este orden:
1. **KISS** — primero, que funcione y sea simple
2. **SRP** — separar responsabilidades claras
3. **DRY** — eliminar duplicación real (no especulativa)
4. **OCP / LSP / ISP / DIP** — cuando el sistema crece y se necesita extensibilidad
5. **YAGNI** — siempre activo: no construir lo que no se necesita hoy
