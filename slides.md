---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# some information about your slides (markdown enabled)
title: Transacciones y transacciones anidadas
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Transacciones y transacciones anidadas

---

# ¿Que es una transaccion?

Son operaciones que garantizan la integridad y consistencia de los datos.
<br/>
<br/>

- ⚛ **Atomicidad** - Se consideran una unidad atómica.
  <br/>
- ⚖ **Todo o nada** - La transacción se completa o no se realiza en absoluto.
  <br/>
- 🧠 **Integridad** - Los datos se mantienen íntegros incluso en caso de errores o fallos del sistema.

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/features/slide-scope-style
-->

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---

# ¿Cómo se escriben las transacciones en T-SQL?

En Transact-SQL (T-SQL), se utiliza la siguiente estructura básica para una transacción:

1. **BEGIN TRANSACTION**: Inicia una nueva transacción.
2. **COMMIT**: Confirma la transacción si todas las instrucciones se ejecutan correctamente.
3. **ROLLBACK**: Revierte la transacción si ocurre un error, deshaciendo todas las operaciones realizadas desde el inicio de la transacción.

A continuación, se presenta un ejemplo de código que utiliza transacciones para insertar datos en dos tablas:

---

```sql {all|1|3,14,15,21|4-13|16-20|all} twoslash
BEGIN TRANSACTION

    BEGIN TRY
        -- Inserta datos en la primera tabla
        INSERT INTO Tabla1 (columna1, columna2)
        VALUES (valor1, valor2);

        -- Inserta datos en la segunda tabla
        INSERT INTO Tabla2 (columna1, columna2)
        VALUES (valor3, valor4);

        -- Confirma la transacción si no hubo errores
        COMMIT;
    END TRY
    BEGIN CATCH
        -- Si ocurre un error, revierte todos los cambios
        ROLLBACK;

        -- Manejo de error, como registrar el mensaje de error
        PRINT 'Ocurrió un error. La transacción ha sido revertida.';
    END CATCH
```

---

# ¿Qué son las transacciones anidadas?

Son transacciones iniciadas dentro de otra transacción.

- En T-SQL, permiten dividir el trabajo en sub-operaciones dentro de una transacción principal, manteniendo control sobre los datos.

- Solo el primer <code>BEGIN TRANSACTION</code> permite hacer un <code>COMMIT</code> definitivo.

- Si ocurre un <code>ROLLBACK</code> en una transacción anidada, se revierte toda la transacción principal, asegurando la consistencia de los datos.

A continuación, se presenta un ejemplo de código que utiliza transacciones anidadas:

---

```sql {all|1|1-6,14-22|7-13|all} twoslash
BEGIN TRANSACTION

    BEGIN TRY
        -- Primera transacción
        INSERT INTO Tabla1 (columna1) VALUES ('Valor1');

        -- Transacción anidada
        BEGIN TRANSACTION
            INSERT INTO Tabla2 (columna2) VALUES ('Valor2');

            -- Commit de la transacción anidada
            COMMIT;
        END TRANSACTION

        -- Commit de la primera transacción
        COMMIT;
    END TRY
    BEGIN CATCH
        -- Si ocurre un error, se realiza un rollback total
        ROLLBACK;
        PRINT 'Error en la transacción, se ha revertido.';
    END CATCH;
```

---

# 1. Uso de `SCOPE_IDENTITY()`

<br/>

<code>SCOPE_IDENTITY()</code> es una función que devuelve el último valor de identidad generado en la misma sesión y ámbito de ejecución.
<br/>
<br/>

Es útil en transacciones cuando es necesario obtener el valor de una clave primaria recién insertada y utilizarlo en operaciones posteriores dentro de la misma transacción.
<br/>
<br/>

### A continuacion se dara un ejemplo de uso de <code>SCOPE_IDENTITY()</code> en una transacción

---

```sql {all|4,5|6|8,9|all} twoslash
BEGIN TRANSACTION

    BEGIN TRY
        -- Inserta un nuevo registro y obtiene el ID generado
        INSERT INTO Tabla1 (columna1) VALUES ('Valor');
        DECLARE @NuevoID INT = SCOPE_IDENTITY();

        -- Usa el ID generado en otra operación
        INSERT INTO Tabla2 (columna1, columna2) VALUES (@NuevoID, 'Otro Valor');

        -- Confirma la transacción si no hubo errores
        COMMIT;
    END TRY
    BEGIN CATCH
        -- Reversa la transacción en caso de error
        ROLLBACK;
        PRINT 'Error en la transacción, se ha revertido.';
    END CATCH;
```

---

# Caso de uso real, transaccion exitosa

```sql {all|4-7|8-12|12-15|17-19|21-25|all} twoslash
BEGIN TRY
    BEGIN TRANSACTION;

    -- Inserta un registro en la tabla Reparaciones
    INSERT INTO [dbo].[Reparaciones] (idPresupuesto, reparado, observaciones, fechaDeFinalizacion, irreparable)
    VALUES (1021, 1, 'Reparación completada', GETDATE(), 0);

    -- Inserta un registro en la tabla Entregas relacionado con la reparación
    INSERT INTO [dbo].[Entregas] (idReparacion, fecha, idMetodoPago)
    VALUES (1021, GETDATE(), 1);

    -- Actualiza el estado de baja del equipo relacionado
    UPDATE [dbo].[Equipos]
    SET baja = 'si'
    WHERE idEquipo = (SELECT idEquipo FROM [dbo].[Revisiones] WHERE idEquipo = (SELECT idRevision FROM [dbo].[Presupuestos] WHERE idPresupuesto = 1021));

    -- Confirma la transacción
    COMMIT;
    PRINT 'Transacción completada con éxito';
END TRY
BEGIN CATCH
    -- Realiza un rollback si hay un error
    ROLLBACK;
    PRINT 'Ocurrió un error. Se realizó un rollback de la transacción';
END CATCH;
```

---

# Transaccion erronea a proposito

```sql {all|8-10|21-28|all} twoslash
BEGIN TRY
    BEGIN TRANSACTION;

    -- Inserta un registro en la tabla Reparaciones
    INSERT INTO [dbo].[Reparaciones] (idPresupuesto, reparado, observaciones, fechaDeFinalizacion, irreparable)
    VALUES (1, 1, 'Reparaci�n con error', GETDATE(), 0);

    -- Provoca un error intencional (violaci�n de clave primaria en Entregas)
    INSERT INTO [dbo].[Entregas] (idReparacion, fecha, idMetodoPago)
    VALUES (9999, GETDATE(), 1);  -- idReparacion 9999 no existe en Reparaciones

    -- Actualiza el estado de baja del equipo relacionado
    UPDATE [dbo].[Equipos]
    SET baja = 'si'
    WHERE idEquipo = (SELECT idEquipo FROM [dbo].[Revisiones] WHERE idEquipo = (SELECT idEquipo FROM [dbo].[Presupuestos] WHERE idPresupuesto = 1));

    -- Confirma la transacci�n
    COMMIT;
    PRINT 'Transacci�n completada con �xito';
END TRY
BEGIN CATCH
    -- Realiza un rollback si hay un error
    ROLLBACK;
    PRINT 'Ocurri� un error. Se realiz� un rollback de la transacci�n';
END CATCH;
```
