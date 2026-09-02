# El gate que fallaba abierto

*2026-09-02 · primera entrada*

Ayer descubrí que el candado que protege mis commits llevaba medio día sin
cerrar. No lanzaba errores, no avisaba nada, y todos mis commits "pasaban las
revisiones". Esa es exactamente la trampa, y por eso esta bitácora empieza aquí.

## El contexto en tres líneas

Trabajo con agentes de IA sobre sistemas en producción, y una de mis reglas
obliga a que **todo commit pase por un ciclo de revisión con constancia**
(receipt). El mecanismo es un hook global de `pre-commit` que consulta a una
herramienta de revisión ([gentle-ai](https://github.com/Gentleman-Programming/gentle-ai))
y bloquea el commit si no hay constancia. Durante la mañana funcionó: dos
commits sin receipt, dos bloqueos reales.

A mediodía actualicé la herramienta a su versión nueva. El resto del día cerré
siete commits, cada uno con su ciclo de revisión completo. Todo en verde.

## La contradicción que destapó todo

En la noche, otro usuario comentó en un issue mío: a él, con la misma versión,
la compuerta le **bloqueaba todo** — incluso con revisiones aprobadas. A mí me
dejaba pasar todo. Misma versión, síntomas opuestos. Uno de los dos entendía
mal, o había algo peor debajo.

En vez de discutir, medí. Repositorio desechable, cuatro comandos:

```
$ gentle-ai review validate --gate pre-commit ; echo $?
  "result": "invalidated", "allowed": false
  0        <- código de salida
```

Ahí está: **el veredicto dice "inválido" y el código de salida dice "todo
bien".** Mi hook —como casi cualquier hook— confiaba en el código de salida.

La prueba definitiva: un commit **sin ninguna revisión**, en un candidato que
la herramienta acababa de declarar inválido... pasó la compuerta sin ruido.

Siete commits de "enforcement" esa tarde. Ceremonia real, candado de teatro.

## Por qué esto es peor que un bug normal

Un guard que falla **cerrado** molesta: bloquea trabajo legítimo, alguien se
queja en una hora, se arregla. Un guard que falla **abierto** no molesta a
nadie — y por eso puede vivir meses. El sistema se ve más verde que nunca
justo cuando nada está siendo verificado. El silencio del control apagado es
idéntico al silencio del control satisfecho, y nadie audita lo que está en
verde.

El otro usuario y yo teníamos el mismo defecto con síntomas opuestos porque
leíamos canales distintos del mismo programa: él el JSON (todo bloqueado), yo
el código de salida (todo permitido). Publiqué la medición en el
[issue](https://github.com/Gentleman-Programming/gentle-ai/issues/3939) y las
piezas de tres reportes independientes encajaron.

## El arreglo, en dos capas

1. **Volví a la versión anterior** de la herramienta — la única donde el ciclo
   cierra completo y el candado responde en ambas direcciones. Guardar el
   binario viejo antes de actualizar costó un comando al mediodía y salvó la
   noche.
2. **El hook quedó más duro que antes**: ahora lee el veredicto del JSON
   (`"allowed": true`) con sus propios ojos, en vez de confiar en el código de
   salida de otro proceso. Probado en las dos direcciones: sin revisión
   bloquea, con revisión y constancia pasa.

## La lección que me llevo

**Lo que verifica también se verifica.** Tengo esa frase escrita como
principio desde hace semanas, y aun así el verificador de mis commits operó
medio día sin que yo verificara que verificaba. No lo cazó la atención — lo
cazó una contradicción externa y un experimento de cuatro comandos.

La moraleja práctica, si tienes cualquier control automático del que dependes:

- Pruébalo **en las dos direcciones**: que deje pasar lo bueno es la mitad;
  que bloquee lo malo es la mitad que nadie prueba.
- Cada tanto, **dale un caso malo a propósito** y mira si grita. Un control
  que nunca has visto fallar no es un control probado — es un control con
  suerte.
- Y cuando dos personas reportan síntomas opuestos del mismo sistema, no
  suele ser que una se equivoque: suele ser que el sistema tiene dos caras y
  cada quien mira una.

---

*Esta bitácora deja una entrada por día. Lo que no se midió, se declara. Mañana:
tres relojes se estrenan a las 06:30, 07:30 y 08:00 — un sistema que debe
avisar, repararse y dejar constancia sin que nadie se lo pida.*
