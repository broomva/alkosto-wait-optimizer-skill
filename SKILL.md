---
name: alkosto-wait-optimizer
description: "Estimar tiempo de espera optimo para la promocion de Alkosto (cliente 25 o 50) con dos enfoques: flujo de compras observado en cajas o timestamps de anuncios de ganadores. Usar cuando se necesite decidir si conviene esperar, calcular tiempo probable al proximo ganador, o fijar un cutoff de espera."
---

# Alkosto Wait Optimizer

Aplicar un calculo operativo (no oficial) para estimar espera y definir un tiempo maximo razonable.

## Recopilar datos

Elegir uno de dos metodos:

1. `purchase_rate` (flujo de caja)
- Contar compras cerradas en una ventana de 2 minutos.
- Registrar cajas observadas.
- Registrar cajas abiertas totales (si es visible).
- Definir dia: entre semana (`K=25`) o fin de semana/festivo (`K=50`).

2. `winner_timestamps` (solo anuncios)
- Guardar 4-8 timestamps de ganadores en orden.
- Medir minutos transcurridos desde el ultimo anuncio.

## Calculo: purchase_rate

Variables:
- `lambda_obs = compras_observadas / minutos_observados`
- Modelo `global`: `lambda_total = lambda_obs * (cajas_totales / cajas_observadas)`
- Modelo `per_lane`: `lambda_total = lambda_obs / cajas_observadas`
- Ajuste conservador: `lambda_cons = lambda_total * (1 - buffer)`
- Intervalo entre ganadores: `T = K / lambda_cons`

Derivados:
- Espera media al proximo ganador: `T/2`
- Regla practica de espera: esperar `W = min(T * prob_objetivo, max_wait)`
- Si no ocurre ganador antes de `W`: re-medir 2 minutos y recalcular.

## Calculo: winner_timestamps

1. Calcular intervalos `delta_i = t_i - t_(i-1)`.
2. Calcular:
- `mu = promedio(delta_i)`
- `sigma = desviacion estandar(delta_i)`
- `CV = sigma / mu`
3. Seleccionar modelo:
- `CV < 0.4`: comportamiento regular
- `0.4 <= CV <= 0.7`: mixto
- `CV > 0.7`: aleatorio
4. Espera sugerida:
- Regular: `W ~= max(mu - elapsed, 0)`
- Aleatorio (exponencial): `W = -mu * ln(1 - p_objetivo)`
- Mixto: promedio de ambas
- Siempre aplicar tope `max_wait`

## Decision rule

Usar esta regla final:
- Si en `W` minutos no sale ganador, no seguir esperando a ciegas.
- Tomar nuevos datos (2 minutos de cajas o 2-3 anuncios adicionales) y recalcular.
- Si el nuevo `W` sigue alto frente al valor de tu tiempo, cortar espera.

## Economia opcional

Si se conoce:
- `valor_bono_esperado`
- `valor_tiempo_por_minuto`

Calcular:
- `EV(W) = P(evento_en_W) * valor_bono_esperado`
- `Costo(W) = W * valor_tiempo_por_minuto`
- `Neto = EV(W) - Costo(W)`

No recomendar espera larga cuando `EV/min < costo/min`.
