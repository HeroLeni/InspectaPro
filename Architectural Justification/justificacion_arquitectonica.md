# Justificación Arquitectónica del Sistema de Inspecciones
## Arquitectura Híbrida: SQL + NoSQL

Esta arquitectura usa **dos bases de datos** trabajando juntas. Cada una hace lo que mejor sabe hacer.

---

## Respuestas a las 7 Preguntas Clave

### 1. ¿Qué datos viven en SQL y por qué?

**En SQL (PostgreSQL):**
- **Subscription** (suscripciones y planes)
- **Company** (empresas clientes)
- **User** (usuarios del sistema)
- **Role** (roles: Admin, Técnico)
- **user_company** (relación muchos a muchos)
- **inspection_type** (catálogo de tipos de inspección)
- **inspection** (metadata: id, estado, fechas, asignaciones)

**Por qué SQL:**
- Necesitamos **relaciones estrictas** con Foreign Keys (un usuario pertenece a una empresa)
- Requiere **validaciones transaccionales ACID** (no exceder límites de suscripción)
- Los **estados** deben cambiar de forma controlada (DRAFT → ASSIGNED → IN_PROGRESS)
- **Reportes y métricas** son rápidos con JOINs e índices
- **Auditoría temporal** clara (quién cambió qué y cuándo)

### 2. ¿Qué datos viven en NoSQL y por qué?

**En NoSQL (MongoDB):**
- **Formularios de inspección** (secciones, campos, valores)
- **Fotos y archivos adjuntos** (metadata de archivos)
- **Notas del técnico** (comentarios, observaciones)
- **Logs de actividad** (timeline de acciones)
- **Datos de sincronización offline**

**Por qué NoSQL:**
- **Esquema flexible**: Cada tipo de inspección tiene campos diferentes (eléctrica vs. HVAC vs. piscina)
- **Estructura anidada**: Fotos con metadata, defectos con sub-items, checklists complejos
- **Trabajo offline**: El técnico sincroniza el documento completo cuando recupera conexión
- **Sin migraciones**: Agregar nuevos campos no requiere cambiar esquema de base de datos
- **Velocidad de escritura**: Miles de pequeñas actualizaciones (agregar foto, modificar nota)

### 3. ¿Qué riesgos existirían en un modelo completamente relacional (solo SQL)?

**Riesgo 1 - Explosión de tablas:**
- Para 20 tipos de inspección diferentes necesitas 20+ tablas de campos
- Cada cambio en un tipo requiere migración de esquema
- Código duplicado y difícil de mantener

**Riesgo 2 - Rendimiento en datos anidados:**
- Para mostrar una foto con metadata: 5-6 JOINs
- Queries lentos para estructuras complejas
- Índices complicados y costosos

**Riesgo 3 - Trabajo offline imposible:**
- Los técnicos no pueden trabajar sin conexión
- SQLite local + sincronización es muy compleja
- Riesgo de corrupción de Foreign Keys al sincronizar

**Riesgo 4 - Rigidez del esquema:**
- Agregar un campo nuevo = migración de base de datos
- No puedes tener campos condicionales (solo para cierto tipo de inspección)
- Versionado de formularios muy difícil

### 4. ¿Qué riesgos existirían en un modelo completamente documental (solo NoSQL)?

**Riesgo 1 - Sin integridad referencial:**
- Podrías asignar una inspección a un usuario que no existe
- Empresas eliminadas seguirían referenciadas
- No hay Foreign Keys que validen automáticamente

**Riesgo 2 - Límites de suscripción no confiables:**
- Race condition: dos técnicos crean inspección simultáneamente
- La empresa excede su límite de 100 inspecciones
- Problema de facturación y pérdida de ingresos

**Riesgo 3 - Duplicación de datos:**
- Cada inspección guarda datos completos del técnico y empresa
- Si Juan cambia su email, hay que actualizar 500 documentos
- Inconsistencias inevitables

**Riesgo 4 - Reportes lentos:**
- Agregaciones complejas (total por mes, por región, por estado)
- Lookups (JOINs simulados) son 10x más lentos que SQL
- No hay índices compuestos eficientes para reportes

**Riesgo 5 - Validaciones de negocio débiles:**
- Cambio de estado sin validar transición válida
- No puedes hacer "todo o nada" en múltiples documentos fácilmente
- Estados inconsistentes entre documentos relacionados

### 5. ¿Cómo están conectados ambos modelos?

**Estrategia: Reference Pattern (mismo ID)**

**SQL tiene la metadata:**
```sql
inspection (id: "abc123", company_id, status: "SUBMITTED", assigned_user_id, created_at)
```

**MongoDB tiene el contenido:**
```json
{
  "_id": "abc123",  // ← Mismo ID que SQL
  "sections": [...formulario...],
  "status": "SUBMITTED"  // ← Copia para búsquedas rápidas
}
```

**Flujo de trabajo:**

1. **Crear inspección:**
   - SQL crea registro (con validaciones)
   - MongoDB crea documento vacío con mismo ID
   - Si falla MongoDB → rollback en SQL

2. **Técnico llena formulario:**
   - Actualiza directo en MongoDB
   - SQL no se toca (no es necesario)

3. **Cambiar estado:**
   - SQL actualiza (validando transición)
   - MongoDB copia el nuevo estado
   - SQL siempre es la "verdad oficial"

4. **Mostrar inspección:**
   - Buscar en SQL y MongoDB en paralelo
   - Combinar resultados en la API
   - Retornar al usuario

### 6. Consideraciones sobre la consistencia y la trazabilidad

**Niveles de Consistencia:**

| Dato | Tipo | ¿Por qué? |
|------|------|-----------|
| Estados de inspección | **Fuerte (SQL)** | Crítico para el workflow, debe ser exacto |
| Límites de suscripción | **Fuerte (SQL)** | Afecta facturación, cero tolerancia a errores |
| Asignaciones usuario-empresa | **Fuerte (SQL)** | Seguridad y permisos |
| Datos de formulario | **Eventual (MongoDB)** | Puede tardar segundos en sincronizar, no es crítico |
| Logs de actividad | **Eventual (MongoDB)** | Histórico, no afecta operaciones en tiempo real |

**Trazabilidad (Auditoría):**

- **SQL:** Registra cambios de estado importantes
  - Quién cambió de DRAFT a ASSIGNED
  - Cuándo se aprobó o rechazó
  - Tabla `historial_estados` con timestamps

- **MongoDB:** Registra todas las acciones del técnico
  - "14:23 - Agregó 3 fotos"
  - "14:30 - Modificó campo de voltaje"
  - "14:35 - Agregó nota sobre corrosión"

**Sincronización y Reconciliación:**
- Cada hora un job verifica que SQL y MongoDB coincidan
- Si hay diferencia, SQL gana (es la verdad oficial)
- MongoDB se actualiza automáticamente
- Alertas al equipo técnico si hay muchas inconsistencias

### 7. ¿Cómo el modelo de negocio influye en las decisiones de diseño de datos?

**Factor 1: SaaS Multi-Tenant (múltiples empresas)**
- **Decisión:** Todas las empresas en la misma BD, separadas por `company_id`
- **Impacto:** Cada query filtra por empresa, seguridad a nivel de base de datos
- **Razón:** Más económico que BD separada por cliente, fácil de escalar

**Factor 2: Pricing por Límites (max_inspections)**
- **Decisión:** Validar límites en SQL con transacciones
- **Impacto:** Imposible exceder el plan contratado
- **Razón:** Si usas MongoDB, race conditions permitirían exceder límites = pérdida de ingresos

**Factor 3: Técnicos en Campo (trabajo sin conexión)**
- **Decisión:** MongoDB como almacén principal de formularios
- **Impacto:** App funciona 100% offline, sincroniza al conectar
- **Razón:** Zonas rurales/remotas sin cobertura, inspecciones no pueden detenerse

**Factor 4: Tipos de Inspección Variables**
- **Decisión:** Esquema flexible en MongoDB
- **Impacto:** Agregar nuevo tipo de inspección = configuración, no código
- **Razón:** Negocio necesita adaptarse rápido a nuevos mercados/regulaciones

**Factor 5: Cumplimiento Legal (ISO 9001, auditorías)**
- **Decisión:** Doble auditoría (SQL + MongoDB)
- **Impacto:** Registro completo de quién hizo qué y cuándo
- **Razón:** Certificaciones requieren trazabilidad completa, exportación diaria a archivo

**Factor 6: Escalabilidad del Negocio**
- **Decisión:** SQL para metadata, MongoDB para contenido
- **Impacto:** SQL escala verticalmente (servidor más potente), MongoDB horizontalmente (más servidores)
- **Razón:** 1000 empresas = poca metadata, 100,000 inspecciones = mucho contenido

---

## Diagrama Simple de la Arquitectura

```
[App Móvil] → [API]
                |
       ┌────────┴────────┐
       ↓                 ↓
  [PostgreSQL]      [MongoDB]
  - Usuarios        - Formularios
  - Empresas        - Fotos
  - Suscripciones   - Notas
  - Estados         - Logs de actividad
  - Relaciones      - Datos anidados
```

## Resumen Final

### Lo que debes recordar:

**SQL = Control y Validación**
- Relaciones entre datos (usuarios, empresas, suscripciones)
- Validaciones críticas de negocio (límites, permisos)
- Estados controlados (workflow de aprobación)
- Reportes y métricas rápidas

**MongoDB = Flexibilidad y Velocidad**
- Formularios con estructura variable
- Trabajo offline de técnicos
- Datos anidados complejos
- Cambios de esquema sin migraciones

**Cómo trabajan juntos:**
- Mismo ID en ambas bases de datos
- SQL es la "verdad oficial" para estados críticos
- MongoDB se sincroniza con SQL
- Si hay conflicto, SQL siempre gana

**Por qué esta arquitectura:**
1. Técnicos pueden trabajar sin internet ✅
2. Validaciones de negocio son 100% confiables ✅
3. Agregar nuevos tipos de inspección es fácil ✅
4. Reportes administrativos son rápidos ✅
5. Sistema puede crecer sin problemas ✅

No estamos usando dos bases de datos por complicarnos la vida. Cada una resuelve problemas específicos que la otra no puede resolver bien. Esta es la clave de una buena arquitectura híbrida.
