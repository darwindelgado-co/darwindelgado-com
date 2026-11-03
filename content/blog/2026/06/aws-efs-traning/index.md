---
date: '2026-06-12T09:00:00-05:00'
tags: ['aws', 'cloud', 'efs', 'multi-az', 'arquitectura', 'nfs']
title: 'Arquitectura EFS Multi-AZ: Almacenamiento compartido de alta disponibilidad en AWS'
slug: 'Elastic-File-System'
---

Si tienes dos instancias EC2 en distintas Availability Zones y necesitas que ambas lean y escriban los mismos archivos en tiempo real (sin usar Rsync ni copiar carpetas a mano), la respuesta es Amazon EFS. En esta guía **desplegaré** un sistema de archivos NFS administrado por AWS y te **mostraré** paso a paso cómo montarlo en ambas instancias para lograr una arquitectura Multi-AZ altamente disponible.

<!--more-->

<!-- DIAGRAMA PRINCIPAL: Ocupa el 100% del ancho, con bordes redondeados y sombra -->
<img src="./efs-storage.png" alt="Diagrama de arquitectura EFS Multi-AZ conectando dos EC2 a través de Mount Targets" style="width: 100%; max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); margin-top: 1rem; margin-bottom: 2rem;">

-----

## 1. Creo el filesystem

Primero voy a la consola de EFS → Create file system → Name: `training-efs`, VPC: `training-vpc`.

En las opciones de personalización elijo: **Storage class** Standard, **Throughput** Elastic (así pago por lo que uso, ideal para cargas de trabajo variables) y activo la **encriptación at-rest**.

En **File system policy**, marco la opción **"Enforce in-transit encryption for all clients"**. Esto es crucial: obliga a que todo el tráfico NFS viaje cifrado con TLS. Sin esto, los datos viajarían en texto plano por la red.

Así queda el filesystem una vez creado:

{{< figure
  src="./aws-efs.png"
  alt="Pantalla de detalle del filesystem training-efs"
  caption="Detalle del filesystem training-efs en la consola de AWS"
  class="screenshot-img"
>}}

## 2. Security Group y Arquitectura de Red (Multi-AZ)

El mount target necesita recibir tráfico NFS en el puerto 2049 desde mis EC2.

Voy a EC2 → Security Groups → Create security group. Name: `training-sg-efs`.
En la regla de entrada (Inbound rule) selecciono NFS, puerto 2049, y como source pongo **el Security Group de mis EC2**. *(Buena práctica: nunca abrir el puerto a `0.0.0.0/0`)*.

Vuelvo a EFS → mi filesystem → Network → edito los mount targets → asigno `training-sg-efs`.

> **Nota de Arquitectura:** Como tengo instancias en dos AZs distintas, AWS crea automáticamente un **Mount Target en cada AZ**. Esto es vital por dos razones: **Alta Disponibilidad** (si una AZ falla, la otra sigue accediendo) y **Bajo Latencia** (el tráfico no cruza entre zonas, evitando costos de transferencia y latencia).

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  caption="Reglas inbound del security group permitiendo el puerto NFS (2049)"
  class="screenshot-img"
>}}

## 3. Lanzo dos EC2 en AZ diferentes

Despliego dos instancias EC2 (una en la AZ-a y otra en la AZ-b) dentro de subredes privadas. A ambas les asigno el IAM Role `AmazonSSMManagedInstanceCore` para que pueda acceder a ellas de forma segura vía SSM Session Manager, sin necesidad de abrir el puerto 22. Uso Amazon Linux 2 como sistema operativo.

## 4. Monto el filesystem en ambas

Me conecto a cada EC2 vía SSM Session Manager y ejecuto:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
