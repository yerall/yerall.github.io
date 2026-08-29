---
title: Quiz Game en Python
date: 2026-08-29
categories: [Tools, Python]
tags: [Python, Github]
---

Este es un pequeño juego de preguntas y respuestas (quiz) hecho en Python. El programa lee las preguntas desde un archivo de texto externo, se las muestra al usuario, recoge sus respuestas y al final entrega un resumen con los aciertos, los errores y el detalle de las preguntas falladas.

El código fuente se encuentra aquí [Enlace-Repositorio](https://github.com/yerall/quiz-game-python)

## **Descripción**

El programa lee las preguntas desde un archivo llamado `preguntas.txt`, muestra cada pregunta al usuario y solicita una respuesta.

Al finalizar el cuestionario, muestra:

- Número de respuestas correctas.
- Número de respuestas incorrectas.
- Lista de las preguntas respondidas incorrectamente.

## **Requisitos**

- Python 3
- Un archivo `preguntas.txt` con las preguntas y respuestas, ubicado en la misma carpeta que el script.

## **Ejemplo de uso**

Clona el repositorio y ejecuta el programa con:

```bash
git clone https://github.com/yerall/quiz-game-python
cd quiz-game-python
python quiz.py
```

Cada pregunta en `preguntas.txt` debe tener el siguiente formato, separada de las demás por una línea en blanco:

```
¿Cuál es el océano más grande del mundo?
a) Atlántico
b) Índico
c) Pacífico
d) Ártico
RESPUESTA:c
```

Durante la ejecución, el programa muestra una pregunta a la vez:

```
¿Cuál es el planeta más grande del Sistema Solar?
a) Tierra
b) Marte
c) Júpiter
d) Saturno
Respuesta: c
Correcto

Presiona ENTER para continuar...
```

Y al terminar todas las preguntas, muestra el resumen final:

```
===== RESULTADOS =====
Correctas: 13
Incorrectas: 2

Preguntas incorrectas:
- ¿Cuántos minutos tiene una hora?
- ¿Qué instrumento se utiliza para medir la temperatura?
```
