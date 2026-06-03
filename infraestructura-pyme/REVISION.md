# REVISION.md – Reflexión del equipo

---

## 1. ¿Qué conflictos tuvisteis y cómo los resolvisteis?

Tuvimos dos conflictos a lo largo del proyecto.

El primero fue en `02-diseno.md`, en la sesión 2. Los dos editamos la tabla de versiones de software a la vez sin coordinarnos, y cuando intentamos mergear el segundo PR, GitHub nos avisó de que había conflicto. Abrimos el archivo, vimos los marcadores `<<<<<<<` y `>>>>>>>`, y decidimos quedarnos con la versión más actualizada de Apache y añadir también la fila de Certbot que había añadido el otro. Lo resolvimos con `git add`, `git commit` y `git push` normal.

El segundo fue en `ssh-firewall.md`, en la sesión 3. Pasó algo parecido pero esta vez lo resolvimos con rebase en lugar de merge. Hicimos `git pull --rebase origin main`, resolvimos el conflicto a mano combinando las dos versiones del archivo, y luego `git rebase --continue` y `git push --force-with-lease`. El resultado es un historial más limpio que con merge.

---

## 2. ¿Qué comandos Git usasteis más?

Los que más usamos fueron `git status`, `git add`, `git commit` y `git push` y `git pull`, que son los del día a día. También usamos mucho `git checkout -b` para crear las ramas de cada tarea.

Los que menos conocíamos al principio eran `git pull --rebase` y `git push --force-with-lease`. Al principio siempre tirábamos de merge porque era lo que sabíamos, pero con el conflicto de la sesión 3 entendimos para qué sirve el rebase.

---

## 3. ¿Qué haríais diferente en un próximo proyecto?

Hacer `git pull` más a menudo antes de ponerse a editar. Los dos conflictos que tuvimos fueron porque estuvimos trabajando en paralelo demasiado tiempo sin sincronizar.

También habría estado bien decidir desde el principio qué secciones concretas iba a tocar cada uno dentro de cada archivo, no solo repartir los archivos. Así habríamos evitado pisar el trabajo del otro.

Y escribir mejores mensajes de commit desde el principio. Al principio poníamos cosas como "cambios" o "actualizado" y luego cuesta entender el historial.

---

## 4. ¿Cómo os funcionó el intercambio de roles?

Mejor de lo esperado. Al tener que trabajar sobre lo que había escrito el otro, primero tuviste que leerlo bien para entenderlo, y eso hizo que detectáramos cosas que se podían mejorar o que no eran del todo coherentes con el resto.

Lo más raro fue editar archivos que había escrito el compañero, porque al principio no sabías muy bien hasta dónde podías cambiar. Pero una vez que lo hablamos no hubo problema.

En general creemos que es una buena práctica porque obliga a que los dos conozcan todo el proyecto, no solo su parte.

---

*Ver también: [CHANGELOG.md](CHANGELOG.md) | [README.md](README.md)*