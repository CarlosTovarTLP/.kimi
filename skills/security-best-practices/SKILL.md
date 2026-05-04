---
name: security-best-practices
description: Aplica buenas prácticas de seguridad en el código Angular del proyecto. Úsalo cuando necesites revisar o corregir vulnerabilidades, sanitizar entradas de usuario, proteger datos sensibles, asegurar llamadas HTTP, manejar secrets correctamente, o auditar un componente/servicio antes de llevarlo a producción. Cubre OWASP Top 10 aplicado a Angular 21.
---

Eres un experto en seguridad para aplicaciones Angular 21 del proyecto de sostenibilidad UNIBÁN.

## Áreas de seguridad cubiertas

### 1. XSS (Cross-Site Scripting)

**Angular escapa automáticamente** el contenido en templates `{{ }}` y `[attr]`. Sin embargo:

```typescript
// PELIGROSO — nunca usar innerHTML con datos del usuario
<div [innerHTML]="userInput"></div>

// SEGURO — usar DomSanitizer si es imprescindible
import { DomSanitizer } from '@angular/platform-browser';

@Component({...})
export class MiComp {
  private sanitizer = inject(DomSanitizer);

  safeHtml(value: string) {
    return this.sanitizer.sanitize(SecurityContext.HTML, value);
  }
}
```

**Nunca usar:**
- `bypassSecurityTrustHtml()` con contenido del usuario
- `bypassSecurityTrustScript()`
- `eval()` o `Function()` con datos externos

---

### 2. Protección de secrets y variables de entorno

**Reglas absolutas:**
- **Nunca** poner API keys, passwords ni tokens en el código fuente
- **Nunca** commitear `environment.prod.ts` con credenciales reales
- Las variables de entorno deben gestionarse en el pipeline CI/CD o en el servidor

```typescript
// CORRECTO — usar environment para configuración, no para secrets
export const environment = {
  production: true,
  apiBaseUrl: 'https://uniban-backend...run.app',  // URL pública ✓
  // NO poner: apiKey, dbPassword, privateToken     // ✗
};
```

**Verificar `.gitignore`** incluya:
```
environment.prod.ts  # si contiene datos sensibles
.env
*.key
```

---

### 3. Almacenamiento seguro en cliente

El proyecto usa `localStorage` para JWT y datos de usuario. Consideraciones:

```typescript
// UnibanAuthService — revisar estas prácticas
private JWT_KEY = 'uniban_jwt';

// Verificar siempre antes de usar el token
getToken(): string | null {
  const token = localStorage.getItem(this.JWT_KEY);
  if (!token || this.isJwtExpired()) return null;
  return token;
}
```

**Reglas:**
- Nunca guardar passwords en `localStorage`
- Verificar expiración del JWT antes de cada uso (ya implementado en `UnibanAuthGuard`)
- En datos muy sensibles, preferir `sessionStorage` sobre `localStorage`
- No guardar información PII (datos personales) en storage del cliente sin necesidad

---

### 4. Seguridad en llamadas HTTP

El interceptor `UnibanAuthInterceptor` ya añade el token. Verificar adicionalmente:

```typescript
// Nunca construir URLs con concatenación directa de inputs del usuario
// PELIGROSO
const url = `${this.baseUrl}/items/${userInput}`;

// SEGURO — usar HttpParams
const params = new HttpParams().set('id', userId.toString());
this.http.get(this.baseUrl, { params });

// Validar tipos numéricos antes de usarlos en URLs
async obtener(id: number): Promise<Item> {
  if (!Number.isInteger(id) || id <= 0) throw new Error('ID inválido');
  return firstValueFrom(this.http.get<Item>(`${this.baseUrl}/${id}`));
}
```

**Headers de seguridad:** verificar que el backend retorne:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Strict-Transport-Security`

---

### 5. Validación de formularios

```typescript
// Siempre validar en el cliente Y en el backend
form = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  cantidad: [null, [Validators.required, Validators.min(0), Validators.max(999999)]],
  nombre: ['', [Validators.required, Validators.maxLength(200)]],
});

// Sanitizar strings antes de enviar si provienen de inputs libres
function sanitizeString(value: string): string {
  return value.trim().slice(0, 500); // limitar longitud
}
```

**Nunca confiar** solo en validación del lado cliente — el backend debe revalidar todo.

---

### 6. Control de acceso en el frontend

```typescript
// CORRECTO — usar PermissionGuard en rutas sensibles
{
  path: 'admin',
  component: AdminComponent,
  data: { permission: PERMISOS.ADMINISTRACION },
  canActivate: [PermissionGuard],  // ← obligatorio en rutas protegidas
}

// En templates — ocultar, no solo deshabilitar elementos sensibles
@if (puedeEditar) {
  <button class="btn-primary">Editar</button>
}
// No usar: <button [disabled]="!puedeEditar"> para acciones críticas
```

---

### 7. Manejo seguro de errores

```typescript
// PELIGROSO — exponer detalles internos al usuario
catch (error) {
  console.error('Error en DB:', error); // puede revelar estructura interna
  this.mensaje = JSON.stringify(error); // ✗ nunca mostrar al usuario
}

// SEGURO — logging interno, mensaje genérico al usuario
catch (error) {
  console.error('[MiServicio] Error al guardar:', error); // solo en consola dev
  Swal.fire({ title: 'Error', text: 'No se pudo guardar. Intenta de nuevo.', icon: 'error' });
}
```

---

### 8. Dependencias y auditoría

```bash
# Verificar vulnerabilidades en dependencias
npm audit

# Ver solo críticas y altas
npm audit --audit-level=high

# Actualizar dependencia vulnerable específica
npm update nombre-paquete
```

---

## Checklist de seguridad antes de producción

- [ ] Sin `[innerHTML]` con datos de usuario sin sanitizar
- [ ] Sin secrets en archivos de código o environment
- [ ] JWT validado antes de cada uso
- [ ] Todas las rutas sensibles con `PermissionGuard`
- [ ] Formularios con validadores de longitud máxima
- [ ] Errores no exponen detalles técnicos al usuario
- [ ] `npm audit` sin vulnerabilidades críticas/altas
- [ ] `.gitignore` incluye archivos con credenciales

## Antes de auditar

1. Leer `uniban-auth-interceptor.ts` — verificar manejo del token
2. Leer `uniban-auth.guard.ts` y `permission.guard.ts` — verificar lógica de acceso
3. Buscar usos de `[innerHTML]`, `bypassSecurityTrust*`, `localStorage`
4. Revisar `environment.ts` y `environment.prod.ts` — verificar que no haya secrets
5. Ejecutar `npm audit` para ver el estado de dependencias
