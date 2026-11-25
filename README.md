# 🧪 CTF Write-ups — The Hacker Labs

Colección de informes de resolución de máquinas CTF realizadas en **laboratorio propio y autorizado**, con fines de aprendizaje de pentesting. El foco está en la **metodología y el razonamiento** de cada fase, no en las soluciones rápidas.

🌐 **Sitio web (informes con diseño):** `https://leinadfh.github.io/thehakerslabs-writeups/`

> ⚠️ **Sin spoilers:** estos write-ups **no incluyen las flags** y evitan revelar valores sensibles, para no estropear el reto a otras personas. Se documenta *cómo se piensa*, no *qué hay que copiar*.

---

## 📋 Máquinas

| Máquina | Plataforma | SO | Dificultad | Técnicas clave | Write-up |
|---|---|---|---|---|---|
| **Academy** | The Hacker Labs | Linux (Debian) | Fácil–Media | WordPress · XML-RPC brute force · RCE autenticado · privesc por cron | [Markdown](./Academy/) · [HTML](./docs/academy/) |

*(Se irán añadiendo máquinas conforme las resuelva.)*

> El write-up vive en **dos formatos**: Markdown (`Academy/README.md`, se lee directo en GitHub) y HTML con diseño (`docs/academy/`, se publica vía GitHub Pages).

## 🌐 Activar GitHub Pages

1. Sube el repo a GitHub.
2. **Settings → Pages → Build and deployment → Source: _Deploy from a branch_**.
3. Branch: `main`, carpeta: **`/docs`**. Guarda.
4. En ~1 min estará en `https://leinadfh.github.io/thehakerslabs-writeups/`.

---

## 🧰 Stack habitual

`nmap` · `gobuster`/`feroxbuster` · `wpscan` · `burp` · `curl` · `pspy64` · `hashcat`/`john` · `linpeas` · Kali Linux

## 🗺️ Metodología

1. **Descubrimiento** de hosts en la red.
2. **Enumeración** de puertos y servicios.
3. **Enumeración de aplicación** (web, versiones, usuarios).
4. **Acceso inicial** (foothold).
5. **Escalada de privilegios** a root/admin.
6. **Post-explotación** y documentación.

---

## ⚖️ Aviso legal

Todo el contenido es **educativo**. Las pruebas se realizan exclusivamente sobre:
- Máquinas de CTF descargadas y ejecutadas en un **homelab aislado propio**, y/o
- Plataformas donde la publicación de write-ups está **expresamente permitida** (máquinas públicas/retiradas).

No se incluye información de ningún sistema, cliente o engagement real. Atacar sistemas sin autorización explícita es ilegal.

## 📄 Licencia

[MIT](./LICENSE) — siéntete libre de aprender del material citando la fuente.
