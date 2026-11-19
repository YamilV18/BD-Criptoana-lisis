# Equipo y aporte individual

| # | Integrante | Qué demuestra | Evidencia |
|---|---|---|---|
| 1 | Yamil Valente Quispe Cuellar | Monto: pipeline U1, modelo en vivo y declaración de que la serie no supera a la persistencia | [Bitácora 1](bitacoras/integrante1.md) |
| 2 | Condori Quispe Brayan Raúl | Ballenas: RandomForest ponderado por AUC-PR y límites del recall en vivo | [Bitácora 2](bitacoras/integrante2.md) |
| 3 | Juli Machaca Renzo Rodi | Rol: resultado negativo defendido y serie que mejora a la persistencia | [Bitácora 3](bitacoras/integrante3.md) |
| 4 | Coaquira Puma Raquel Kantu | Cantidad: regresión sin regularizar y serie con muestra pequeña | [Bitácora 4](bitacoras/integrante4.md) |

Común al grupo: arquitectura, DataOps, hardening, demo y repositorio.

## Evidencia de autoría

!!! info "Sobre los commits"
    Por la forma en que trabajó el equipo, los commits del repositorio se
    subieron desde una sola cuenta (`YamilV18`), que actuó como integrante
    encargado del repositorio. Eso **no refleja quién escribió cada parte**.
    La autoría individual se verifica con las evidencias de la tabla.

| # | Integrante | Archivos propios | Evidencia verificable |
|---|---|---|---|
| 1 | Yamil Valente Quispe Cuellar | `notebooks/U{1,2,3}_Integrante1_*.ipynb` | [Bitácora 1](bitacoras/integrante1.md), manifiestos en `artifacts/integrante1/u3/`, panel `instance=integrante1` en Grafana |
| 2 | Condori Quispe Brayan Raúl | `notebooks/U{1,2,3}_Integrante2_*.ipynb` | [Bitácora 2](bitacoras/integrante2.md), manifiestos en `artifacts/integrante2/u3/`, panel `instance=integrante2` |
| 3 | Juli Machaca Renzo Rodi | `notebooks/U{1,2,3}_Integrante3_*.ipynb` | [Bitácora 3](bitacoras/integrante3.md), manifiestos en `artifacts/integrante3/u3/`, panel `instance=integrante3` |
| 4 | Coaquira Puma Raquel Kantu | `notebooks/U{1,2,3}_Integrante4_*.ipynb` | [Bitácora 4](bitacoras/integrante4.md), manifiestos en `artifacts/integrante4/u3/`, panel `instance=integrante4` |

Cada notebook corre en su propia instancia de métricas (`instance=integranteN`),
y cada manifiesto lleva el `runId` de la ejecución de ese integrante.