# 🤖 DeliveryBot - Sistema Automatizado de Pedidos y Reportes

DeliveryBot es un bot de Telegram automatizado mediante **n8n**, diseñado para gestionar, registrar y analizar el historial de pedidos de un negocio de comida o delivery (como *Delicias del Ayer*), conectándose de manera fluida con Google Sheets y procesando analíticas en tiempo real.

---

## 🚀 Características Principales

* **Recepción por Telegram:** Captura interacciones y comandos directos de los usuarios (ej. `/reporte`).
* **Sincronización con Google Sheets:** Almacena automáticamente cada pedido en una hoja de cálculo centralizada.
* **Procesamiento de Datos en JavaScript:** Un nodo de código dedicado analiza y calcula métricas clave sin depender de servicios de IA externos lentos.
* **Métricas Avanzadas de Negocio:**
  * Conteo total de pedidos activos (excluyendo cancelados).
  * Cálculo de ingresos totales generados.
  * Top 5 de productos más vendidos con sus respectivas cantidades.
  * Horas de mayor afluencia (horas pico) de pedidos.

---

## 🛠️ Stack Tecnológico

* **Automatización:** n8n (Self-hosted o Cloud)
* **Base de Datos / Almacenamiento:** Google Sheets
* **Interfaz de Usuario:** Telegram Bot API
* **Lógica y Procesamiento:** JavaScript (NodeJS)

---

## 📦 Estructura del Flujo en n8n

1. **Telegram Trigger:** Escucha los comandos entrantes (ej. `/reporte`).
2. **Google Sheets (Get Rows):** Extrae el historial completo desde la hoja de cálculo `PEDIDOS`.
3. **Nodo de Código (Analizar Datos y Generar Reporte):** Script en JavaScript encargado de filtrar, ordenar, sumar ingresos y estructurar el texto final del reporte.
4. **Telegram (Send Message):** Envía el mensaje formateado de vuelta al administrador o chat correspondiente.

---

## 💻 Script de Análisis (JavaScript)

El núcleo del procesamiento de datos utiliza el siguiente script dentro del nodo de n8n:

```javascript
// 1. Obtenemos todas las filas del nodo de Google Sheets
const pedidos = $('Get row(s) in sheet').all().map(item => item.json);

if (pedidos.length === 0) {
    return [{ json: { reporte_texto: "⚠️ No hay pedidos registrados todavía." } }];
}

let totalIngresos = 0;
let ventasPorProducto = {};
let ventasPorHora = {};
let totalPedidos = 0;

// 2. Analizamos pedido por pedido
pedidos.forEach(p => {
    if (p.estado !== "Cancelado") {
        totalPedidos++;
        
        totalIngresos += parseFloat(p.total_pago || 0);

        if (p.hora) {
            let horaCorta = String(p.hora).split(':')[0] + ":00";
            ventasPorHora[horaCorta] = (ventasPorHora[horaCorta] || 0) + 1;
        }

        if (p.detalles_pedido) {
            const lineas = p.detalles_pedido.split('\n');
            lineas.forEach(linea => {
                const match = linea.match(/•\s*(\d+)x\s*(.+?)\s*\(\$/);
                if (match) {
                    const cantidad = parseInt(match[1], 10);
                    const nombreProducto = match[2].trim();
                    ventasPorProducto[nombreProducto] = (ventasPorProducto[nombreProducto] || 0) + cantidad;
                }
            });
        }
    }
});

// 3. Ordenamos Top Productos
const arrayTopProductos = Object.entries(ventasPorProducto)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5)
    .map(prod => ({
        producto: prod[0],
        cantidad_vendida: prod[1]
    }));

// 4. Ordenamos Horas Pico
const arrayTopHoras = Object.entries(ventasPorHora)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 3)
    .map(hora => ({
        hora: hora[0],
        cantidad_pedidos: hora[1]
    }));

// 5. Formateamos los textos para Telegram
const topProductosTexto = arrayTopProductos.map(p => `🍔 ${p.cantidad_vendida} uds - ${p.producto}`).join('\n');
const topHorasTexto = arrayTopHoras.map(h => `⏰ ${h.hora} (${h.cantidad_pedidos} pedidos)`).join('\n');

const mensajeReporte = `📊 *REPORTE DE VENTAS DELIVERYBOT*\n\n` +
                        `📦 *Total de Pedidos:* ${totalPedidos}\n` +
                        `💰 *Ingresos Totales:* $${totalIngresos.toFixed(2)}\n\n` +
                        `🏆 *TOP 5 - Productos más vendidos:*\n${topProductosTexto || "Sin datos"}\n\n` +
                        `🔥 *Horas de mayor afluencia:*\n${topHorasTexto || "Sin datos"}`;

// 6. Retornamos el JSON estructurado
return [{
    json: {
        total_pedidos: totalPedidos,
        ingresos_totales: totalIngresos,
        top_productos: arrayTopProductos,
        horas_pico: arrayTopHoras,
        reporte_texto: mensajeReporte
    }
}];