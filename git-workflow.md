# Flujo de git del equipo — este documento se dividió en dos

Lo que estaba acá creció y se partió, porque las dos mitades envejecen a
velocidades distintas:

- **[`manual-git.md`](./manual-git.md)** — el manual práctico. Genérico, vale
  para cualquier repo del equipo. **Es el que se lee para trabajar.**
- **[`decisiones-git.md`](./decisiones-git.md)** — el registro de por qué cada
  regla es como es: qué incidente la obligó, qué se descartó y con qué
  evidencia. Append-only. **Es el que se lee cuando una regla parece burocracia.**

Este archivo queda como puntero porque hay enlaces apuntándole.

---

## Lo único que no está en el manual, porque es específico de estos dos repos

⚠️ **La troncal NO se llama igual en los dos** (verificado contra los remotos):

| Repo | Troncal |
|---|---|
| [`jikaidoko/midnight-hackathon-ba`](https://github.com/jikaidoko/midnight-hackathon-ba) — donde se codea | **`main`** |
| `jikaidoko/hackmid` / `amparo-prep` — esta carpeta, referencia del equipo | **`master`** |

Si no te acordás cuál es, no adivines. El remoto lo dice:

```bash
git branch -r
```
