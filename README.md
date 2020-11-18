# Triggers - Administración de Base de Datos

- Victoria Manrique Rolo

## Ejercicio 1
Dada la base de dato de viveros:
- Procedimiento: crear_email devuelva una dirección de correo electrónico con el siguiente formato:
- Un conjunto de caracteres del nombe y/o apellidos
- El carácter @.
- El dominio pasado como parámetro.

Una vez creada la tabla escriba un trigger con las siguientes características:

Trigger: trigger_crear_email_before_insert
- Se ejecuta sobre la tabla clientes.
- Se ejecuta antes de una operación de inserción.
- Si el nuevo valor del email que se quiere insertar es NULL, entonces se le creará automáticamente una dirección de email y se insertará en la tabla.
- Si el nuevo valor del email no es NULL se guardará en la tabla el valor del email.

### Procedimiento `crear_email`

Este procedimiento creará un email a partir de un dominio dado. Para poder crear el email necesitaremos el nombre del cliente y además una variable de salida que será el email en si, esta variable de la salida la indicamos con la palabra `OUT`. Con todos estos datos ya podemos crear el email, para ello usamos la función `CONCAT` que viene por defecto en SQL y esta concatenación la guardamos en la variable email.

```sql
CREATE DEFINER=`jeffrey`@`localhost` PROCEDURE `crear_email`(dominio varchar(25), nombre varchar(55), OUT email varchar(55))
BEGIN
 set email:= CONCAT(nombre, '@',dominio);
END
```

### Trigger en la tabla `Cliente`

En la tabla `Cliente`creamos un trigger, este trigger lo crearemos `before insert` para que creemos el email del cliente al vez que insertamos, para ello comprobamos si el campo `Email` es nulo o no, si es nulo rellenamos por defecto llamando a la procedure `crear_email` que nos devuelve el nivel a insertar.

```sql
CREATE DEFINER=`jeffrey`@`localhost` TRIGGER `viveros`.`Cliente_BEFORE_INSERT` BEFORE INSERT ON `Cliente` FOR EACH ROW
BEGIN
	call crear_email('viveros.com',  NEW.Nombre, @email);
    if NEW.Email is null
    then
		  set NEW.Email = @email;
	end if;
END
```

## Ejercicio 2

**Crear un trigger permita verificar que las personas en el Municipio del catastro no pueden vivir en dos viviendas diferentes**

Para lograr esto comprobamos si todos los campos, claves de `Piso` y claves de `Unidad Familiar`, son no nulos significa que una persona vive en un piso y en una unidad familiar a la vez por lo que lanzamos un error y no dejamos que inserte.

```sql
CREATE DEFINER=`jeffrey`@`localhost` TRIGGER `catastro`.`Persona_BEFORE_INSERT` BEFORE INSERT ON `Persona` FOR EACH ROW
BEGIN
	if NEW.Piso_idPiso is not null and NEW.Piso_Bloque_Construccion_Numero is not null 
    and NEW.Unidad_Familiar_Construccion_Numero1
    then
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Una persona no puede vivir en dos viviendas';
	end if;
END
```
## Ejercicio 3

**Crear el o los trigger que permitan mantener actualizado el stock de la base de dato de viveros.**

Para lograr esto creamos un trigger `before insert` en la tabla `Pedido_has_Producto`. El primer paso será contar la cantidad de productos existentes que se incluyen en el pedido, una vez logrado esto. Comprobamos que la cantidad que existen sea menor a la cantidad incluida en el pedido, si esto se cumple, realizamos un `UPDATE` en la tabla `Producto` donde cambiamos la columna `Stock`.

```sql
CREATE DEFINER=`jeffrey`@`localhost` TRIGGER `viveros`.`Pedido_has_Producto_BEFORE_INSERT` BEFORE INSERT ON `Pedido_has_Producto` FOR EACH ROW
BEGIN
	select Stock into @total from Producto where Código_Barras = NEW.Producto_Código_Barras;
    if @total > 0 and @total >= NEW.Cantidad
    then
		update Producto set Stock = Stock - NEW.Cantidad where Código_Barras = NEW.Producto_Código_Barras;
	else
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'No existen productos en stock';
	end if;
END
```