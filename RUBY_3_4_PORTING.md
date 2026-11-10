# Integración de Ruby 3.4 en MKXP-Z

Registro del port de Ruby 3.1.3 a **Ruby 3.4.12**, realizado en noviembre de
2026. Describe los cambios aplicados y los problemas encontrados para servir
de referencia en futuras actualizaciones. No es una declaración de soporte
para todas las plataformas ni de mejora de rendimiento.

## 1. Punto de partida y estado

- Rama de trabajo: `deps/ruby-3.4`, creada desde `dev`, commit
  `f02d586624dc` (prefijo del commit base).
- Se añadió Ruby 3.4 como alternativa. Ruby 3.1 sigue siendo el valor
  predeterminado; sus archivos de integración no se modificaron.
- La compilación comprobada es Windows x64, desde Linux/WSL, con MinGW
  GCC 13 **Win32**, `release`, `-O3`, LTO y enlace estático de Ruby.
- La compilación terminó y se verificaron estáticamente el formato PE x64
  y las importaciones de DLL. El usuario confirmó el arranque sin errores.
  Eso no sustituye las comprobaciones de partidas, extensiones y plataformas.
- No se modificaron el ejecutable original del juego ni sus scripts de
  `Data/`. Tampoco se ha cambiado la configuración predeterminada a Ruby 3.4.
- Durante las compilaciones de noviembre de 2026 el port aún no estaba
  versionado. Consultar Git para conocer el estado actual.

La candidata comprobada se creó como `candidate-S1RF3l69`. SHA-256 de su
`Game.exe`:

```text
4c663119a8d1fdf8a85cc9ddf9019085da535dd23069da294f958d4516c63e0d
```

## 2. Selección de versión y archivos añadidos

La selección ya existía en el motor:

- [meson_options.txt](./meson_options.txt): opción `mri_version`, por defecto
  `3.1`.
- [src/meson.build](./src/meson.build): selecciona el subproyecto
  `ruby-<mri_version>`.
- [binding/meson.build](./binding/meson.build): lee esa misma versión.
- [meson.build](./meson.build): instala la biblioteca Ruby desde
  `lib/ruby/<mri_version>.0`.

No fue necesario modificar esos cuatro archivos. Se añadieron:

1. [subprojects/ruby-3.4.wrap](./subprojects/ruby-3.4.wrap).
2. [subprojects/packagefiles/ruby-3.4/](./subprojects/packagefiles/ruby-3.4/).
3. Los 22 parches `ruby-3.4-*.patch` enumerados en la sección siguiente.

Datos del archivo fuente:

```text
URL: https://github.com/ruby/ruby/archive/refs/tags/v3_4_12.tar.gz
Archivo: ruby-3_4_12.tar.gz
Directorio extraído: ruby-3_4_12
SHA-256: bf8a420cf41b56354d2e5f3e8effb1c0e03552854bc987d657256969ef116efb
```

El paquete `ruby-3.4/` se copió del de Ruby 3.1. Los cambios están en
`meson.build` y en el `configure` generado. Los helpers, los archivos de
configuración de plataforma y el listado vacío de bundled gems se conservaron.
Un `configure~` generado como copia de respaldo no forma parte del port y no
debe incluirse en una entrega.

## 3. Inventario de parches, en orden de aplicación

El orden efectivo está en el wrap. No reordenar parches sin volver a comprobar
su aplicación: varios modifican el mismo `configure.ac` o Makefile.
Los enlaces siguientes apuntan a los cambios concretos, no solo a su resumen.

| Parche | Responsabilidad |
|---|---|
| [alignof](./subprojects/packagefiles/ruby-3.4-alignof.patch) | Desactivar una comprobación problemática de alineación para i386 FreeBSD. |
| [bsd](./subprojects/packagefiles/ruby-3.4-bsd.patch) | Adaptaciones de compilación para FreeBSD. |
| [bundle](./subprojects/packagefiles/ruby-3.4-bundle.patch) | Evitar `-bundle_loader` en el enlace estático de extensiones en macOS. |
| [darwin-in6](./subprojects/packagefiles/ruby-3.4-darwin-in6.patch) | Evitar modificar un header de sistema que puede no existir en macOS. |
| [deps](./subprojects/packagefiles/ruby-3.4-deps.patch) | Usar la libffi seleccionada por el build de MKXP-Z para Fiddle. |
| [disable-lto-for-link-tests](./subprojects/packagefiles/ruby-3.4-disable-lto-for-link-tests.patch) | Permitir desactivar LTO en pruebas de enlace de extensiones mediante una variable de configuración. |
| [dllexport](./subprojects/packagefiles/ruby-3.4-dllexport.patch) | Retirar import/export de DLL de los headers usados para enlazar Ruby estáticamente en Windows. |
| [exp](./subprojects/packagefiles/ruby-3.4-exp.patch) | No generar `ruby.exp` en el caso de enlace estático de Windows. |
| [fake](./subprojects/packagefiles/ruby-3.4-fake.patch) | Compatibilidad del script `-fake.rb` con la construcción en Windows. |
| [init](./subprojects/packagefiles/ruby-3.4-init.patch) | Crear archivos estáticos para inicialización de extensiones y codificaciones. |
| [libruby-static](./subprojects/packagefiles/ruby-3.4-libruby-static.patch) | Seleccionar libruby-static para los ejecutables auxiliares del build de Windows. |
| [libs](./subprojects/packagefiles/ruby-3.4-libs.patch) | Posponer las comprobaciones de libcrypt y zlib hasta el final de `configure.ac`. |
| [pkg-config](./subprojects/packagefiles/ruby-3.4-pkg-config.patch) | Omitir la generación forzada de la entrada pkg-config de Ruby. |
| [program](./subprojects/packagefiles/ruby-3.4-program.patch) | Usar baseruby, omitir el ejecutable Ruby independiente y desactivar la generación del builtin binario precompilado. |
| [rbinstall](./subprojects/packagefiles/ruby-3.4-rbinstall.patch) | Instalar default gems en el destino del Ruby construido, no en el del Ruby anfitrión. |
| [reconfig](./subprojects/packagefiles/ruby-3.4-reconfig.patch) | Evitar una segunda ejecución de configure sin las variables que suministró Meson. |
| [verbose](./subprojects/packagefiles/ruby-3.4-verbose.patch) | Mostrar los comandos de compilación, también al construir extensiones. |
| [win32-winnt](./subprojects/packagefiles/ruby-3.4-win32-winnt.patch) | Evitar redefinir `_WIN32_WINNT` cuando ya procede del cross-file. |
| [windows-builtin-setjmp](./subprojects/packagefiles/ruby-3.4-windows-builtin-setjmp.patch) | No forzar builtin setjmp ni emular round; corregir el orden de inclusión de setjmpex. |
| [mkxp-z-patches-windows-libs](./subprojects/packagefiles/ruby-3.4-mkxp-z-patches-windows-libs.patch) | Enlazar pthread y, si existe, libgcc estáticamente en Windows. |
| [mkxp-z-patches](./subprojects/packagefiles/ruby-3.4-mkxp-z-patches.patch) | Conservar las modificaciones de semántica Ruby y compatibilidad Win32 propias de MKXP-Z. |
| [gcc-lto-setjmp](./subprojects/packagefiles/ruby-3.4-gcc-lto-setjmp.patch) | Evitar un fallo interno de GCC 13 al optimizar `ractor_check_ints` con LTO. |

El último elemento del wrap es el
[end-of-diff-files.patch](./subprojects/packagefiles/end-of-diff-files.patch)
ya existente. Añade un archivo marcador para mantener una coma después de cada
parche funcional; no cambia el runtime.

### Parches de Ruby 3.1 que no se trasladaron como archivos separados

| Parche anterior | Decisión del port |
|---|---|
| `bison` | Ruby 3.4 utiliza Lrama; se retiró también la variable `YACC` del paquete Meson. |
| `ccache` | El bloque de configuración de MJIT que modificaba ya no está en Ruby 3.4. |
| `extmk` | Ruby 3.4 ya define `Gem.target_rbconfig`. |
| `random` | El arreglo anterior era específico de x86 Windows; no se trasladó al port x64. Revisarlo si se añade x86. |
| `round` | Su cambio quedó incluido en `windows-builtin-setjmp`, que retira el bloque que forzaba emular `round`. |
| `windows-arm64` | Los cambios correspondientes ya están en la fuente de Ruby 3.4. No se probó aquí Windows ARM64. |
| `clock-gettime` | La fuente de Ruby 3.4 ya contiene el ajuste del parche anterior. Esto no corrige el problema distinto del reloj del GC descrito más abajo. |

### Cambios de semántica que deben conservarse conscientemente

Este Ruby no es un intérprete sin modificar. El parche `mkxp-z-patches`
mantiene comportamientos que ya tenía la distribución de MKXP-Z:

- Bypass de comprobaciones de métodos privados y protegidos en
  `rb_method_call_status` y `vm_call_method`.
- En Ruby 3.4 se añadió el bypass de `vm_call_symbol`: el despacho utilizado
  por `Symbol#to_proc` y `public_send` comprueba visibilidad por una ruta
  separada. Copiar solamente el parche antiguo dejaba esos casos sin cubrir.
- Omisión del error de superclass mismatch en `rb_define_class` y
  `vm_check_if_class`.
- La comprobación de visibilidad de `defined?` no se modificó, como en el
  port de Ruby 3.1.
- Renombrado de la macro Ruby `Sleep` a `RB_Sleep`; se conservaron las
  llamadas a la API Win32 `Sleep` y se ajustaron los correspondientes `#undef`.
- Omisión del aviso/error de argumentos sobrantes de `sprintf`.
- Uso de `CREATE_NO_WINDOW` en la ruta de creación de procesos parcheada.

En otra actualización hay que localizar todas esas rutas de nuevo. La
aplicación limpia de un diff no prueba que siga cubriendo el despacho actual.

## 4. Adaptación de meson.build y de las extensiones

En [ruby-3.4/meson.build](./subprojects/packagefiles/ruby-3.4/meson.build):

- Se cambió el nombre del proyecto y del override de dependencia a `ruby-3.4`.
- Se retiró el workaround `-fno-sanitize=address` del paquete antiguo.
  La build validada no activa ASan.
- Se retiró `YACC`, al cambiar el generador del parser.
- Se retiraron de la lista explícita de archivos estáticos `bigdecimal`,
  `nkf`, `racc/cparse` y `syslog`; se añadió `ext/erb/escape/escape.a`.
  No asumir que las bibliotecas retiradas estén disponibles con `require`:
  si el juego las necesita habrá que integrar sus versiones correspondientes.
- Se conserva la construcción estática, sin RubyGems, sin GMP y excluyendo
  las extensiones openssl, readline, dbm y gdbm según la integración previa.
- Se añadió `stack_protector=no` al entorno de configure.
- Para libYAML vendorizada se añadieron `--with-libyaml-include` y
  `--with-libyaml-lib`. Las rutas actuales contienen `libyaml-0.2.5`; hay que
  revisarlas si cambia esa dependencia.

El resto de los archivos estáticos de extensiones y codificaciones se comparó
con los producidos por el build. El caso de `pty` en plataformas no Windows
sigue declarado, pero no se validó en esta compilación.

## 5. Bloqueos encontrados y soluciones aplicadas

### Configure: «cannot compute sizeof(long long)»

WSL puede ejecutar los conftest `.exe` mediante interoperabilidad con Windows.
Configure llegó a tratar el entorno como ejecutable en lugar de limitarse a
cross-compilar. Desde Ruby 3.2, el stack protector de MinGW podía introducir
una dependencia de `libssp-0.dll`; sin esa DLL el conftest fallaba.

Se usó `stack_protector=no`, manteniendo el comportamiento de la build anterior
de Ruby 3.1. Es una decisión de configuración, no una solución general para
todo entorno Windows. No debe extrapolarse a builds con otros requisitos de
endurecimiento; una alternativa futura es integrar correctamente su runtime.

### Builtins: «RubyVM.each_builtin is undefined»

El Ruby anfitrión era 3.2, no 3.4. El build intentaba generar
`builtin_binary.inc` con una API propia del intérprete correspondiente.

El parche `program` fuerza tanto la selección de baseruby como
`X_BUILTIN_BINARY=no`. No basta con cambiar solamente la ruta de miniruby.
El helper [build-miniruby.py](./subprojects/packagefiles/ruby-3.4/build-miniruby.py)
conservado genera un wrapper hacia baseruby con `-fake.rb`: no es un intérprete
Windows Ruby 3.4 utilizable para probar el juego.

### Enlace: «undefined reference to setjmpex»

`scheduler.o` utilizaba `RUBY_SETJMP`. `vm_core.h` incluía `<setjmp.h>` antes
de que otros headers incluyeran `<setjmpex.h>`, por lo que MinGW no dejaba
definida la macro `setjmpex`.

Se añadió a `windows-builtin-setjmp` la inclusión de `<setjmpex.h>` desde
`vm_core.h` cuando `RUBY_USE_SETJMPEX` está activado. No se cambió arbitrariamente
la selección de setjmp ni su ABI.

### GCC 13 + LTO: «internal compiler error: SSA corruption»

Algunas pruebas de enlace de las extensiones acababan incorporando el VM y
fallaban al optimizar `ractor_check_ints`. Eso podía hacer que mkmf detectase
funciones existentes como ausentes. El fallo visible de strscan era:
«static declaration of rb_reg_onig_match follows non-static declaration».

Se reprodujo el fallo de enlace fuera del juego. El nuevo parche
`gcc-lto-setjmp` añade `__attribute__((__noipa__))` a esa función para GCC,
pero no para Clang. Con ello se conservó LTO en el resto del build.
La condición actual no está limitada a GCC 13; revisarla al cambiar de
compilador. No se forzó un resultado positivo de detección de strscan.

### Psych: falta `ext/psych/psych.a`

La versión de Psych incluida ya no trae libYAML dentro. Sus comprobaciones
no encontraban `yaml.h`: se recibían rutas a `subprojects/libyaml`, pero el
directorio real era `subprojects/libyaml-0.2.5`.

Las opciones explícitas de libYAML del paquete Meson hicieron que se detectaran
el header y `yaml_get_version`, y se construyera `psych.a`.

### Mensajes ignorados y avisos

La regla del target auxiliar `dummy` intentaba ejecutar `STRIP=false` y
mostraba un error ignorado. Eso era distinto de un fallo de enlace.
La compilación final terminó con avisos de fuentes upstream; no se declara
que el build esté libre de warnings.

## 6. Configure generado y procedimiento para otra actualización

[ruby-3.4/configure](./subprojects/packagefiles/ruby-3.4/configure) se generó con
**GNU Autoconf 2.71**, desde la fuente con los parches aplicados. No editar
manualmente ese archivo como sustituto de modificar `configure.ac`.

Procedimiento recomendado:

1. Elegir el tag exacto, descargar su archivo y calcular su SHA-256.
2. Crear un wrap y paquete separados si cambia la serie de Ruby. Conservar
   el port anterior como referencia.
3. Aplicar y revisar cada parche sobre una extracción limpia, en orden.
   Investigar los parches que fallen o parezcan redundantes; no aceptar
   automáticamente fuzz ni trasladar todos los cambios a ciegas.
4. Revisar especialmente el VM, Win32, los headers y las extensiones que
   hayan pasado a bundled gems o cambiado de directorio.
5. Regenerar configure con una versión de Autoconf compatible:
   `autoconf -o /ruta/al/paquete/ruby-X.Y/configure`, desde la fuente parcheada.
6. Comprobar de nuevo todos los diffs sobre otra extracción limpia y comparar
   el resultado con el árbol utilizado para generarlos. Esto se hizo para
   el port actual; los parches se verificaron también después de añadir el
   arreglo de LTO.
7. Reextraer el subproyecto y regenerar su build cuando cambien el wrap,
   los diffs o los packagefiles. Meson no vuelve a copiar automáticamente
   los packagefiles sobre una extracción ya existente.
   Identificar exactamente los directorios de esa versión antes de apartarlos;
   no borrar el árbol general de subprojects ni otros builds.
8. Construir y revisar los logs de configure/mkmf, los archivos estáticos y
   las importaciones de DLL. Una detección «no» puede ocultar un fallo del
   compilador, no solo una API ausente.
9. Preparar una candidata con su propia stdlib y probarla manualmente.

Para este proyecto, el helper situado **fuera del repositorio MKXP-Z**,
`../update_mkxp.sh`, recibió la opción `--ruby X.Y`. Esta opción:

- Pasa `-Dmri_version=X.Y`.
- Separa el build alternativo con el sufijo `-rubyX.Y`.
- No reutiliza el enlace al build histórico de 3.1 para una serie diferente.
- Rechaza deploy/restore con la opción de Ruby alternativa.

Desde el repositorio del juego, con la rama de port actual:

```bash
bash update_mkxp.sh --ruby 3.4 --jobs 8 --reconfigure
```

El helper conserva logs y opciones en el directorio de build. Su
`source-changes.patch` solo contiene cambios de archivos seguidos por Git.
Las candidatas iniciales se construyeron cuando el port aún no estaba
versionado, por lo que ese parche y el commit base no identifican por sí
solos su fuente completa. En builds posteriores el helper guarda también
`source-untracked.tar.gz`, con wraps y parches de `subprojects/` y archivos
de `subprojects/packagefiles/` sin seguimiento, incluidos los ignorados en
esas rutas. No incluye otros archivos sin seguimiento.

## 7. Distribución y comprobaciones manuales

- Ruby está enlazado en el ejecutable candidato. La DLL antigua
  `x64-msvcrt-ruby310.dll` no indica la versión de ese ejecutable ni se
  utiliza para proporcionar su Ruby.
- La ruta estándar de instalación es `lib/ruby/3.4.0`, aunque el intérprete
  informe `RUBY_VERSION=3.4.12`. El componente `3.4.0` no es el patchlevel.
- La candidata usa `rubyLoadpath` con `.`, `stdlib` y `stdlib/x64-mingw64`.
- Mantener separadas las stdlib de Ruby 3.1 y 3.4. El helper local
  `../copy_mkxp_runtime.sh`, situado fuera de este repositorio, elimina
  `stdlib/` del destino tras confirmar `SI` y copia la del origen. También
  actualiza el ejecutable, la configuración, la licencia, las DLL de la raíz
  y `scripts/`, pero no elimina archivos sobrantes fuera de `stdlib/`.
- El helper de copia no crea backups. Cerrar el juego y respaldar el runtime
  del destino antes de usarlo: si la copia falla tras eliminar `stdlib/`, el
  destino puede quedar incompleto. `--dry-run` no modifica archivos, pero
  compara contra la stdlib existente y no muestra necesariamente todos los
  archivos que se eliminarán o volverán a copiar en la ejecución real.
  En la prueba anterior se apartaron la stdlib y el exe como
  `stdlib-ruby3.1` y `Game-ruby3.1.exe`; el helper no crea esas copias.
- La creación de candidatas de `../update_mkxp.sh` copia primero el juego y
  sincroniza `stdlib/` con el artefacto, eliminando archivos sobrantes de la
  candidata. Si era un enlace simbólico, lo retira de la copia sin seguirlo.
  Meson reutiliza el directorio del artefacto, por lo que también hay que
  comprobar que su stdlib corresponda a la versión compilada. El modo
  `--deploy` todavía superpone la stdlib en el juego, aunque crea antes un
  backup completo; comprobar que no queden archivos mezclados al desplegar.
- La línea antigua que añade `Data/Ruby Library 3.3.0` al load path no se
  modificó. Se añade al final y puede proporcionar bibliotecas que no estén
  presentes antes en el load path. Su retirada y la revisión de JSON quedan
  pendientes de la adopción definitiva del runtime.
- La integración Ruby 3.4 no incluye las extensiones estáticas `bigdecimal`,
  `nkf`, `racc/cparse` ni `syslog`. La extensión `bigdecimal.so` de
  `Data/Ruby Library 3.3.0/x64-mingw-ucrt` depende de
  `x64-ucrt-ruby330.dll`; no debe asumirse que sea compatible con el Ruby
  3.4 estático de la candidata.
- El `preload.rb` del juego contiene aliases de `Dir.exists?` y
  `File.exists?`, eliminados en Ruby 3.2. Sin embargo, las candidatas no
  declaran `preloadScript` en sus `mkxp.json`, y el valor nativo por defecto
  es una lista vacía. Una configuración de usuario podría modificarlo.
  Antes de activar ese archivo hay que adaptarlo o verificarlo: llama a
  `MKXP.puts`, mientras este binding registra `System.puts`.
- Sin esos aliases, hay dos incompatibilidades probables que no se han
  probado en el juego: el exportador de mapas usa `Dir.exists?` y las
  operaciones `CFG[]` y `CFG[]=` llaman a `File.exists?` desde el binding.
  `CFG.to_hash`, que utiliza el profiler, sigue otra ruta.
- YJIT está desactivado en las candidatas Windows; en Ruby 3.4 tampoco se
  compiló RJIT. No atribuirles mejoras.

Respaldar las partidas **fuera de las carpetas candidatas**: dos copias del
juego pueden usar la misma ubicación externa de guardado.

Comprobaciones del usuario:

1. Confirmar Ruby 3.4.12 al arrancar y revisar errorlog/mkxp.log.
2. Cargar una copia de una partida de 3.1; guardar, cerrar y recargar.
3. Recorrer menús, PC, combates y funciones MakerStudio que usen JSON.
4. Comparar rendimiento bajo las condiciones de la siguiente sección.
5. Validar por separado otras plataformas antes de declarar soporte.

## 8. Comparación de rendimiento y protocolo de prueba

La primera comparación **no aisló solamente Ruby**:

| Candidata | Fuente y toolchain |
|---|---|
| Ruby 3.1.3, `candidate-LJxh8xAx` | Rama de FluidSynth 2.6.2 con los cambios de Scene, filtro de sprites y cachés de texto; MinGW POSIX. |
| Ruby 3.4.12, `candidate-S1RF3l69` | Base `dev` con el port Ruby; MinGW Win32, sin aquellos cambios de optimización. |

En dos ejecuciones del benchmark, la media de los mejores totales fue
1170,0 ms frente a 1624,5 ms. La prueba de pasabilidad pasó de 678,55 a
1117,05 ms. Son observaciones de esas candidatas, no una regresión atribuida
de manera aislada a Ruby.

Para comparar con la misma base `dev`/Win32 se construyó Ruby 3.1 con esa
base y toolchain. Puede reconstruirse en el checkout `deps/ruby-3.4`,
seleccionando 3.1; sus packagefiles nuevos no se seleccionan en ese caso:

```bash
bash update_mkxp.sh --ruby 3.1 --jobs 8 --reconfigure
```

El directorio de compilación de 3.1 pudo utilizar antes FluidSynth 2.6.2.
Comprobar tras reconfigurar que `build.ninja` utiliza `fluidsynth-2.5.0`, igual
que la candidata 3.4; no dar por buena la comparación si aún usa 2.6.2. La
compilación `candidate-ARRujZpB` terminó con esa versión y MinGW Win32.

En la comparación posterior entre `candidate-ARRujZpB` y
`candidate-jU8zOKbN`, ambas con `dev`, MinGW Win32 y FluidSynth 2.5.0, la
media de las sumas de medianas fue aproximadamente un 22 % mayor con Ruby
3.4; en pasabilidad fue aproximadamente un 46 % mayor. Son tiempos de
cargas sintéticas, no una caída equivalente de FPS. Las pocas invocaciones y
las diferencias de estado limitan la atribución de los resultados.

Al preparar la comparación se observaron diferencias entre la copia Windows
3.4 y el juego que se copió para 3.1: MakerStudio (`meta.txt` y
`014_GraphicRegion.rb`), `Data/MapInfos.rxdata` y `Data/encounters.dat`,
además de archivos de datos solo presentes en la copia Windows. El juego
carga los archivos de `Data/Scripts/` desde el cargador de
`Data/Scripts.rxdata`; las dos copias de esos scripts coincidían. El profiler
1.1.0 estaba en la candidata 3.1, mientras la copia Windows 3.4 conservaba
1.0.0. Igualar los datos y plugins partiendo de una copia común, no
superponiendo las bibliotecas estándar de versiones distintas.

Repetir las **mediciones** de ambas versiones tras cualquier ajuste del
profiler, aunque se reutilice el ejecutable 3.4. Usar la misma versión del
plugin, datos, copia inicial de partida, mapa, configuración de gráficos y
turbo desactivado. Mantener `fixedFramerate=120` y
`syncToRefreshrate=false`. Alternar los runtimes y repetir varias veces.

### Problema pendiente: reloj del GC de Windows

En estas compilaciones, `current_process_time` del GC usa `GetProcessTimes`.
Su conversión guarda en `tv_nsec` el resto de FILETIME, que está en unidades
de 100 ns, sin multiplicarlo por 100. El código existe tanto en Ruby 3.1
como en Ruby 3.4.

Esto puede infravalorar el tiempo o introducir incrementos cercanos a
1000 ms al cruzar un segundo de tiempo de CPU. Un intervalo real de
5 ms entre 2,999 y 3,004 segundos se convierte en unos 990,05 ms.
Por tanto, `gc_ms=991` o `1991` no demuestra pausas reales de esa duración.
Los tiempos del benchmark usan `CLOCK_MONOTONIC`, no ese contador.

**No se ha aplicado un parche a ese reloj.** El profiler marca `gc_ms` como
`unreliable_windows` en Windows y conserva `GC.count`. No arreglarlo
multiplicando el resultado del plugin: el cruce de segundo hace que esa
transformación no recupere la medida original. Si se corrige la fuente Ruby,
habrá que reconstruir ambas candidatas.

### Protocolo 2 del benchmark/profiler

Los archivos están en el plugin `[RM] Ruby Profiler` del repositorio del
juego, no en este repositorio MKXP-Z. La versión 1.1.0 registra:

- `protocol=2`, el FPS lógico (`Graphics.frame_rate`) y los valores
  configurados de `fixedFramerate`, `syncToRefreshrate`, `vsync` y
  `frameSkip`. Estos últimos proceden de `CFG.to_hash`, que refleja la
  configuración combinada cargada al iniciar; no miden el FPS real.
- `gc_ms=unreliable_windows` en Windows, sin descartar `gc_runs`.
- Ventanas de frames sin el benchmark bloqueante. Al cambiar de etiqueta se
  descarta el intervalo que cruza la transición, cuya división exacta no se
  conoce, y el primer update de la etiqueta nueva sirve como referencia. Los
  contadores del GC y de objetos se consultan en cada update para no atribuir
  a la ventana previa los cambios posteriores; estas consultas también
  añaden un pequeño coste de instrumentación. Los tiempos `ruby_*` son
  intervalos entre updates, no tiempo de CPU exclusivo de Ruby.
- Una ejecución de calentamiento por carga fuera de la medición y cinco
  repeticiones medidas, cada una precedida de `GC.start`. Se registran las
  cinco muestras, el mínimo y la mediana. `SUM_OF_MEDIANS` suma las medianas
  de las cinco cargas: **no** es la mediana de una ejecución integral.
- Mapa, dimensiones, coordenadas del jugador y número de eventos del mapa.
- La prueba de pasabilidad original, que consulta todo el mapa con
  `self_event=nil`. No representa exactamente el recorrido del jugador ni
  mide directamente los FPS al caminar.

Las cargas tienen cantidades fijas, pero no son estrictamente deterministas:
crear Pokémon consume aleatoriedad y compilar el guardado depende del estado
de la partida. Repetir desde una copia inicial común y no guardar sobre la
única partida original después de las pruebas.

## 10. Port experimental a Ruby 4.1.0-preview1

### Control y alcance

Trabajo iniciado el 10 de noviembre de 2026 en `deps/ruby-4.1`, desde
`deps/ruby-3.4` (`15ccd23483b6c341232467a6413dd1b697bdd2b2`).
El repositorio anidado solo tenía `mkxp-z.wiki/` sin seguimiento. Los cambios
locales del repositorio padre no forman parte del port.

La versión objetivo es **4.1.0-preview1**, no una versión estable. Ruby 3.1
continúa siendo el valor predeterminado; Ruby 3.4 conserva su integración.
El alcance es analizar, editar y compilar: no ejecutar el juego, copiar una
candidata sobre él, hacer commit, push ni desplegar.

Fuente oficial y checksum comprobado:

```text
https://cache.ruby-lang.org/pub/ruby/4.1/ruby-4.1.0-preview1.tar.gz
SHA-256: fb530bb88e90de3ae91c1310a27ae090e6bd989b8726de50ef12a890a85b88ef
```

### Integración y diferencias con 3.4

- [ruby-4.1.wrap](./subprojects/ruby-4.1.wrap) fija la versión preliminar.
- [packagefiles/ruby-4.1](./subprojects/packagefiles/ruby-4.1/) contiene la
  integración Meson, los mismos helpers de construcción de 3.4 y un
  `configure` regenerado con Autoconf 2.71. No requiere instalar Autoconf
  para consumir el wrap.
- [ruby-4.1-mkxp.patch](./subprojects/packagefiles/ruby-4.1-mkxp.patch)
  reúne los parches de construcción y de semántica del VM adaptados. Se
  comprobó su aplicación sin fuzz sobre una extracción limpia.
- Se conservan los bypass de visibilidad, `Symbol#to_proc`/`public_send`,
  superclass mismatch del VM y el override de `rb_define_class` de nivel
  superior. Esta última ruta pasó a `rb_define_class_id_under_no_pin`;
  el port separa su implementación para limitar el override a
  `rb_define_class`. Retira la nueva macro que omitía esa función; las
  APIs `_under` mantienen sus comprobaciones.
- `ractor_check_ints` ya no existe: no se traslada su workaround GCC
  `__noipa__` a otra función sin evidencia de un fallo.
- `rb_w32_start_process`, usado por el worker RJIT, tampoco existe.
  Su parche `CREATE_NO_WINDOW` no se aplica a otras rutas de `Process.spawn`.
- Monitor y Pathname pasan al core; Fiddle y win32ole son bundled gems.
  Ninguno conserva el archivo `.a` anterior bajo `ext/`, por lo que se
  retiran esos cuatro archivos del enlace. Como en 3.4, no se instalan
  bundled gems: Fiddle y win32ole no se distribuyen en este runtime mínimo.
  Los juegos que los requieran deben revisar sus dependencias y las ABI
  de sus extensiones nativas. Win32API sigue usando MiniFFI del motor.
- `io/wait` también está integrado en el core, que registra sus nombres de
  carga. Se retira `ext/io/wait/wait.a` del enlace; no existe en 4.1.
- Se añaden los archivos estáticos de `digest/blake3` y `digest/crc32`,
  nuevos en 4.1 y referenciados por su inicializador de extensiones.
- El tarball oficial incluye gems desempaquetadas en `.bundle`, a diferencia
  del archivo Git usado en 3.4. Vaciar `gems/bundled_gems` no evita que las
  plantillas nuevas intenten compilarlas. Se excluyen también de
  `configure-ext.mk.tmpl` y `exts.mk.tmpl`, incluso si existen makefiles
  antiguos. El primer intento enlazó bigdecimal/debug contra Ruby estático
  y falló con `undefined reference to lgamma_r`; esas extensiones dinámicas
  no forman parte del runtime mínimo ni se distribuyen.
- El baseruby Linux disponible es 3.2.3. La instalación con el `monitor.rb`
  de Ruby 4.1 falló porque ese archivo espera que `Monitor` exista en el core.
  La integración precarga el `monitor.rb` del anfitrión mediante su ruta
  absoluta, obtenida con el `RbConfig` del anfitrión antes de aplicar
  `-fake.rb`. Esto afecta a las herramientas de construcción, no al juego.
- La stdlib de esta preview se instala en **`4.1.0+5`**, no `4.1.0`.
  El subproyecto expone `ruby_stdlib_dir`; el instalador principal lo
  consulta y conserva la ruta anterior como valor predeterminado para
  otros subproyectos.
- Se conserva el control Windows x64, MinGW GCC 13 Win32, `release`,
  `-O3`, LTO y Ruby estático. También conserva `stack_protector=no` del
  control 3.4 para no añadir `libssp-0.dll`; no es una mejora de seguridad.

### YJIT: soporte de Windows no equivale a soporte de MinGW

El [anuncio oficial de Ruby 4.1.0-preview1](https://www.ruby-lang.org/en/news/2026/10/12/ruby-4-1-0-preview1-released/)
anuncia YJIT para Windows. La fuente de esta versión, en `configure.ac`,
rechaza explícitamente habilitarlo en MinGW:

```text
YJIT is not supported on mingw. Use win32\configure.bat --enable-yjit on mswin instead
```

Este port pasa `--disable-yjit --disable-zjit` explícitamente. Instalar Rust
o activar `YJITEnable` no resuelve esa incompatibilidad. No se ha eliminado
la comprobación upstream ni se ha cambiado silenciosamente a MSVC.

Una candidata Windows con YJIT necesita otra integración: MSVC, Rust para
el destino correspondiente, dependencias con CRT/ABI compatibles y enlace
de los archivos del JIT. También habría que revisar que el código generado
por YJIT respete los cambios de semántica del VM propios de MKXP-Z.

### Compilar sin ejecutar ni preparar una copia del juego

Desde la raíz de este repositorio MKXP-Z, con Meson y MinGW disponibles:

```bash
BUILD="$HOME/.cache/la-base-de-sky/mkxp-z/windows-x86_64-release-ruby4.1"
meson setup "$BUILD" . \
  --prefix "$BUILD/artifact" \
  --cross-file windows/meson-gcc-x86_64.txt \
  --buildtype release -Dstrip=true -Db_lto=true \
  -Dmri_version=4.1 -Ddefault_system=disabled
MESON_NUM_PROCESSES=2 meson compile -C "$BUILD" -j 2
meson install -C "$BUILD" --no-rebuild --tags mkxp-z
```

La instalación de Meson se limita al prefijo externo. El helper del
repositorio padre también permite compilar sin preparar una copia del juego:

```bash
bash ../update_mkxp.sh --ruby 4.1 --jobs 2 --build-only
```

`--jobs` limita tanto Ninja como los proyectos externos de Meson. Sin
`--build-only`, el helper prepara una copia candidata. No mezclar la stdlib
4.1 con la de otras versiones.

### Resultado de compilación e inspección estática

Compilación, enlace e instalación completados el 10 de noviembre de 2026.
La candidata está en
`~/.cache/la-base-de-sky/mkxp-z/windows-x86_64-release-ruby4.1/artifact/`.
No se ejecutó el binario ni se desplegó al juego.

- `mkxp-z.x86_64.exe`: PE32+ GUI x86-64, release, O3, LTO y strip.
- Versión encontrada en las cadenas del ejecutable:
  `ruby 4.1.0preview1 (2026-10-11 master 2d08b5f42f) [x64-mingw64]`.
- SHA-256 del ejecutable instalado:
  `5c00f5a09bf94eb817e2c65f114b1fb7575f94436d378d5ccbfcfb361699b03f`.
- Configuración generada: `RUBY_ABI_VERSION=5`, `USE_YJIT=0`,
  `USE_ZJIT=0`. La stdlib se instala desde `4.1.0+5` a `artifact/stdlib`.
- Los 18 imports DLL del PE son bibliotecas de Windows; no importa una
  DLL de Ruby ni otra DLL no sistema. Esto no comprueba cargas dinámicas
  mediante `LoadLibrary`, servicios del sistema ni disponibilidad en todas
  las versiones de Windows.
- Los 29 archivos estáticos de extensiones construidos están contemplados
  en el enlace. El wrap se reconstruyó desde el tarball verificado: los
  parches se aplicaron sin fuzz ni offsets y sus 26 fuentes, junto a los
  overlays, coincidieron con las fuentes usadas en la compilación.
- Se conservan fuera del juego los logs, opciones de Meson, diff del
  control, archivo de packagefiles nuevos y manifiestos de hashes. Son
  cambios sin commit sobre `15ccd23`; el hash Git por sí solo no identifica
  esta candidata.

La compilación tiene advertencias GCC/LTO: ODR en SPIRV-Headers y posibles
accesos fuera de límites en Ruby (`regparse.c`, SHA1), FreeType y libjxl.
No se han silenciado ni determinado aquí si son falsos positivos. No se
presenta como una build sin advertencias ni validada en ejecución.

Los hashes del ejecutable y configuración originales del juego siguen
iguales a los registrados al inicio. No se modificaron sus archivos ni
los cambios locales ajenos del repositorio padre.

El ejecutable 3.4 disponible previamente procede de otra captura de fuentes
(`f02d586...` con el port entonces sin seguimiento), no de una reconstrucción
exacta del control actual. Para una comparación atribuible al cambio de Ruby,
reconstruir también el control `15ccd23` con las mismas opciones.

### Arranque en Windows: `Errno::EBADF` con msvcrt

Síntoma de la primera candidata: el juego se cerraba a los pocos segundos sin
`errorlog.txt` y en modo debug quedaba en negro. Con trazas temporales en el
binding (ya retiradas) Ruby inicializaba bien y el fallo aparecía al leer el
primer archivo de `Data/Scripts`:

```text
000:Main:29:in 'IO#read': Bad file descriptor @ io_fread - Data/Scripts/001_Settings/002_Settings.rb (Errno::EBADF)
```

Causa: la build enlaza `msvcrt.dll` (`RUBY_MSVCRT_VERSION 60`, igual que 3.1 y
3.4), pero Ruby 4 eliminó de `win32/win32.c` las ramas para msvcrt y asume
UCRT: busca `__pioinfo` en `ucrtbase.dll` y usa la estructura `ioinfo` de UCRT,
así que cualquier descriptor abierto resulta inválido.

Solución: `ruby-4.1-mkxp.patch` restaura en `win32/win32.c` las ramas de 3.4
(`FILE`, `ioinfo` heredado, `__pioinfo` importado de msvcrt, `IOINFO_L2E 5` y
sin búsqueda en `ucrtbase`). Con UCRT el código no cambia. El EXE importa
`__pioinfo` de `msvcrt.dll`. Validado por el usuario en Windows: el juego
arranca.

Notas para depurar arranques:

- Sin `winConsole`, el binding redirige `$stdout`/`$stderr` a `NUL`; arrancar
  con `MKXPZ_WINDOWS_CONSOLE=1` o con el argumento `debug` para ver la salida.
- El `errorlog.txt` de Essentials se escribe en `System.data_directory`
  (`%APPDATA%\<título>\`), no junto al EXE.
- `PluginManager.error` termina con `Kernel.exit!`, sin errorlog.
- El `preload.rb` del juego llama a `MKXP.puts`, por lo que necesita antes el
  wrapper del motor:

```json
"rubyLoadpath": [".", "stdlib", "stdlib/x64-mingw64"],
"preloadScript": ["scripts/preload/mkxp_wrap.rb", "preload.rb"]
```

### Rendimiento y comprobaciones pendientes

**Hechos:** actualizar el intérprete cambia VM, GC y bibliotecas. El
anuncio y la fuente describen mejoras del GC; no son mediciones de
Essentials. Esta build no incorpora YJIT.

**Hipótesis:** un JIT soportado podría acelerar bucles Ruby calientes.
Las llamadas frecuentes a C++, las transferencias GPU, esperas, audio y
sincronización pueden limitar la ganancia. Un intérprete más nuevo
también puede rendir peor; las mediciones previas de 3.4 siguen siendo
relevantes como advertencia, no como pronóstico de 4.1.

**Pendiente de prueba manual por el usuario**, en una copia separada y con
backup independiente de las partidas externas:

1. Confirmar `RUBY_DESCRIPTION`, stdlib y ausencia de JIT; revisar los logs.
2. Cargar partida, recorrer mapas y eventos, abrir menús y entrar en combate.
3. Comprobar gráficos, audio, plugins, dependencias Ruby y extensiones nativas.
4. Guardar y recargar la copia de prueba; comprobar compatibilidad de Marshal.
5. Alternar control 3.4 y candidata 4.1 desde los mismos datos, configuración
   y estado inicial. Registrar calentamiento, medianas, dispersión y memoria.

Compilar e inspeccionar un PE no demuestra compatibilidad funcional ni
mejora de FPS. Para volver al control de fuentes, conservar primero el
diff y los packagefiles nuevos y regresar a `deps/ruby-3.4`; no es necesario
restaurar el juego porque no se despliega este port.
