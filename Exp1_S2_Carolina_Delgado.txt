/*
    CASO 1: GENERACIÓN DE USUARIOS Y CLAVES
*/

/* CONFIGURACIÓN DE LA LOCALIZACIÓN
   Se establecen los parámetros NLS para asegurar que el idioma (mensajes, nombres de días/meses)
   sea Español y el territorio (moneda, separadores decimales, formato fecha por defecto) sea Chile.
*/
ALTER SESSION SET NLS_LANGUAGE = 'SPANISH';
ALTER SESSION SET NLS_TERRITORY = 'CHILE';

/* DEFINICIÓN DE VARIABLE BIND 
   Se define una variable bind para la fecha de proceso, permitiendo que el 
   cálculo sea paramétrico y no dependa de fechas fijas en el código.
*/
VARIABLE b_fecha_proceso VARCHAR2(10);
EXEC :b_fecha_proceso := TO_CHAR(SYSDATE, 'DD/MM/YYYY');

-- Truncar la tabla
TRUNCATE TABLE USUARIO_CLAVE;

/* BLOQUE PL/SQL ANÓNIMO 
*/
DECLARE
    -- Cursor explícito para recorrer todos los empleados
    CURSOR cur_empleados IS
        SELECT e.id_emp, 
               e.numrun_emp, 
               e.dvrun_emp, 
               e.pnombre_emp, 
               e.snombre_emp,
               e.appaterno_emp, 
               e.apmaterno_emp, 
               e.fecha_nac, 
               e.fecha_contrato, 
               e.sueldo_base, 
               ec.nombre_estado_civil
        FROM empleado e
        JOIN estado_civil ec ON e.id_estado_civil = ec.id_estado_civil -- Con estado civil
        ORDER BY e.id_emp;

    -- Variable de tipo registro para el cursor
    reg cur_empleados%ROWTYPE;

    -- Variables escalares
    v_nombre_completo   usuario_clave.nombre_empleado%TYPE;
    v_run_emp            empleado.numrun_emp%TYPE;
    v_sueldo            empleado.sueldo_base%TYPE;
    v_estado_civil      estado_civil.nombre_estado_civil%TYPE;
    v_appaterno         empleado.appaterno_emp%TYPE;
    
    -- Variables para lógica de negocio
    v_fecha_proc        DATE;
    v_annos_servicio    NUMBER(3);
    v_nombre_usuario    VARCHAR2(50);
    v_clave_usuario     VARCHAR2(50);
    
    -- Variables auxiliares para construcción de usuario
    v_letra_ec          VARCHAR2(1);
    v_3let_nom          VARCHAR2(3);
    v_largo_nom          NUMBER(2);
    v_ult_dig_sueldo    VARCHAR2(1);
    v_sufijo_usuario    VARCHAR2(1);
    
    -- Variables auxiliares para construcción de clave
    v_3er_dig_run       VARCHAR2(1);
    v_anno_nac_calc     NUMBER(4);
    v_ult_3_sueldo_calc NUMBER(4);
    v_letras_clave      VARCHAR2(2);
    v_mes_anio_db       VARCHAR2(6);
    
    -- Contador de iteraciones
    v_contador          NUMBER := 0;
    v_total_esperado    NUMBER;

BEGIN

    -- Convertir la variable bind a fecha para cálculos internos
    v_fecha_proc := TO_DATE(:b_fecha_proceso, 'DD/MM/YYYY');
    
    -- Obtener total para validación final de commit
    SELECT COUNT(*) INTO v_total_esperado FROM empleado;

    -- Inicio de Iteración
    OPEN cur_empleados;
    LOOP
        FETCH cur_empleados INTO reg;
        EXIT WHEN cur_empleados%NOTFOUND;
        
        -- Inicializar variables locales con datos del cursor
        v_run_emp       := reg.numrun_emp;
        v_sueldo        := reg.sueldo_base;
        v_estado_civil  := reg.nombre_estado_civil;
        v_appaterno     := reg.appaterno_emp;
        
        -- Construcción del Nombre Completo (Pnombre + Appaterno + Apmaterno)
        v_nombre_completo := reg.pnombre_emp || ' ' || reg.appaterno_emp || ' ' || reg.apmaterno_emp;

        -- Nombre de usuario
        
        -- 1. Primera letra estado civil (minúscula)
        v_letra_ec := SUBSTR(LOWER(v_estado_civil), 1, 1);
        
        -- 2. Tres primeras letras del primer nombre
        v_3let_nom := SUBSTR(reg.pnombre_emp, 1, 3);
        
        -- 3. Largo del primer nombre
        v_largo_nom := LENGTH(reg.pnombre_emp);
        
        -- 4. Último dígito sueldo base
        v_ult_dig_sueldo := SUBSTR(TO_CHAR(v_sueldo), -1);
        
        -- 5. Años trabajados (Calculado fecha contrato vs fecha proceso)
        -- Uso de funciones SQL dentro de PL/SQL para cálculo preciso
        v_annos_servicio := TRUNC(MONTHS_BETWEEN(v_fecha_proc, reg.fecha_contrato) / 12);
        
        -- 6. Sufijo 'X' si lleva menos de 10 años
        IF v_annos_servicio < 10 THEN
            v_sufijo_usuario := 'X';
        ELSE
            v_sufijo_usuario := '';
        END IF;

        -- CONCATENACIÓN FINAL USUARIO
        v_nombre_usuario := v_letra_ec || v_3let_nom || v_largo_nom || '*' || 
                            v_ult_dig_sueldo || reg.dvrun_emp || v_annos_servicio || v_sufijo_usuario;


        -- Clave
        
        -- 1. Tercer dígito del RUN
        v_3er_dig_run := SUBSTR(TO_CHAR(v_run_emp), 3, 1);
        
        -- 2. Año nacimiento + 2
        v_anno_nac_calc := TO_NUMBER(TO_CHAR(reg.fecha_nac, 'YYYY')) + 2;
        
        -- 3. Tres últimos dígitos sueldo base - 1
        v_ult_3_sueldo_calc := TO_NUMBER(SUBSTR(TO_CHAR(v_sueldo), -3)) - 1;
        
        -- 4. Letras apellido según Estado Civil (Estructura condicional requerida)
        IF v_estado_civil IN ('CASADO', 'ACUERDO DE UNION CIVIL') THEN
            -- Dos primeras letras
            v_letras_clave := SUBSTR(LOWER(v_appaterno), 1, 2);
            
        ELSIF v_estado_civil IN ('DIVORCIADO', 'SOLTERO') THEN
            -- Primera y última letra
            v_letras_clave := SUBSTR(LOWER(v_appaterno), 1, 1) || SUBSTR(LOWER(v_appaterno), -1);
            
        ELSIF v_estado_civil = 'VIUDO' THEN
            -- Antepenúltima y penúltima letra
            -- Ejemplo: Si es "PEREZ" (Largo 5), Antepenúltima es 'R' (-3), Penúltima es 'E' (-2)
            v_letras_clave := SUBSTR(LOWER(v_appaterno), -3, 1) || SUBSTR(LOWER(v_appaterno), -2, 1);
            
        ELSIF v_estado_civil = 'SEPARADO' THEN
            -- Dos últimas letras
            v_letras_clave := SUBSTR(LOWER(v_appaterno), -2);
        END IF;
        
        -- 5. Mes y año de la base de datos (Numérico, formato MMYYYY)
        v_mes_anio_db := TO_CHAR(v_fecha_proc, 'MMYYYY');

        -- CONCATENACIÓN FINAL CLAVE
        v_clave_usuario := v_3er_dig_run || v_anno_nac_calc || v_ult_3_sueldo_calc || 
                           v_letras_clave || reg.id_emp || v_mes_anio_db;

        -- Insertar Datos
        INSERT INTO USUARIO_CLAVE (
            id_emp, 
            numrun_emp, 
            dvrun_emp, 
            nombre_empleado, 
            nombre_usuario, 
            clave_usuario
        ) VALUES (
            reg.id_emp,
            reg.numrun_emp,
            reg.dvrun_emp,
            v_nombre_completo,
            v_nombre_usuario,
            v_clave_usuario
        );
        
        -- Incrementar contador
        v_contador := v_contador + 1;

    END LOOP; -- Fin iteración empleados
    CLOSE cur_empleados;

    -- Confirmación de transacciones
    IF v_contador = v_total_esperado THEN
        COMMIT;
    ELSE
        ROLLBACK;
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        IF cur_empleados%ISOPEN THEN
            CLOSE cur_empleados;
        END IF;
        ROLLBACK;
END;
/

/* Consulta para validar resultados */
SELECT * FROM USUARIO_CLAVE ORDER BY id_emp;
