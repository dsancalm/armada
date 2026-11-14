# Pruebas en consola: perfil de rendimiento por juego

Guía para desplegar y probar en una consola Armada la rama
`claude/blissful-noether-uoniio` **sin reinstalar el sistema**. Todo lo que se
despliega aquí se deshace reiniciando la consola.

Este fichero es material de pruebas. Antes de abrir el PR a `main` hay que
decidir si se queda en el repo o se borra.

## Qué se está probando

Cada juego puede tener su propio perfil (Eco / Balanced / Performance),
elegido en Armada Control > juego > Avanzado > Performance > "Power Profile".

- Al lanzar el juego se aplica su perfil.
- Al salir vuelve el perfil del sistema (el del menú rápido de Steam).
- Un juego sin perfil usa el del sistema, como hasta ahora.
- Si cambias el perfil en el menú rápido **mientras corre un juego con perfil
  propio**, el cambio vale solo hasta que ese juego termina. No se guarda como
  perfil del sistema.
- El perfil del juego nunca se escribe en `/var/lib/armada/powerd.state`: si
  la consola se apaga a mitad de partida, arranca con el perfil del sistema.

## Estado de la verificación

Lo que ya está comprobado en PC:

- `tests/perf-settings-test.sh` pasa, con casos nuevos para todo lo anterior.
- Tests e i18n del plugin pasan y `npm run build` genera `dist`.

Lo que **no** está comprobado y depende de esta prueba:

- Nada se ha ejecutado en una consola real.
- Los tests corrieron con 8 CPUs simuladas (la máquina de desarrollo tenía 4).
  La lógica está validada; la afinidad real en 8 cores, no.
- El desplegable del plugin no se ha visto renderizado.
- No se sabe si Steam refresca el menú rápido cuando powerd cambia el perfil
  por su cuenta (caso 1).
- No se sabe si Steam reescribe el perfil al entrar o salir de un juego
  (caso 16).

Supuestos sobre la consola sin verificar. Si alguno falla en el paso 2, para:

- `bootc usr-overlay` está disponible.
- El usuario `armada` tiene `sudo`.
- SELinux no bloquea los ficheros copiados (`getenforce`).

## Requisitos

- PC con este repo, `git`, `ssh`/`scp` y Node 22 (el mismo que usa el
  `Containerfile` para compilar el plugin).
- Consola y PC en la misma red.
- SSH activado en la consola: Armada Control > Settings. Usuario `armada`.

En los comandos, sustituye `IP` por la dirección de la consola.

## 1. Preparar en el PC

Bajar la rama:

```bash
git fetch origin claude/blissful-noether-uoniio && git checkout claude/blissful-noether-uoniio
```

Compilar el plugin (deja el resultado en `decky/armada-control/dist`):

```bash
cd decky/armada-control && npm ci && npm run build && cd ../..
```

El build muestra un aviso de tipos en `src/lib/lockWatcher.ts`. Ya estaba en
`main` y no impide que se genere `dist`.

## 2. Desplegar en la consola

### 2.1 Hacer `/usr` escribible

En la consola (por SSH o en su terminal). Dura hasta el próximo reinicio:

```bash
sudo bootc usr-overlay
```

Si el comando no existe, la alternativa es un `mount --bind` de cada fichero
modificado sobre el original, que también se pierde al reiniciar.

### 2.2 Ficheros de sistema (cuatro)

Desde el PC, en la raíz del repo:

```bash
ssh armada@IP 'mkdir -p /tmp/pp'
```

```bash
scp system_files/usr/lib/armada/armada_perf.py \
    system_files/usr/libexec/armada/armada-control \
    system_files/usr/libexec/armada/armada-powerd \
    system_files/usr/bin/armada-power armada@IP:/tmp/pp/
```

```bash
ssh -t armada@IP '
  sudo install -m 0644 /tmp/pp/armada_perf.py /usr/lib/armada/armada_perf.py &&
  sudo install -m 0755 /tmp/pp/armada-control /usr/libexec/armada/armada-control &&
  sudo install -m 0755 /tmp/pp/armada-powerd  /usr/libexec/armada/armada-powerd &&
  sudo install -m 0755 /tmp/pp/armada-power   /usr/bin/armada-power &&
  sudo systemctl restart armada-powerd armada-control'
```

Qué hace cada fichero:

| Fichero | Papel |
|---|---|
| `armada_perf.py` | Valida `powerProfile` y lo lee del estado compartido |
| `armada-control` | Publica el perfil del juego al lanzarlo y lo retira al salir |
| `armada-powerd` | Aplica el perfil efectivo y lo expone a Steam |
| `armada-power` | CLI; `status` muestra ahora también `system_profile` |

Si los servicios no arrancan, mira el journal y el contexto SELinux
(`ls -Z /usr/libexec/armada/`, `chcon --reference=<original>`).

### 2.3 Plugin de Decky

```bash
scp -r decky/armada-control/dist armada@IP:/var/home/armada/homebrew/plugins/armada-control/
```

```bash
ssh -t armada@IP 'sudo systemctl restart plugin_loader'
```

## 3. Comprobar que el despliegue está activo

En la consola:

```bash
armada-power status
```

- Debe aparecer una línea `system_profile=` justo debajo de `profile=`. Si no
  está, sigue corriendo el `armada-powerd` antiguo: repite el paso 2.2.
- `profile` es el perfil efectivo (el que se está aplicando ahora).
  `system_profile` es el del sistema (el que se guarda).

En Armada Control:

- Con un juego seleccionado, en Avanzado > Performance aparece el desplegable
  "Power Profile" con "Use System Profile" y los tres perfiles.
- En la vista global (sin juego seleccionado) **no** aparece.

## 4. Preparación de las pruebas

1. En la pestaña Power, edita los tres perfiles para que se distingan a
   simple vista: governor distinto o frecuencia máxima de CPU distinta.
2. Deja el sistema en **Eco** desde el menú rápido de Steam.
3. Elige un **juego X** y asígnale **Performance** en "Power Profile".
4. Elige un **juego Y** y déjalo en "Use System Profile".
5. Comprueba en las propiedades de X e Y que las opciones de lanzamiento
   contienen `/usr/libexec/armada/armada-game-launch %command%`. Sin ese
   wrapper el perfil por juego no se aplica (limitación conocida).

## 5. Cómo observar

Deja una sesión SSH con el journal abierto durante toda la prueba:

```bash
journalctl -u armada-powerd -u armada-control -f
```

En otra sesión, cada vez que toque mirar el estado:

```bash
armada-power status; cat /run/armada/perf-state.json; cat /var/lib/armada/powerd.state
```

Para ver que el perfil se aplica de verdad al hardware:

```bash
grep . /sys/devices/system/cpu/cpufreq/policy*/scaling_governor /sys/devices/system/cpu/cpufreq/policy*/scaling_max_freq
```

Mensajes del journal que escribe esta función:

| Mensaje | Significado |
|---|---|
| `game profile performance, now performance` | Entró un juego con perfil propio |
| `game profile cleared, now eco` | El juego salió (o se le quitó el perfil) |
| `profile set to balanced` | Alguien cambió el perfil **del sistema** |
| `profile set to balanced until the game exits` | Cambio manual durante un juego con perfil; no se guarda |

Tiempos: los cambios tardan hasta 3 s (el ciclo de powerd). Si el cambio viene
del plugin, suma cerca de 1 s más por el guardado diferido.

## 6. Casos de prueba

Sigue este orden. La numeración es la del documento de diseño, por eso no es
correlativa. Rellena la última columna con OK, FALLO o una nota.

| # | Qué haces | Qué debes ver | Resultado |
|---|---|---|---|
| 1 | Lanzar X | `profile=Performance` y `system_profile=Eco`; `perf-state.json` tiene `override.powerProfile`; governor y frecuencias cambian; el menú rápido muestra Performance | |
| 2 | Salir de X | `profile=Eco`; `powerd.state` nunca dejó de decir `profile=eco` | |
| 3 | Lanzar Y | Sigue en Eco. En `perf-state.json` hay `override` pero **sin** `powerProfile` | |
| 7 | Con Y abierto, cambiar el perfil en el menú rápido | Cambia el perfil del sistema y `powerd.state`. El journal dice `profile set to …` sin "until the game exits". Vuelve a dejarlo en Eco | |
| 4 | Con X abierto, cambiar su perfil desde el plugin | Se aplica sin relanzar el juego | |
| 5 | Con X abierto, poner "Use System Profile" | Vuelve a Eco. Después vuelve a asignarle Performance | |
| 6 | Con X abierto, cambiar el perfil en el menú rápido | Se aplica; el journal dice `… until the game exits`; `system_profile=Eco` y `powerd.state` no cambian. Al salir de X vuelve Eco | |
| 8 | Suspender y reanudar con X abierto | Sigue el perfil de X | |
| 9 | Forzar el cierre de X | Vuelve Eco | |
| 10 | `sudo systemctl restart armada-powerd` con X abierto | Vuelve con el perfil de X | |
| 11 | `sudo systemctl restart armada-control` con X abierto | Mantiene el perfil de X y sigue volviendo a Eco al salir | |
| 13 | Con X abierto, mirar `fan_pwm` en `armada-power status` | La curva de ventilador sigue al perfil de X | |
| 14 | Desactivar los ajustes por juego de X y lanzarlo | Sigue en Eco; `override` sin `powerProfile` | |
| 15 | Quitar el wrapper de las opciones de lanzamiento de X y lanzarlo | Sigue en Eco y no hay `override` (limitación conocida) | |
| 16 | Activar el perfil por juego nativo de Steam (TDP) en X y repetir 1 y 2 | Igual que 1 y 2. En el journal no aparece ningún `profile set to` que no hayas hecho tú | |
| 12 | Apagar o reiniciar con X abierto. **El último: deshace el despliegue** | Tras arrancar, `powerd.state` dice `profile=eco` | |

Notas sobre casos concretos:

- **Caso 1, menú rápido.** Si las frecuencias cambian pero el desplegable de
  Steam sigue mostrando Eco, la función trabaja y solo falla el refresco
  visual. Anótalo así; no es lo mismo que un fallo de aplicación.
- **Caso 13, pestaña Fans.** La pestaña Fans de Armada Control lee el perfil
  de `powerd.state`, así que mostrará la curva del perfil del sistema aunque
  se esté aplicando la del juego. Es esperado. Comprueba con
  `armada-power status`.
- **Caso 16.** Si aparece un `profile set to` espontáneo al entrar o salir del
  juego, para ahí: significa que Steam escribe el perfil por su cuenta y hay
  que revisar el diseño antes de seguir.
- **Caso 12.** Tras el reinicio la consola vuelve al código original. Para
  repetir pruebas hay que desplegar otra vez desde el paso 2.

## 7. Si algo falla

Para cada fallo, guarda:

- El número de caso.
- La salida de los tres comandos de estado del punto 5.
- El trozo del journal de ese momento.

## 8. Volver atrás

- **Reiniciar la consola** restaura `/usr` y el plugin de la imagen
  (`armada-decky-sync.service` lo vuelve a copiar en cada arranque).
- Lo único que sobrevive al reinicio es la clave `powerProfile` en
  `/etc/armada/game-tweaks.json`. El código original la ignora. Para dejarlo
  limpio, pon "Use System Profile" en cada juego antes de reiniciar.

## Limitaciones conocidas

No son fallos de esta función; vienen del circuito de ajustes por juego:

- Solo aplica a juegos lanzados con `armada-game-launch %command%`.
- Hasta 3 s de retraso al entrar y al salir.
- Un solo juego a la vez: con dos abiertos manda el último lanzado.
- La curva de ventilador forma parte del perfil y cambia con él.
