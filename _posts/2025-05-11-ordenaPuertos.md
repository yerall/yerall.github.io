---
title: Escáner de Puertos y Ordenamiento con Nmap
date: 2025-05-11
categories: [Tools, Bash]
tags: [Tools, Bash, Github]
image: /assets/img/commons/ordenaPuertos/banner.png
---

¡Saludos!

Este script en Bash realiza un escaneo simple en busca de puertos en un objetivo dado. Proporciona los puertos abiertos del objetivo especifico de forma ordenada para su uso en posteriores procesos. Este script es bastante simple pero su utilidad es muy grande en la medida de lo posible al eliminar ese tiempo de acomodar la gran cantidad de puertos abiertos que se pueden encontrar en un objetivo.

Es código fuente se encuentra aquí [Enlace-Script](https://github.com/yerall/ordena_puertos)


## **Características**

- Realizar un escaneo con Nmap `nmap -p- --open -n -sS -Pn --min-rate 6000 "$1"`

Donde:

- `nmap`: Es el comando principal de la herramienta Nmap.
- `-p-`: Esta opción indica que escanee todos los puertos, desde el 1 hasta 65535.
- `-open`: Esta opción hace que solo los puertos que están abiertos.
- `n`: Esta opción indica que no realice la resolución de DNS durante el escaneo. Esto significa que no intentará convertir las direcciones IP en nombres de host.
- `sS`: Esta opción especifica el tipo de escaneo a realizar. En este caso, es un escaneo de tipo SYN.
- `Pn`: Esta opción indica que no realice el descubrimiento de hosts. Normalmente, intenta determinar qué hosts están activos en la red antes de escanearlos, pero esta opción deshabilita esa función y escanea el host especificado sin hacer ninguna verificación previa.
- `-min-rate 6000`: Esta opción establece la tasa mínima de paquetes que enviará por segundo. En este caso, se establece en 6000 paquetes por segundo. Ajustar esta tasa puede ayudar a controlar la velocidad del escaneo y evitar que la red se vea abrumada por el tráfico generado por Nmap.
- `"$1"`: Se reemplazará con el primer argumento proporcionado cuando se ejecute el comando.

> El comando puede ser editado a preferencia pero con la recomendación de siempre verificar que funcione correctamente para evitar escaneos fallidos
{: .prompt-tip }

- Toma los puertos abiertos del objetivo especifico para ordenarlos para su uso en posteriores procesos.

## **Uso**

```bash
git clone <Enlace>
cd ordena_puertos
chmod +x ordenapuertos.sh
sudo ./ordenapuertos.sh <objetivo>
```

## **Ejemplo de salida**

```bash
sudo ./ordenapuertos.sh 192.168.100.18
--------------------------------------------------
// PUERTOS ORDENADOS //
--------------------------------------------------
21,22,23,25,53,80,111,139,445,512,513,514,1099,1524,2049,2121,3306,3632,5432,5900,6000,6667,6697,8009,8180,8787,36273,47693,56699,59046
--------------------------------------------------
```
