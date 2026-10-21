# Sistema de Reservas de Salones — Eventos Premier S.A.S.

Base de datos relacional para administrar salones, clientes, reservas, pagos y auditoría. Proyecto desarrollado para **MySQL 8.0+** y almacenamiento InnoDB.

## Contenido

| Archivo | Propósito |
|---|---|
| `01_schema_ddl.sql` | Crea la base de datos, tablas, restricciones, índices y relaciones. |
| `02_functions.sql` | Calcula totales con IVA y verifica disponibilidad por rango. |
| `03_triggers.sql` | Actualiza estado del salón y registra auditorías. |
| `04_views_queries.sql` | Vista resumen y consultas de negocio. |
| `05_sample_data.sql` | Inserta datos demostrativos y muestra llamadas de ejemplo. |
| `06_procedures.sql` | Procedimiento adicional para crear reservas con validación de solapamientos. |

## Requisitos

- MySQL Server 8.0 o superior.
- MySQL Workbench o cliente de consola.
- Un usuario con permisos para crear bases de datos, tablas, funciones, triggers y procedimientos.

## Instalación con MySQL Workbench

1. Abre MySQL Workbench y conéctate a tu servidor local.
2. Abre `01_schema_ddl.sql` y ejecútalo completo.
3. Ejecuta en orden `02_functions.sql`, `03_triggers.sql`, `04_views_queries.sql`.
4. Ejecuta `06_procedures.sql` para habilitar la creación validada de reservas.
5. Ejecuta `05_sample_data.sql` para cargar datos de demostración.
6. Actualiza el panel **Schemas** y selecciona `eventos_premier_db`.

Ejecuta los archivos completos para que Workbench procese correctamente los bloques `DELIMITER`.

## Instalación desde consola

Desde la carpeta donde descargaste los scripts:

```bash
mysql -u root -p < 01_schema_ddl.sql
mysql -u root -p < 02_functions.sql
mysql -u root -p < 03_triggers.sql
mysql -u root -p < 04_views_queries.sql
mysql -u root -p < 06_procedures.sql
mysql -u root -p < 05_sample_data.sql
```

Introduce la contraseña cuando se solicite. Si el usuario no es `root`, reemplázalo por tu usuario de MySQL.

## Ejemplos prácticos

### Calcular una reserva con IVA

```sql
USE eventos_premier_db;
SELECT calcular_total_reserva(160000.00, 3.00) AS total;
-- Resultado esperado: 571200.00
```

### Verificar disponibilidad

```sql
SELECT verificar_disponibilidad(
  1, '2026-12-10 09:00:00', '2026-12-10 12:00:00'
) AS disponible;
-- 1 = libre; 0 = no disponible, rango inválido o salón inexistente.
```

### Crear una reserva con validación

```sql
CALL crear_reserva(
  1,
  1,
  '2026-12-10 09:00:00',
  '2026-12-10 12:00:00',
  'Reunión empresarial de demostración'
);
```

El procedimiento calcula las horas y el valor con IVA, verifica que el cliente exista y rechaza cruces con otras reservas confirmadas. Si la operación falla, revierte la transacción y devuelve el error.

### Registrar un pago

```sql
INSERT INTO pagos (reserva_id, monto_pagado, metodo_pago, referencia)
VALUES (1, 200000.00, 'Transferencia', 'TRX-DEMO-001');
```

El monto debe ser mayor que cero y la reserva debe existir.

### Consultar la vista

```sql
SELECT * FROM vista_resumen_reservas
ORDER BY fecha_hora_inicio DESC;
```

### Cambiar un precio y revisar auditoría

```sql
UPDATE salones
SET precio_hora = 175000.00
WHERE id = 2;

SELECT *
FROM auditoria_precios
WHERE salon_id = 2
ORDER BY fecha_cambio DESC;
```

El trigger registra usuario, fecha, precio anterior y precio nuevo cuando cambia el precio por hora.

### Cancelar una reserva

```sql
UPDATE reservas
SET estado_reserva = 'Cancelada'
WHERE id = 1;
```

Los reportes que cuentan reservas válidas excluyen las canceladas. La tabla `auditoria_reservas` conserva el cambio de estado.

## Reglas y decisiones de diseño

- **Dinero:** `DECIMAL` evita errores de precisión típicos de tipos aproximados como `FLOAT`.
- **Integridad:** claves foráneas impiden borrar salones/clientes con reservas existentes. Los pagos se eliminan en cascada al borrar su reserva, según el requerimiento.
- **Solapamiento:** se consideran cruzados los intervalos cuando `inicio_existente < fin_nuevo` y `fin_existente > inicio_nuevo`. Una reserva que termina justo cuando empieza otra no se considera solapada.
- **Estados:** `estado_reserva` distingue reservas confirmadas, canceladas y finalizadas.
- **Índices:** apoyan búsquedas por estado, capacidad, cliente, salón y fechas.
- **Auditoría:** los triggers almacenan cambios de precio y cambios de estado de reserva.

## Nota importante sobre el estado del salón

El trigger solicitado cambia el estado del salón a `Ocupado` al insertar cualquier reserva y lo libera al eliminar la última reserva confirmada. Esto cumple el requerimiento académico, pero en un sistema real una reserva futura no significa que el salón esté ocupado en este instante. En producción conviene tratar `estado` como estado operativo (por ejemplo, mantenimiento) y calcular la disponibilidad por fecha y hora desde `reservas`. Además, las reservas deben crearse mediante `crear_reserva` para centralizar la validación de solapamientos.

## Consultas de análisis incluidas

`04_views_queries.sql` contiene:
1. Reservas iniciadas dentro de un rango usando `BETWEEN`.
2. Salones con capacidad superior a 50 y estado disponible.
3. Clientes corporativos con más de tres reservas no canceladas.
4. Saldo pendiente por reserva.
5. Ingresos contratados por salón.
6. Historial de variaciones de precios.

## Buenas prácticas

- No guardar contraseñas ni datos sensibles innecesarios en scripts o repositorios.
- Usar un usuario MySQL con permisos mínimos necesarios en entornos reales.
- Respaldar la base de datos antes de cambios de esquema.
- Probar funciones, triggers y procedimientos primero en una base de datos de desarrollo.
- Versionar scripts y documentar cambios en Git.
- Para despliegues reales, definir políticas de cancelación, reembolsos, zona horaria, facturación y retención de auditoría.

## Autoría

**Autor:** Nestor Santiago Cobos Manrique  
**Proyecto:** Sistema de Reservas de Salones — Eventos Premier S.A.S.  
**Tecnología:** MySQL 8.0+, SQL, InnoDB.
