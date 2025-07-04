**Taller MySQL – Parte 2 – Daniel Durán**

**5. Subconsultas**

1\. Consultar el producto más caro en cada categoría

SELECT 
    t.tipo_nombre,
    p.nombre,
    p.precio
FROM Productos p
JOIN tiposproductosjq t ON p.tipo_id = t.id
WHERE p.precio = (
    SELECT MAX(precio) 
    FROM Productos p2 
    WHERE p2.tipo_id = p.tipo_id
);

2\. Encontrar el cliente con mayor total en pedidos

SELECT 
    c.nombre,
    SUM(p.total) as total_gastado
FROM Clientes c
JOIN Pedidos p ON c.id = p.cliente_id
GROUP BY c.nombre
ORDER BY total_gastado DESC
LIMIT 1;

3\. Listar empleados que ganan más que el salario promedio

SELECT 
    nombre,
    salario
FROM Empleados
WHERE salario > (SELECT AVG(salario) FROM Empleados)
ORDER BY salario DESC;

4\. Consultar productos que han sido pedidos más de 5 veces

SELECT 
    p.nombre,
    COUNT(*) as veces_pedido
FROM Productos p
JOIN detallespedido dp ON p.id = dp.producto_id
GROUP BY p.nombre
HAVING COUNT(*) > 5
ORDER BY veces_pedido DESC;

5\. Listar pedidos cuyo total es mayor al promedio de todos los pedidos

SELECT 
    p.id,
    p.fecha,
    p.total,
    c.nombre
FROM Pedidos p
JOIN Clientes c ON p.cliente_id = c.id
WHERE p.total > (SELECT AVG(total) FROM Pedidos)
ORDER BY p.total DESC;

6\. Seleccionar los 3 proveedores con más productos

SELECT 
    p.nombre,
    COUNT(*) as total_productos
FROM Proveedores p
JOIN Productos pr ON p.id = pr.proveedor_id
GROUP BY p.nombre
ORDER BY total_productos DESC
LIMIT 3;

7\. Consultar productos con precio superior al promedio en su tipo.

SELECT 
    p.id,
    p.nombre,
    p.precio,
    tp.tipo_nombre,
    tp.descripcion AS tipo_descripcion
FROM Productos p
INNER JOIN TiposProductos tp ON p.tipo_id = tp.id
WHERE p.precio > (
    SELECT AVG(p2.precio)
    FROM Productos p2
    WHERE p2.tipo_id = p.tipo_id
)
ORDER BY tp.tipo_nombre, p.precio DESC;

8\. Mostrar clientes que han realizado más pedidos que la media

SELECT 
    c.id,
    c.nombre,
    c.email,
    COUNT(p.id) AS total_pedidos,
    ROUND((SELECT AVG(pedidos_por_cliente) 
           FROM (SELECT COUNT(id) AS pedidos_por_cliente 
                 FROM Pedidos 
                 GROUP BY cliente_id) AS subquery), 2) AS promedio_pedidos
FROM Clientes c
INNER JOIN Pedidos p ON c.id = p.cliente_id
GROUP BY c.id, c.nombre, c.email
HAVING COUNT(p.id) > (
    SELECT AVG(pedidos_por_cliente)
    FROM (
        SELECT COUNT(id) AS pedidos_por_cliente
        FROM Pedidos
        GROUP BY cliente_id
    ) AS promedio_tabla
)
ORDER BY total_pedidos DESC;

9\. Encontrar productos cuyo precio es mayor que el promedio de todos los productos.

SELECT 
    p.id,
    p.nombre,
    p.precio,
    tp.tipo_nombre,
    prov.nombre AS proveedor,
    ROUND((SELECT AVG(precio) FROM Productos), 2) AS promedio_general
FROM Productos p
INNER JOIN TiposProductos tp ON p.tipo_id = tp.id
INNER JOIN Proveedores prov ON p.proveedor_id = prov.id
WHERE p.precio > (SELECT AVG(precio) FROM Productos)
ORDER BY p.precio DESC;

10\. Mostrar empleados cuyo salario es menor al promedio del departamento.

SELECT 
    e.id,
    e.nombre,
    e.puesto AS departamento,
    e.salario,
    ROUND((SELECT AVG(e2.salario) 
           FROM Empleados e2 
           WHERE e2.puesto = e.puesto), 2) AS promedio_departamento,
    e.fecha_contratacion
FROM Empleados e
WHERE e.salario < (
    SELECT AVG(e2.salario)
    FROM Empleados e2
    WHERE e2.puesto = e.puesto
)
ORDER BY e.puesto, e.salario;

**6. Procedimientos Almacenados**

1\. Crear un procedimiento para actualizar el precio de todos los productos de un proveedor.

DELIMITER $$

CREATE PROCEDURE ActualizarPreciosProveedor(
    IN proveedor_id INT,
    IN porcentaje DECIMAL(5,2)
)
BEGIN
    DECLARE proveedor_existe INT DEFAULT 0;
    SELECT COUNT(*) INTO proveedor_existe 
    FROM Proveedores 
    WHERE id = proveedor_id;
    
    IF proveedor_existe = 0 THEN
        SELECT 'Error: Proveedor no encontrado' AS mensaje;
    ELSE
        UPDATE Productos 
        SET precio = ROUND(precio * (1 + porcentaje/100), 2)
        WHERE proveedor_id = proveedor_id;
        
        SELECT 
            CONCAT('Precios actualizados para el proveedor: ', 
                   (SELECT nombre FROM Proveedores WHERE id = proveedor_id)) AS mensaje,
            ROW_COUNT() AS productos_actualizados,
            CONCAT(porcentaje, '%') AS porcentaje_aplicado;
    END IF;
END$$

DELIMITER ;

2\. Un procedimiento que devuelva la dirección de un cliente por ID.

DELIMITER //

CREATE PROCEDURE ObtenerDireccionCliente(
    IN cliente_id_param INT
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SELECT 'Error: No se pudo obtener la dirección del cliente' AS mensaje;
    END;
    
    SELECT 
        c.nombre AS nombre_cliente,
        uc.direccion,
        uc.ciudad,
        uc.estado,
        uc.codigo_postal,
        uc.pais,
        CONCAT(uc.direccion, ', ', uc.ciudad, ', ', uc.estado, ' ', uc.codigo_postal, ', ', uc.pais) AS direccion_completa
    FROM 
        Clientes c
    INNER JOIN 
        UbicacionCliente uc ON c.id = uc.cliente_id
    WHERE 
        c.id = cliente_id_param;
    
    IF ROW_COUNT() = 0 THEN
        SELECT CONCAT('No se encontró información de dirección para el cliente con ID: ', cliente_id_param) AS mensaje;
    END IF;
END //

DELIMITER ;

3\. Crear un procedimiento que registre un pedido nuevo y sus detalles

DELIMITER //

CREATE PROCEDURE RegistrarPedidoSencillo(
    IN cliente_id INT,
    IN producto_id INT,
    IN cantidad INT
)
BEGIN
    DECLARE nuevo_pedido_id INT;
    DECLARE precio_producto DECIMAL(10,2);
    DECLARE total_pedido DECIMAL(10,2);
    
    SELECT precio INTO precio_producto 
    FROM Productos 
    WHERE id = producto_id;
    
    SET total_pedido = precio_producto * cantidad;
    
    INSERT INTO Pedidos (cliente_id, fecha, total)
    VALUES (cliente_id, CURDATE(), total_pedido);
    
    SET nuevo_pedido_id = LAST_INSERT_ID();
    
    INSERT INTO DetallesPedido (pedido_id, producto_id, cantidad, precio)
    VALUES (nuevo_pedido_id, producto_id, cantidad, precio_producto);
    
    SELECT 
        'Pedido creado exitosamente' AS mensaje,
        nuevo_pedido_id AS pedido_id,
        total_pedido AS total;
END //

DELIMITER ;

4\. Un procedimiento para calcular el total de ventas de un cliente

DELIMITER //

CREATE PROCEDURE TotalVentasCliente(
    IN cliente_id INT
)
BEGIN
    DECLARE total_ventas DECIMAL(10,2) DEFAULT 0;
    DECLARE num_pedidos INT DEFAULT 0;
    DECLARE nombre_cliente VARCHAR(100);
    
    SELECT nombre INTO nombre_cliente 
    FROM Clientes 
    WHERE id = cliente_id;
    
    SELECT 
        COALESCE(SUM(total), 0),
        COUNT(*)
    INTO total_ventas, num_pedidos
    FROM Pedidos 
    WHERE cliente_id = cliente_id;
    
    SELECT 
        cliente_id AS id_cliente,
        nombre_cliente AS cliente,
        total_ventas AS total_ventas,
        num_pedidos AS numero_pedidos;
END //

DELIMITER ;

5\. Crear un procedimiento para obtener los empleados por puesto.

DELIMITER //

CREATE PROCEDURE EmpleadosPorPuesto(
    IN puesto_buscado VARCHAR(50)
)
BEGIN
    SELECT 
        id AS id_empleado,
        nombre AS nombre_empleado,
        puesto,
        salario,
        fecha_contratacion AS fecha_contrato
    FROM Empleados
    WHERE puesto = puesto_buscado
    ORDER BY nombre;
    
    SELECT 
        puesto_buscado AS puesto,
        COUNT(*) AS total_empleados,
        AVG(salario) AS salario_promedio,
        MIN(salario) AS salario_minimo,
        MAX(salario) AS salario_maximo
    FROM Empleados
    WHERE puesto = puesto_buscado;
END //

DELIMITER ;

6\. Un procedimiento que actualice el salario de empleados por puesto.

DELIMITER //

CREATE PROCEDURE ActualizarSalarioPuesto(
    IN puesto_objetivo VARCHAR(50),
    IN nuevo_salario DECIMAL(10,2)
)
BEGIN
    DECLARE empleados_afectados INT;
    
    SELECT COUNT(*) INTO empleados_afectados
    FROM Empleados
    WHERE puesto = puesto_objetivo;
    
    IF empleados_afectados = 0 THEN
        SELECT 
            'No se encontraron empleados con ese puesto' AS mensaje,
            puesto_objetivo AS puesto_buscado,
            0 AS empleados_actualizados;
    ELSE
        UPDATE Empleados
        SET salario = nuevo_salario
        WHERE puesto = puesto_objetivo;
        
        SELECT 
            'Salarios actualizados exitosamente' AS mensaje,
            puesto_objetivo AS puesto,
            nuevo_salario AS nuevo_salario,
            empleados_afectados AS empleados_actualizados;
    END IF;
END //

DELIMITER ;

7\. Crear un procedimiento que liste los pedidos entre dos fechas

DELIMITER //

CREATE PROCEDURE ListarPedidosEntreFechas(
    IN fecha_inicio DATE,
    IN fecha_fin DATE
)
BEGIN
    IF fecha_inicio > fecha_fin THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La fecha de inicio no puede ser posterior a la fecha fin';
    END IF;
    
    SELECT 
        p.id AS pedido_id,
        p.fecha AS fecha_pedido,
        p.total AS total_pedido,
        c.id AS cliente_id,
        c.nombre AS nombre_cliente,
        c.email AS email_cliente,
        uc.ciudad AS ciudad_cliente,
        uc.pais AS pais_cliente,
        COUNT(dp.id) AS cantidad_productos,
        SUM(dp.cantidad) AS total_items
    FROM Pedidos p
    INNER JOIN Clientes c ON p.cliente_id = c.id
    LEFT JOIN UbicacionCliente uc ON c.id = uc.cliente_id
    LEFT JOIN DetallesPedido dp ON p.id = dp.pedido_id
    WHERE p.fecha BETWEEN fecha_inicio AND fecha_fin
    GROUP BY p.id, p.fecha, p.total, c.id, c.nombre, c.email, uc.ciudad, uc.pais
    ORDER BY p.fecha DESC, p.total DESC;
    
    SELECT 
        COUNT(*) AS total_pedidos,
        SUM(total) AS suma_total,
        AVG(total) AS promedio_total,
        MIN(total) AS pedido_minimo,
        MAX(total) AS pedido_maximo,
        MIN(fecha) AS primera_fecha,
        MAX(fecha) AS ultima_fecha
    FROM Pedidos
    WHERE fecha BETWEEN fecha_inicio AND fecha_fin;
    
END //

DELIMITER ;

8\. Un procedimiento para aplicar un descuento a productos de una categoría

DELIMITER //

CREATE PROCEDURE AplicarDescuentoCategoria(
    IN categoria_id INT,
    IN porcentaje_descuento DECIMAL(5,2)
)
BEGIN
    IF categoria_id IS NULL OR porcentaje_descuento IS NULL THEN
        SELECT 'Error: Parámetros no pueden ser nulos' AS mensaje;
    ELSEIF porcentaje_descuento <= 0 OR porcentaje_descuento > 100 THEN
        SELECT 'Error: El porcentaje debe estar entre 0.01 y 100' AS mensaje;
    ELSE
        UPDATE Productos 
        SET precio = precio * (1 - porcentaje_descuento / 100)
        WHERE tipo_id = categoria_id;
        
        SELECT 
            ROW_COUNT() AS productos_actualizados,
            porcentaje_descuento AS descuento_aplicado,
            'Descuento aplicado exitosamente' AS mensaje;
            
        SELECT 
            p.id,
            p.nombre,
            tp.tipo_nombre AS categoria,
            p.precio AS precio_final
        FROM Productos p
        INNER JOIN TiposProductos tp ON p.tipo_id = tp.id
        WHERE p.tipo_id = categoria_id;
    END IF;
END //

DELIMITER ;

9\. Crear un procedimiento que liste todos los proveedores de un tipo de producto.

DELIMITER //

CREATE PROCEDURE ListarProveedoresPorTipo(
    IN tipo_id INT
)
BEGIN
    IF tipo_id IS NULL THEN
        SELECT 'Error: El ID del tipo de producto no puede ser nulo' AS mensaje;
    ELSE
        SELECT DISTINCT
            prov.id AS proveedor_id,
            prov.nombre AS nombre_proveedor,
            prov.contacto,
            prov.telefono,
            prov.direccion,
            tp.tipo_nombre AS tipo_producto,
            COUNT(prod.id) AS cantidad_productos
        FROM Proveedores prov
        INNER JOIN Productos prod ON prov.id = prod.proveedor_id
        INNER JOIN TiposProductos tp ON prod.tipo_id = tp.id
        WHERE tp.id = tipo_id
        GROUP BY prov.id, prov.nombre, prov.contacto, prov.telefono, prov.direccion, tp.tipo_nombre
        ORDER BY prov.nombre;
        
        SELECT 
            COUNT(DISTINCT prov.id) AS total_proveedores,
            tp.tipo_nombre AS tipo_producto,
            COUNT(prod.id) AS total_productos
        FROM Proveedores prov
        INNER JOIN Productos prod ON prov.id = prod.proveedor_id
        INNER JOIN TiposProductos tp ON prod.tipo_id = tp.id
        WHERE tp.id = tipo_id
        GROUP BY tp.tipo_nombre;
    END IF;
END //

DELIMITER ;

10\. Un procedimiento que devuelva el pedido de mayor valor

DELIMITER //

CREATE PROCEDURE PedidoMayorValor()
BEGIN
    SELECT 
        p.id AS pedido_id,
        p.fecha AS fecha_pedido,
        p.total AS valor_pedido,
        c.id AS cliente_id,
        c.nombre AS nombre_cliente,
        c.email AS email_cliente,
        uc.ciudad AS ciudad_cliente,
        uc.pais AS pais_cliente
    FROM Pedidos p
    INNER JOIN Clientes c ON p.cliente_id = c.id
    LEFT JOIN UbicacionCliente uc ON c.id = uc.cliente_id
    WHERE p.total = (SELECT MAX(total) FROM Pedidos)
    ORDER BY p.fecha DESC
    LIMIT 1;
    
    SELECT 
        dp.id AS detalle_id,
        prod.nombre AS nombre_producto,
        tp.tipo_nombre AS categoria,
        dp.cantidad,
        dp.precio AS precio_unitario,
        (dp.cantidad * dp.precio) AS subtotal
    FROM DetallesPedido dp
    INNER JOIN Productos prod ON dp.producto_id = prod.id
    INNER JOIN TiposProductos tp ON prod.tipo_id = tp.id
    WHERE dp.pedido_id = (
        SELECT id FROM Pedidos 
        WHERE total = (SELECT MAX(total) FROM Pedidos)
        ORDER BY fecha DESC
        LIMIT 1
    )
    ORDER BY subtotal DESC;
END //

DELIMITER ;
