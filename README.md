# Harem Club · Caja Puerta

Aplicación web para celulares que el personal de seguridad usa en la puerta: registro de entradas, cobro, correcciones, rendimiento por trabajador, cierre de caja, auditoría y Excel con diseño.

## 1. Archivos que se suben a GitHub
Sube **todos** estos archivos a la raíz del repositorio:

| Archivo | Para qué sirve |
|---|---|
| `index.html` | La aplicación completa |
| `manifest.json` | Datos de la app instalada (nombre, colores, iconos) |
| `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` | Iconos de la app |
| `apple-touch-icon.png` | Icono para iPhone |
| `README.md` | Esta guía |

## 2. Publicar en GitHub Pages
1. Crea un repositorio (ej. `harem-caja`) y sube los archivos.
2. **Settings → Pages → Branch: `main` / carpeta `/ (root)` → Save**.
3. En 1–2 minutos tendrás el link `https://TU-USUARIO.github.io/harem-caja/`.
4. En el celular abre el link y usa **Menú ⋮ → Instalar app / Añadir a pantalla de inicio**.

Para actualizar la app: reemplaza `index.html` en el repositorio y haz commit. Los datos no se tocan porque viven en Supabase.

## 3. Guardar los datos en la nube (Supabase)
GitHub Pages solo muestra la página, no guarda datos. Sin Supabase cada celular guardaría solo lo suyo y el administrador no vería lo de seguridad.

1. Crea un proyecto en https://supabase.com.
2. En **SQL Editor** ejecuta el siguiente script. Es seguro de repetir y **no borra datos existentes**:

```sql
create table if not exists registros (
  id text primary key,
  kind text,
  data jsonb not null,
  created_at timestamptz default now()
);
alter table registros enable row level security;

-- Se reemplaza la política abierta por políticas separadas: nadie puede borrar.
drop policy if exists "acceso_app" on registros;
drop policy if exists "leer" on registros;
drop policy if exists "insertar" on registros;
drop policy if exists "actualizar" on registros;
create policy "leer"       on registros for select using (true);
create policy "insertar"   on registros for insert with check (true);
create policy "actualizar" on registros for update using (true) with check (true);

-- La auditoría no se puede modificar: cualquier intento de cambiarla se ignora.
create or replace function proteger_auditoria() returns trigger
language plpgsql as $$
begin
  if old.kind = 'audit' then return null; end if;
  return new;
end $$;
drop trigger if exists t_proteger_auditoria on registros;
create trigger t_proteger_auditoria before update on registros
  for each row execute function proteger_auditoria();
```

3. En **Project Settings → API** copia `Project URL` y la clave pública (`anon` / `publishable`).
4. En `index.html`, sección `CONFIG`, completa `SUPABASE_URL` y `SUPABASE_KEY`.
5. Haz commit. El indicador de arriba dirá **Guardado en la nube**.

La app sincroniza cada 20 segundos; no usa tiempo real instantáneo.

## 4. Usuarios y contraseñas
En `CONFIG.USUARIOS` de `index.html` hay 1 usuario **Administrador** y 3 de seguridad (Carlos, Rocky, Ralfo), cada uno con su contraseña y su serie de boleta (AD02, WA02, WA03, WA04). Para cambiar una contraseña edita su línea `clave:` y sube el archivo de nuevo.

## 5. Qué hace la app
**Registro de entradas**
- Tipos: Casa, Taxi, Marketing, Botella y Free Pass.
- Precio por persona: S/ 50, 60, 70, 80, 90, 100 y 130. El total es precio × personas.
- **Botella:** no usa precios fijos; se escribe el monto que se quiera cobrar.
- Pagos: Efectivo, Yape, Plin, Transferencia y Tarjeta. Ningún método lleva recargo.
- **Taxi:** exige nombre y celular (9 dígitos que empiecen con 9).
- **Free Pass:** gratis; exige autorizador y motivo. Suma personas, no dinero. El administrador lo ve en su Panel y lo marca como verificado.

**Seguridad (Carlos, Rocky, Ralfo)**
- Ven el contador de personas ingresadas hoy y el rendimiento de todos los compañeros (personas, operaciones, pagadas, Free Pass e ingresos).
- Pueden **corregir sus propios registros de la jornada actual** (con motivo obligatorio). El administrador recibe un aviso con los datos anteriores y nuevos.
- No pueden anular directamente: **solicitan la anulación** con motivo y el administrador la aprueba o rechaza.
- No pueden tocar registros de otros ni de jornadas anteriores.
- **Cierre de caja:** al cerrar, el contador, el rendimiento y los movimientos vuelven a 0 y el cierre queda guardado en «Cierres anteriores». Desde ahí pueden abrirlo con «Ver y corregir» (sus registros de la jornada actual; el administrador recibe un aviso de cada cambio) y descargar su Excel.

**Administrador**
- Panel en vivo, Avisos (correcciones y solicitudes de anulación), Historial por día con búsqueda por taxista/celular y filtro por método de pago.
- Corrige o anula cualquier registro; puede **revertir** una corrección de seguridad con motivo.
- Auditoría con rendimiento por trabajador y filtros por trabajador, fecha (calendario), y tipo de entrada.
- Excel del día o de todo el historial.

**Reglas de cálculo**
- La jornada va hasta las 6 a.m., en hora de Lima: lo registrado después de medianoche pertenece a la noche anterior.
- Las operaciones anuladas no suman personas ni dinero. Una anulación pendiente sigue contando hasta que el administrador decida y se muestra aparte.
- Nada se borra: las correcciones y anulaciones conservan los valores originales en la auditoría.

## 6. Limitaciones de seguridad (importante)
- Las contraseñas están en el código de la página. Sirven para ordenar el acceso del personal, **no para frenar a alguien con conocimientos técnicos**.
- Los permisos (quién edita qué, solo la jornada actual, etc.) se aplican en la aplicación, **no en Supabase**. Como el acceso no usa cuentas de Supabase, alguien con la clave pública podría escribir datos directamente en la tabla. Lo que sí protege Supabase con el script de arriba: **nadie puede borrar registros y la auditoría no se puede modificar**.
- Para seguridad real el siguiente paso es migrar a **Supabase Auth** (login por correo) con políticas por usuario. Aún no está hecho.
- No hay notificaciones push: los avisos aparecen al abrir la app o en el siguiente refresco (20 s).
- Las boletas se numeran por serie de usuario; dos celulares con el mismo usuario a la vez podrían repetir número. Usa un usuario por celular.
- La app necesita conexión para guardar en la nube; sin conexión guarda en el celular y sincroniza al volver.
