# Guía Completa de Sintaxis MySQL

## 1. CONSULTAS (SELECT)

### Orden de cláusulas:
```sql
SELECT [DISTINCT] columnas
FROM tabla1
[JOIN tabla2 ON condición]
[WHERE condición]
[GROUP BY columnas]
[HAVING condición]
[ORDER BY columnas [ASC|DESC]]
[LIMIT número [OFFSET número]];
```

### Ejemplos:
```sql
-- Básica
SELECT nombre, email FROM usuarios;

-- Con WHERE
SELECT * FROM libros WHERE precio > 5000;

-- Con JOINs
SELECT u.nombre, l.titulo 
FROM usuarios u 
JOIN prestamos p ON u.id_usuario = p.id_usuario
JOIN libros l ON p.id_libro = l.id_libro;

-- Con GROUP BY y HAVING
SELECT genero, COUNT(*) as total
FROM libros l JOIN generos g ON l.id_genero = g.id_genero
GROUP BY genero
HAVING COUNT(*) > 3;

-- Con subconsultas
SELECT * FROM usuarios 
WHERE id_usuario IN (SELECT DISTINCT id_usuario FROM prestamos);

-- Con funciones de ventana
SELECT nombre, precio, 
       ROW_NUMBER() OVER (ORDER BY precio DESC) as ranking
FROM libros;
```

## 2. INSERTS

### Sintaxis básica:
```sql
INSERT INTO tabla (columna1, columna2, ...) 
VALUES (valor1, valor2, ...);

-- Múltiples filas
INSERT INTO tabla (columna1, columna2) VALUES
(valor1a, valor2a),
(valor1b, valor2b),
(valor1c, valor2c);

-- Desde otra tabla
INSERT INTO tabla1 (columna1, columna2)
SELECT columna1, columna2 FROM tabla2 WHERE condición;

-- Con ON DUPLICATE KEY
INSERT INTO tabla (id, nombre) VALUES (1, 'Juan')
ON DUPLICATE KEY UPDATE nombre = VALUES(nombre);
```

### Ejemplos:
```sql
-- Básico
INSERT INTO usuarios (nombre, email) VALUES ('Juan', 'juan@email.com');

-- Con funciones
INSERT INTO prestamos (fecha_prestamo, fecha_vencimiento, id_usuario, id_libro)
VALUES (CURDATE(), DATE_ADD(CURDATE(), INTERVAL 7 DAY), 1, 1);

-- Condicional
INSERT IGNORE INTO generos (nombre) VALUES ('Ficción');
```

## 3. UPDATES

### Sintaxis:
```sql
UPDATE tabla 
SET columna1 = valor1, columna2 = valor2
[WHERE condición]
[ORDER BY columnas]
[LIMIT número];

-- Con JOINs
UPDATE tabla1 t1
JOIN tabla2 t2 ON t1.id = t2.id
SET t1.columna = valor
WHERE condición;
```

### Ejemplos:
```sql
-- Básico
UPDATE usuarios SET telefono = '123456789' WHERE id_usuario = 1;

-- Con cálculos
UPDATE libros SET precio = precio * 1.1 WHERE fecha_publicacion < '2020-01-01';

-- Con subconsulta
UPDATE usuarios SET tipo_membresia = 'premium'
WHERE id_usuario IN (
    SELECT id_usuario FROM prestamos 
    GROUP BY id_usuario 
    HAVING COUNT(*) > 5
);

-- Con JOIN
UPDATE libros l
JOIN prestamos p ON l.id_libro = p.id_libro
SET l.stock_disponible = l.stock_disponible - 1
WHERE p.estado = 'activo';

-- Condicional
UPDATE prestamos 
SET estado = CASE 
    WHEN fecha_vencimiento < CURDATE() THEN 'vencido'
    WHEN fecha_devolucion IS NOT NULL THEN 'devuelto'
    ELSE 'activo'
END;
```

## 4. DELETES

### Sintaxis:
```sql
DELETE FROM tabla 
[WHERE condición]
[ORDER BY columnas]
[LIMIT número];

-- Con JOINs
DELETE t1 FROM tabla1 t1
JOIN tabla2 t2 ON t1.id = t2.id
WHERE condición;

-- Múltiples tablas
DELETE t1, t2 FROM tabla1 t1
JOIN tabla2 t2 ON t1.id = t2.id
WHERE condición;
```

### Ejemplos:
```sql
-- Básico
DELETE FROM multas WHERE pagada = TRUE AND fecha_multa < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);

-- Con subconsulta
DELETE FROM usuarios 
WHERE id_usuario NOT IN (SELECT DISTINCT id_usuario FROM prestamos);

-- Con JOIN
DELETE p FROM prestamos p
JOIN usuarios u ON p.id_usuario = u.id_usuario
WHERE u.activo = FALSE;
```

## 5. FUNCIONES

### Sintaxis:
```sql
DELIMITER //
CREATE FUNCTION nombre_funcion(parámetros)
RETURNS tipo_dato
[DETERMINISTIC | NOT DETERMINISTIC]
[READS SQL DATA | MODIFIES SQL DATA | NO SQL]
BEGIN
    DECLARE variables;
    -- Lógica
    RETURN valor;
END //
DELIMITER ;
```

### Ejemplos:
```sql
DELIMITER //
CREATE FUNCTION CalcularDiasRetraso(fecha_vencimiento DATE)
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE dias INT DEFAULT 0;
    
    IF fecha_vencimiento < CURDATE() THEN
        SET dias = DATEDIFF(CURDATE(), fecha_vencimiento);
    END IF;
    
    RETURN dias;
END //
DELIMITER ;

-- Función más compleja
CREATE FUNCTION ObtenerMultaTotal(usuario_id BIGINT)
RETURNS DECIMAL(8,2)
READS SQL DATA
BEGIN
    DECLARE total DECIMAL(8,2) DEFAULT 0.00;
    
    SELECT COALESCE(SUM(monto), 0.00) INTO total
    FROM multas m
    JOIN prestamos p ON m.id_prestamo = p.id_prestamo
    WHERE p.id_usuario = usuario_id AND m.pagada = FALSE;
    
    RETURN total;
END //
DELIMITER ;

-- Uso
SELECT nombre, ObtenerMultaTotal(id_usuario) as multa_pendiente FROM usuarios;
```

## 6. PROCEDIMIENTOS ALMACENADOS

### Sintaxis:
```sql
DELIMITER //
CREATE PROCEDURE nombre_procedimiento(
    [IN | OUT | INOUT] parámetro tipo_dato,
    ...
)
BEGIN
    DECLARE variables;
    -- Lógica
    -- Manejo de errores
    -- Transacciones
END //
DELIMITER ;
```

### Ejemplos:
```sql
DELIMITER //
CREATE PROCEDURE RegistrarPrestamo(
    IN p_usuario_id BIGINT,
    IN p_libro_id BIGINT,
    OUT p_resultado VARCHAR(100)
)
BEGIN
    DECLARE stock_actual INT DEFAULT 0;
    DECLARE tipo_usuario VARCHAR(20);
    DECLARE dias_prestamo INT DEFAULT 7;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_resultado = 'Error en la transacción';
    END;
    
    START TRANSACTION;
    
    -- Verificar stock
    SELECT stock_disponible INTO stock_actual 
    FROM libros WHERE id_libro = p_libro_id;
    
    IF stock_actual <= 0 THEN
        SET p_resultado = 'Libro no disponible';
        ROLLBACK;
    ELSE
        -- Obtener tipo de membresía
        SELECT tipo_membresia INTO tipo_usuario 
        FROM usuarios WHERE id_usuario = p_usuario_id;
        
        -- Calcular días según membresía
        CASE tipo_usuario
            WHEN 'premium' THEN SET dias_prestamo = 14;
            WHEN 'vip' THEN SET dias_prestamo = 21;
            ELSE SET dias_prestamo = 7;
        END CASE;
        
        -- Insertar préstamo
        INSERT INTO prestamos (id_usuario, id_libro, fecha_prestamo, fecha_vencimiento)
        VALUES (p_usuario_id, p_libro_id, CURDATE(), DATE_ADD(CURDATE(), INTERVAL dias_prestamo DAY));
        
        -- Actualizar stock
        UPDATE libros SET stock_disponible = stock_disponible - 1 
        WHERE id_libro = p_libro_id;
        
        COMMIT;
        SET p_resultado = 'Préstamo registrado exitosamente';
    END IF;
END //
DELIMITER ;

-- Uso
CALL RegistrarPrestamo(1, 1, @resultado);
SELECT @resultado;
```

## 7. TRIGGERS

### Sintaxis:
```sql
DELIMITER //
CREATE TRIGGER nombre_trigger
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON tabla
FOR EACH ROW
BEGIN
    -- NEW.columna (para INSERT/UPDATE)
    -- OLD.columna (para UPDATE/DELETE)
    -- Lógica del trigger
END //
DELIMITER ;
```

### Ejemplos:
```sql
-- Trigger BEFORE INSERT
DELIMITER //
CREATE TRIGGER ValidarStockAntesPrestamo
BEFORE INSERT ON prestamos
FOR EACH ROW
BEGIN
    DECLARE stock_actual INT;
    
    SELECT stock_disponible INTO stock_actual 
    FROM libros WHERE id_libro = NEW.id_libro;
    
    IF stock_actual <= 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuficiente';
    END IF;
END //
DELIMITER ;

-- Trigger AFTER INSERT
CREATE TRIGGER ActualizarStockDespuesPrestamo
AFTER INSERT ON prestamos
FOR EACH ROW
BEGIN
    UPDATE libros 
    SET stock_disponible = stock_disponible - 1 
    WHERE id_libro = NEW.id_libro;
END //
DELIMITER ;

-- Trigger AFTER UPDATE
CREATE TRIGGER AuditoriaLibros
AFTER UPDATE ON libros
FOR EACH ROW
BEGIN
    INSERT INTO auditoria_libros (id_libro, accion, usuario_sistema, datos_anteriores, datos_nuevos)
    VALUES (
        NEW.id_libro, 
        'UPDATE', 
        USER(), 
        JSON_OBJECT('titulo', OLD.titulo, 'precio', OLD.precio),
        JSON_OBJECT('titulo', NEW.titulo, 'precio', NEW.precio)
    );
END //
DELIMITER ;
```

## 8. EVENTOS

### Sintaxis:
```sql
CREATE EVENT nombre_evento
ON SCHEDULE {AT timestamp | EVERY intervalo [STARTS timestamp] [ENDS timestamp]}
[ON COMPLETION [NOT] PRESERVE]
[ENABLE | DISABLE]
DO
    sentencia_sql;
```

### Ejemplos:
```sql
-- Evento que se ejecuta cada día
CREATE EVENT VerificarPrestamosVencidos
ON SCHEDULE EVERY 1 DAY
STARTS CURRENT_TIMESTAMP
DO
    UPDATE prestamos 
    SET estado = 'vencido' 
    WHERE fecha_vencimiento < CURDATE() AND estado = 'activo';

-- Evento con procedimiento
CREATE EVENT GenerarMultasAutomaticas
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
ON COMPLETION PRESERVE
ENABLE
DO
    CALL ProcesarMultasPorRetraso();

-- Evento una sola vez
CREATE EVENT LimpiezaMensual
ON SCHEDULE AT '2024-12-31 23:59:59'
DO
    DELETE FROM auditoria_libros 
    WHERE fecha_accion < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);

-- Habilitar el programador de eventos
SET GLOBAL event_scheduler = ON;

-- Ver eventos activos
SHOW EVENTS;
```

## Comandos útiles para gestión:

```sql
-- Eliminar
DROP FUNCTION IF EXISTS nombre_funcion;
DROP PROCEDURE IF EXISTS nombre_procedimiento;
DROP TRIGGER IF EXISTS nombre_trigger;
DROP EVENT IF EXISTS nombre_evento;

-- Mostrar código
SHOW CREATE FUNCTION nombre_funcion;
SHOW CREATE PROCEDURE nombre_procedimiento;
SHOW CREATE TRIGGER nombre_trigger;

-- Ver procedimientos y funciones
SHOW PROCEDURE STATUS WHERE Db = 'tu_base_datos';
SHOW FUNCTION STATUS WHERE Db = 'tu_base_datos';
```
