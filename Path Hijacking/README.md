# Linux Privilege Escalation via SUID + PATH Hijacking

## El objetivo de este laboratorio es demostrar cómo un binario SUID mal implementado puede ser explotado mediante PATH hijacking para obtener privilegios de root.


### 🎯 Objetivo
► Construir un binario mal configurado
► Escalar privilegios desde un usuario sin privilegios.
► Analizar un binario SUID.
► Explotar PATH hijacking.


## 🧱 Entorno de laboratorio
► SO: Linux
► Usario sin privilegios: userone
► Binario vulnerable: /opt/statuscheck

## 🔍 Análisis del binario
► Se analiza los archivos desde la raiz de algun archivo con permiso SUID
<p style="color:#00ff00;">
► find / -perm -4000 -ls 2&gt; /dev/null
</p>
<p>
Busca desde la raíz archivos SUID (4000), los lista y redirecciona errores a /dev/null
</p>