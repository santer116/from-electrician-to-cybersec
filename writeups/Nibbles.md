# HTB / CTF Write-up

**Autor:** Santiago Fonseca  
**Máquina:** Niblles  
**IP (objetivo):** 10.129.200.170

**Fecha:** 10-10-2025  

---

## Objetivo
 * Gain a foothold on the target and submit the user.txt flag
 * Escalate privileges and submit the root.txt flag.

---

## Pasos a seguir

 1) Reconocimiento inicial

Comence realizando un primer escaneo para conocer que puertos estaban abiertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.129.200.170 -oN puertos
```
* -p- Incluye todos los puertos TCP (1–65535).
* --open Solo muestra los puertos abiertos.
* -sS SYN scan (half-open): envía SYN y observa respuestas (SYN/ACK → abierto; RST → cerrado).
  Es rápido y evita completar conexiones completas con el objetivo. Requiere privilegios de root.
* --min-rate 5000 intenta enviar al menos 5000 paquetes por segundo (acelera mucho el escaneo).
* -n Acelera el escaneo y evita ruido de consultas DNS. No aplica resolucion DNS.
* -Pn No hacer ping previo; trata al host como “up”. Útil si el objetivo bloquea pings/ICMP o responde mal a sondas.
* -oN Guarda la salida en formato legible (puertos) para evidencias y posterior análisis.
