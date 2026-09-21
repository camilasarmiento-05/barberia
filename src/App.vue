<template>
  <div class="pagina">
    <div class="marca">
      <h1>Barbería Don Ramiro</h1>
      <p>Registro de servicios</p>
    </div>

    <button class="btn-nuevo" @click="abrirModalNuevo">Registrar servicio</button>
    <div class="fila-botones-top">
      <button class="btn-secundario" @click="abrirCatalogo">Catálogo de servicios</button>
      <button class="btn-secundario" @click="modalArchivadosAbierto = true">
        Archivados ({{ serviciosArchivados().length }})
      </button>
    </div>

    <div class="caja-resumen">
      <div class="linea-resumen">
        <span>Servicios activos (sin archivar)</span>
        <span>{{ serviciosActivos().length }}</span>
      </div>
      <div class="linea-resumen verde">
        <span>Recaudado hoy</span>
        <span>{{ formatearPrecio(totalVendidoHoy()) }}</span>
      </div>
      <div class="linea-resumen verde">
        <span>Recaudado en cierres anteriores</span>
        <span>{{ formatearPrecio(totalArchivadoAnterior()) }}</span>
      </div>
      <div class="linea-resumen verde" v-if="totalActivoAnterior() > 0">
        <span>Días anteriores sin cerrar caja</span>
        <span>{{ formatearPrecio(totalActivoAnterior()) }}</span>
      </div>
      <div class="linea-resumen verde total-general">
        <span>Total recaudado general</span>
        <span>{{ formatearPrecio(totalGeneralRecaudado()) }}</span>
      </div>
      <div class="linea-resumen rojo" v-if="totalPendientesCantidad() > 0">
        <span>Con saldo pendiente ({{ totalPendientesCantidad() }})</span>
        <span>{{ formatearPrecio(totalPendientes()) }}</span>
      </div>
    </div>

    <div class="caja-estadisticas">
      <h3>Estadísticas de hoy</h3>
      <div class="linea-resumen">
        <span>Total vendido hoy</span>
        <span>{{ formatearPrecio(totalVendidoHoy()) }}</span>
      </div>
      <div class="linea-resumen">
        <span>Servicios de hoy</span>
        <span>{{ serviciosHoy().length }}</span>
      </div>
      <div class="linea-resumen">
        <span>Promedio de calificación</span>
        <span>{{ promedioCalificacionHoy() > 0 ? promedioCalificacionHoy() + ' ★' : 'Sin datos' }}</span>
      </div>
      <div class="linea-resumen">
        <span>Barbero con más cortes hoy</span>
        <span>{{ barberoConMasCortesHoy() }}</span>
      </div>
    </div>

    <div class="caja-historial-cliente">
      <label>Buscar historial de cliente</label>
      <input type="text" v-model="busquedaCliente" placeholder="Escribe el nombre del cliente...">
      <div v-if="historialCliente()" class="resultado-historial">
        <p>
          <strong>{{ historialCliente().nombre }}</strong> ha venido
          <strong>{{ historialCliente().visitas }}</strong>
          {{ historialCliente().visitas === 1 ? 'vez' : 'veces' }}
        </p>
        <p>Total gastado: <strong>{{ formatearPrecio(historialCliente().totalGastado) }}</strong></p>
      </div>
    </div>

    <div class="caja-deudas" v-if="deudasPorCliente().length > 0">
      <h3>⚠ Saldos pendientes por cliente</h3>
      <div class="linea-deuda" v-for="d in deudasPorCliente()" :key="d.cliente">
        <span>{{ d.cliente }}</span>
        <span>{{ formatearPrecio(d.total) }}</span>
      </div>
    </div>

    <div class="filtros">
      <select v-model="filtroBarbero">
        <option value="Todos">Todos los barberos</option>
        <option v-for="b in barberos" :key="b" :value="b">{{ b }}</option>
      </select>
      <select v-model="filtroEstado">
        <option value="Todos">Todos los estados</option>
        <option v-for="e in estadosPago" :key="e" :value="e">{{ e }}</option>
      </select>
    </div>

    <div class="orden-botones">
      <span class="etiqueta">Ordenar por</span>
      <button
        v-for="opt in opcionesOrden"
        :key="opt.key"
        type="button"
        class="btn-orden"
        :class="{ activo: ordenarPor === opt.key }"
        @click="cambiarOrden(opt.key)"
      >{{ opt.label }} <span v-if="ordenarPor === opt.key">{{ ordenDireccion === 'asc' ? '↑' : '↓' }}</span></button>
    </div>

    <p class="vacio" v-if="!cargando && serviciosFiltrados().length === 0">No hay servicios para mostrar</p>

    <div class="turnos-contenedor">
      <template v-for="turno in ['Mañana', 'Tarde', 'Noche']" :key="turno">
        <div class="turno-grupo" v-if="serviciosPorTurno()[turno].length > 0">
          <div class="turno-titulo">{{ turno }} ({{ serviciosPorTurno()[turno].length }})</div>
          <div class="lista-tickets">
            <div class="ticket" v-for="s in serviciosPorTurno()[turno]" :key="s.id">
              <span class="estado" :class="claseEstado(s.estadoPago)">{{ s.estadoPago }}</span>

              <div class="ticket-titulo">{{ s.cliente }}</div>
              <div class="separador"></div>

              <div class="servicios-bloque">
                <span class="etiqueta">Servicios</span>
                <div class="servicio-item" v-for="(nombre, idx) in s.tiposServicio" :key="idx">{{ nombre }}</div>
              </div>
              <div class="separador"></div>

              <div class="ticket-fila"><span>Barbero</span><span>{{ s.barbero }}</span></div>

              <div class="fecha-hora">
                <div class="caja-fecha">
                  <span class="etiqueta">Fecha</span>
                  <span>{{ formatearFecha(s.fecha) }}</span>
                </div>
                <div class="caja-fecha">
                  <span class="etiqueta">Hora</span>
                  <span>{{ s.hora }}</span>
                </div>
              </div>

              <div class="ticket-fila">
                <span>Pago</span>
                <span>{{ s.metodoPago }}</span>
              </div>

              <div class="separador"></div>
              <div class="ticket-fila ticket-total">
                <span>Total</span>
                <span v-if="s.propina > 0">{{ formatearPrecio(s.precio) }} + {{ formatearPrecio(s.propina) }} propina</span>
                <span v-else>{{ formatearPrecio(s.precio) }}</span>
              </div>
              <div class="ticket-fila fila-abonado" v-if="s.montoAbonado > 0 && s.montoAbonado < precioTotal(s)">
                <span>Abonado</span><span>{{ formatearPrecio(s.montoAbonado) }}</span>
              </div>
              <div class="ticket-fila fila-saldo" v-if="saldoRestante(s) > 0">
                <span>Saldo pendiente</span><span>{{ formatearPrecio(saldoRestante(s)) }}</span>
              </div>

              <div class="historial-abonos" v-if="s.historialAbonos && s.historialAbonos.length > 0">
                <span class="etiqueta">Historial de abonos</span>
                <div class="linea-abono" v-for="(a, idx) in s.historialAbonos" :key="idx">
                  <span>{{ formatearFecha(a.fecha) }}</span>
                  <span>{{ formatearPrecio(a.monto) }}</span>
                </div>
              </div>

              <button
                class="btn-mini abonar"
                v-if="saldoRestante(s) > 0"
                @click="abrirModalAbono(s)"
              >Agregar abono</button>

              <div class="fotos-antes-despues" v-if="s.fotoAntes || s.fotoDespues">
                <div class="foto-item" v-if="s.fotoAntes">
                  <span class="etiqueta">Antes</span>
                  <img :src="s.fotoAntes" class="foto-mini">
                </div>
                <div class="foto-item" v-if="s.fotoDespues">
                  <span class="etiqueta">Después</span>
                  <img :src="s.fotoDespues" class="foto-mini">
                </div>
              </div>

              <div class="separador"></div>

              <div class="calificacion-seccion">
                <span class="etiqueta">Calificación</span>
                <div class="estrellas" :class="{ bloqueada: s.estadoPago !== 'pagado' }">
                  <span
                    v-for="n in [1,2,3,4,5]"
                    :key="n"
                    class="estrella"
                    :class="{ llena: n <= s.calificacion, baja: n <= s.calificacion && s.calificacion <= 2 }"
                    @click="calificarServicio(s, n)"
                  >★</span>
                </div>
                <p class="aviso-baja" v-if="s.estadoPago !== 'pagado'">Debes terminar de pagar para poder calificar</p>
                <p class="aviso-baja" v-else-if="s.calificacion > 0 && s.calificacion <= 2">Cliente insatisfecho, revisar servicio</p>
              </div>

              <label class="etiqueta">Observaciones</label>
              <textarea
                class="obs-input"
                v-model="s.observaciones"
                placeholder="Escribe una observación..."
              ></textarea>

              <p class="aviso-baja" v-if="citaYaPaso(s)">Esta cita ya pasó, no se puede editar ni eliminar.</p>

              <div class="ticket-acciones">
                <button
                  class="btn-mini editar"
                  @click="abrirModalEditar(s)"
                  :disabled="s.calificacion > 0 || citaYaPaso(s)"
                  :title="s.calificacion > 0 ? 'No se puede editar un servicio ya calificado' : (citaYaPaso(s) ? 'No se puede editar una cita que ya pasó' : '')"
                >Editar</button>
                <button
                  class="btn-mini eliminar"
                  @click="pedirConfirmacionEliminar(s)"
                  :disabled="citaYaPaso(s)"
                  :title="citaYaPaso(s) ? 'No se puede eliminar una cita que ya pasó' : ''"
                >Eliminar</button>
              </div>
            </div>
          </div>
        </div>
      </template>
    </div>

    <div class="caja-comisiones">
      <div class="comisiones-header">
        <h3>Comisiones de hoy</h3>
        <button class="btn-mini cerrar-caja" @click="abrirCierreCaja">Cerrar caja</button>
      </div>
      <div class="linea-comision" v-for="b in barberos" :key="b">
        <span class="comision-nombre">{{ b }}</span>
        <span class="comision-pct"><input type="number" v-model.number="comisiones[b]" min="0" max="100">%</span>
        <span class="comision-monto">{{ formatearPrecio(comisionBarbero(b)) }}</span>
      </div>
    </div>

    <div class="fondo-modal" v-if="modalAbierto">
      <div class="modal">
        <h2>{{ modoEdicion ? 'Editar servicio' : 'Nuevo servicio' }}</h2>

        <form @submit.prevent="guardarServicio">
          <label>Cliente</label>
          <input type="text" v-model="formulario.cliente">
          <p class="error" v-if="errores.cliente">{{ errores.cliente }}</p>
          <p class="alerta-fidelidad" v-if="clienteEsFrecuente()">¡Cliente frecuente, aplica 10% de descuento!</p>

          <label>Tipo de servicio</label>
          <div class="lista-checks">
            <label class="check-item" v-for="t in catalogoServicios" :key="t.nombre">
              <input
                type="checkbox"
                :value="t.nombre"
                v-model="formulario.tiposServicio"
                @change="alCambiarServicio(t.nombre)"
              >
              <span>{{ t.nombre }} ({{ formatearPrecio(t.precio) }})</span>
            </label>
          </div>
          <p class="pista-precio"></p>
          <p class="error" v-if="errores.tipoServicio">{{ errores.tipoServicio }}</p>

          <label>Barbero</label>
          <select v-model="formulario.barbero">
            <option value="" disabled>Selecciona...</option>
            <option v-for="b in barberos" :key="b" :value="b">{{ b }}</option>
          </select>
          <p class="error" v-if="errores.barbero">{{ errores.barbero }}</p>

          <div class="fila-doble">
            <div>
              <label>Fecha</label>
              <input type="date" v-model="formulario.fecha" :min="fechaMinima" @change="alCambiarFecha">
              <p class="error" v-if="errores.fecha">{{ errores.fecha }}</p>
            </div>
            <div>
              <label>Hora</label>
              <select v-model="formulario.hora">
                <option value="" disabled>Selecciona...</option>
                <option v-for="h in horasDisponiblesFormulario" :key="h" :value="h">{{ h }}</option>
              </select>
              <p class="pista-precio" v-if="formulario.fecha === fechaHoyString()"></p>
              <p class="error" v-if="errores.hora">{{ errores.hora }}</p>
            </div>
          </div>

          <label>Precio</label>
          <input
            type="text"
            class="precio-bloqueado"
            :value="formatearPrecio(formulario.precio)"
            disabled
            readonly
          >
          <p class="pista-precio">El precio se calcula según los servicios seleccionados</p>
          <p class="error" v-if="errores.precio">{{ errores.precio }}</p>

          <label>Propina (opcional)</label>
          <input type="number" v-model.number="formulario.propina" min="0" placeholder="0">
          <p class="error" v-if="errores.propina">{{ errores.propina }}</p>

          <label>Método de pago</label>
          <select v-model="formulario.metodoPago">
            <option value="" disabled>Selecciona...</option>
            <option v-for="m in metodosPago" :key="m" :value="m">{{ m }}</option>
          </select>
          <p class="error" v-if="errores.metodoPago">{{ errores.metodoPago }}</p>

          <label>Estado del pago</label>
          <select v-model="formulario.estadoPago">
            <option v-for="e in estadosPago" :key="e" :value="e">{{ e }}</option>
          </select>

          <template v-if="formulario.estadoPago === 'abonado'">
            <label>¿Cuánto abona?</label>
            <input type="number" v-model.number="formulario.montoAbonado" min="1">
            <p class="pista-precio" v-if="formulario.precio > 0">Total del servicio: {{ formatearPrecio(formulario.precio + (formulario.propina || 0)) }}</p>
            <p class="error" v-if="errores.montoAbonado">{{ errores.montoAbonado }}</p>
          </template>

          <label>Foto antes (opcional)</label>
          <input type="file" accept="image/*" @change="manejarFoto($event, 'fotoAntes')">
          <img v-if="formulario.fotoAntes" :src="formulario.fotoAntes" class="foto-preview">

          <label>Foto después (opcional)</label>
          <input type="file" accept="image/*" @change="manejarFoto($event, 'fotoDespues')">
          <img v-if="formulario.fotoDespues" :src="formulario.fotoDespues" class="foto-preview">
          <p class="pista-precio">Las fotos se guardan en el navegador; usa imágenes livianas (máx. 2 por servicio).</p>

          <div class="modal-acciones">
            <button type="submit" class="btn-nuevo" :disabled="guardando">
              <span v-if="guardando" class="spinner-boton"></span>
              <span v-else>Guardar</span>
            </button>
            <button type="button" class="btn-mini cancelar" @click="cerrarModal" :disabled="guardando">Cancelar</button>
          </div>
        </form>
      </div>
    </div>

    <div class="fondo-modal fondo-modal-encima" v-if="modalAbonoAbierto">
      <div class="modal">
        <h2>Registrar abono</h2>
        <p v-if="servicioAbonando">
          Cliente: <strong>{{ servicioAbonando.cliente }}</strong><br>
          Saldo pendiente: <strong>{{ formatearPrecio(saldoRestante(servicioAbonando)) }}</strong>
        </p>

        <label>¿Cuánto abona ahora?</label>
        <input type="number" v-model.number="montoNuevoAbono" min="1">
        <p class="error" v-if="errorAbono">{{ errorAbono }}</p>

        <div class="modal-acciones">
          <button class="btn-nuevo" @click="confirmarAbono">Guardar abono</button>
          <button class="btn-mini cancelar" @click="cancelarAbono">Cancelar</button>
        </div>
      </div>
    </div>

    <div class="fondo-modal" v-if="modalEliminarAbierto">
      <div class="modal">
        <h2>¿Eliminar servicio?</h2>
        <p v-if="servicioAEliminar">
          Cliente: <strong>{{ servicioAEliminar.cliente }}</strong> - {{ nombresServicios(servicioAEliminar.tiposServicio) }}
        </p>
        <div class="modal-acciones">
          <button class="btn-mini eliminar" @click="confirmarEliminar">Sí, eliminar</button>
          <button class="btn-mini cancelar" @click="cancelarEliminar">Cancelar</button>
        </div>
      </div>
    </div>

    <div class="fondo-modal" v-if="modalCatalogoAbierto">
      <div class="modal">
        <h2>Catálogo de servicios</h2>

        <div class="catalogo-lista">
          <div class="catalogo-item" v-for="(t, idx) in catalogoServicios" :key="t.nombre">
            <span class="catalogo-nombre">{{ t.nombre }}</span>
            <span class="catalogo-precio">{{ formatearPrecio(t.precio) }}</span>
            <div class="catalogo-acciones">
              <button type="button" class="btn-mini editar" @click="editarServicioCatalogo(idx)">Editar</button>
              <button type="button" class="btn-mini eliminar" @click="eliminarServicioCatalogo(idx)">Eliminar</button>
            </div>
          </div>
        </div>

        <div class="separador"></div>

        <label>Nombre del servicio</label>
        <input type="text" v-model="formCatalogoNombre" placeholder="Ej: Corte niño">
        <label>Precio base sugerido</label>
        <input type="number" v-model.number="formCatalogoPrecio" min="0" placeholder="0">

        <div class="modal-acciones">
          <button type="button" class="btn-nuevo" @click="agregarOActualizarServicioCatalogo">
            {{ catalogoEditandoIndex !== null ? 'Actualizar servicio' : 'Agregar servicio' }}
          </button>
          <button type="button" class="btn-mini cancelar" @click="cerrarCatalogo">Cerrar</button>
        </div>
      </div>
    </div>

    <div class="fondo-modal" v-if="modalCierreAbierto">
      <div class="modal">
        <h2>Cierre de caja</h2>
        <div class="resumen-cierre" v-if="resumenCierre">
          <div class="linea-resumen"><span>Servicios de hoy</span><span>{{ resumenCierre.cantidad }}</span></div>
          <div class="linea-resumen verde"><span>Total efectivo</span><span>{{ formatearPrecio(resumenCierre.efectivo) }}</span></div>
          <div class="linea-resumen verde"><span>Total transferencia</span><span>{{ formatearPrecio(resumenCierre.transferencia) }}</span></div>
          <div class="linea-resumen verde"><span>Total tarjeta</span><span>{{ formatearPrecio(resumenCierre.tarjeta) }}</span></div>
          <div class="linea-resumen rojo"><span>Pendientes por cobrar</span><span>{{ formatearPrecio(resumenCierre.pendientes) }}</span></div>
        </div>
        <p class="pista-precio">Al confirmar, los servicios de hoy se archivarán y dejarán de aparecer en la vista principal, pero podrás verlos y seguir cobrando saldos pendientes desde el botón "Archivados". Las deudas seguirán visibles en el panel de alertas.</p>
        <div class="modal-acciones">
          <button class="btn-nuevo" @click="confirmarCierreCaja">Confirmar y archivar</button>
          <button class="btn-mini cancelar" @click="cancelarCierreCaja">Cancelar</button>
        </div>
      </div>
    </div>

    <div class="fondo-modal" v-if="modalArchivadosAbierto">
      <div class="modal modal-ancho">
        <h2>Servicios archivados</h2>

        <div class="resumen-cierre">
          <div class="linea-resumen">
            <span>Total de tickets archivados</span>
            <span>{{ serviciosArchivados().length }}</span>
          </div>
          <div class="linea-resumen verde">
            <span>Total recaudado en archivados</span>
            <span>{{ formatearPrecio(totalArchivadoTotal()) }}</span>
          </div>
          <div class="linea-resumen rojo" v-if="totalPendienteArchivado() > 0">
            <span>Saldo pendiente dentro de archivados</span>
            <span>{{ formatearPrecio(totalPendienteArchivado()) }}</span>
          </div>
        </div>

        <div class="separador"></div>

        <p class="vacio" v-if="serviciosArchivados().length === 0">Todavía no has cerrado caja, aquí aparecerán los servicios archivados.</p>

        <div class="lista-tickets" v-else>
          <div class="ticket ticket-archivado" v-for="s in serviciosArchivadosOrdenados()" :key="s.id">
            <span class="estado" :class="claseEstado(s.estadoPago)">{{ s.estadoPago }}</span>
            <span class="etiqueta-archivado">Archivado</span>

            <div class="ticket-titulo">{{ s.cliente }}</div>
            <div class="separador"></div>

            <div class="servicios-bloque">
              <span class="etiqueta">Servicios</span>
              <div class="servicio-item" v-for="(nombre, idx) in s.tiposServicio" :key="idx">{{ nombre }}</div>
            </div>
            <div class="separador"></div>

            <div class="ticket-fila"><span>Barbero</span><span>{{ s.barbero }}</span></div>

            <div class="fecha-hora">
              <div class="caja-fecha">
                <span class="etiqueta">Fecha</span>
                <span>{{ formatearFecha(s.fecha) }}</span>
              </div>
              <div class="caja-fecha">
                <span class="etiqueta">Hora</span>
                <span>{{ s.hora }}</span>
              </div>
            </div>

            <div class="ticket-fila"><span>Pago</span><span>{{ s.metodoPago }}</span></div>

            <div class="separador"></div>
            <div class="ticket-fila ticket-total">
              <span>Total</span>
              <span v-if="s.propina > 0">{{ formatearPrecio(s.precio) }} + {{ formatearPrecio(s.propina) }} propina</span>
              <span v-else>{{ formatearPrecio(s.precio) }}</span>
            </div>
            <div class="ticket-fila fila-abonado" v-if="s.montoAbonado > 0 && s.montoAbonado < precioTotal(s)">
              <span>Abonado</span><span>{{ formatearPrecio(s.montoAbonado) }}</span>
            </div>
            <div class="ticket-fila fila-saldo" v-if="saldoRestante(s) > 0">
              <span>Saldo pendiente</span><span>{{ formatearPrecio(saldoRestante(s)) }}</span>
            </div>

            <div class="historial-abonos" v-if="s.historialAbonos && s.historialAbonos.length > 0">
              <span class="etiqueta">Historial de abonos</span>
              <div class="linea-abono" v-for="(a, idx) in s.historialAbonos" :key="idx">
                <span>{{ formatearFecha(a.fecha) }}</span>
                <span>{{ formatearPrecio(a.monto) }}</span>
              </div>
            </div>

            <button
              class="btn-mini abonar"
              v-if="saldoRestante(s) > 0"
              @click="abrirModalAbono(s)"
            >Agregar abono</button>

            <div class="calificacion-seccion" v-if="s.calificacion > 0">
              <span class="etiqueta">Calificación</span>
              <div class="estrellas">
                <span
                  v-for="n in [1,2,3,4,5]"
                  :key="n"
                  class="estrella"
                  :class="{ llena: n <= s.calificacion, baja: n <= s.calificacion && s.calificacion <= 2 }"
                >★</span>
              </div>
            </div>

            <p class="pista-precio" v-if="s.observaciones">{{ s.observaciones }}</p>
          </div>
        </div>

        <div class="modal-acciones">
          <button type="button" class="btn-mini cancelar" @click="modalArchivadosAbierto = false">Cerrar</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useLocalStorage } from '@vueuse/core'

const barberos = ['Don Ramiro', 'Carlos Pérez', 'Andrés Gómez']

const catalogoServicios = useLocalStorage('br-catalogo-servicios', [
  { nombre: 'Corte Mullet', precio: 15000 },
  { nombre: 'Corte Buzz Cut', precio: 15000 },
  { nombre: 'Corte Taper Fade Texturizado', precio: 18000 },
  { nombre: 'Corte French Crop', precio: 18000 },
  { nombre: 'Depilación', precio: 10000 },
  { nombre: 'Barba', precio: 10000 },
  { nombre: 'Corte + barba', precio: 25000 },
  { nombre: 'Cejas', precio: 8000 },
  { nombre: 'Tinte', precio: 30000 }
])

// Cortes de cabello: solo se puede elegir UNO de estos a la vez.
const grupoCortes = [
  'Corte Mullet',
  'Corte Buzz Cut',
  'Corte Taper Fade Texturizado',
  'Corte French Crop'
]
// "Corte + barba" ya incluye un corte y la barba, así que no se puede combinar
// con un corte suelto ni con "Barba". "Barba" sola sí se puede combinar con
// cualquier corte del grupo de arriba, solo no con "Corte + barba".
// Cejas, Depilación y Tinte quedan por fuera y se pueden combinar libremente.

const comisiones = useLocalStorage('br-comisiones-barberos', {
  'Don Ramiro': 50,
  'Carlos Pérez': 40,
  'Andrés Gómez': 40
})

const metodosPago = ['Efectivo', 'Transferencia', 'Tarjeta']
const estadosPago = ['pagado', 'abonado', 'pendiente']

const opcionesOrden = [
  { key: 'fecha', label: 'Fecha' },
  { key: 'precio', label: 'Precio' },
  { key: 'calificacion', label: 'Calificación' }
]

function generarHorasDisponibles() {
  const horas = []
  for (let h = 8; h <= 20; h++) {
    for (const m of [0, 30]) {
      if (h === 20 && m === 30) continue
      const hh = String(h).padStart(2, '0')
      const mm = String(m).padStart(2, '0')
      horas.push(`${hh}:${mm}`)
    }
  }
  return horas
}
const horasDisponibles = generarHorasDisponibles()

const servicios = useLocalStorage('br-servicios-don-ramiro', [])
const cargando = ref(false)

const modalAbierto = ref(false)
const modoEdicion = ref(false)
const idEditando = ref(null)
const errores = ref({})
const guardando = ref(false)

const modalAbonoAbierto = ref(false)
const servicioAbonando = ref(null)
const montoNuevoAbono = ref(null)
const errorAbono = ref('')

const modalEliminarAbierto = ref(false)
const servicioAEliminar = ref(null)

const modalCatalogoAbierto = ref(false)
const formCatalogoNombre = ref('')
const formCatalogoPrecio = ref(null)
const catalogoEditandoIndex = ref(null)

const modalCierreAbierto = ref(false)
const resumenCierre = ref(null)

// Modal para ver los servicios que ya quedaron archivados al cerrar caja.
const modalArchivadosAbierto = ref(false)

const filtroBarbero = ref('Todos')
const filtroEstado = ref('Todos')
const ordenarPor = ref('fecha')
const ordenDireccion = ref('desc')
const busquedaCliente = ref('')

function fechaHoyString() {
  const hoy = new Date()
  const anio = hoy.getFullYear()
  const mes = String(hoy.getMonth() + 1).padStart(2, '0')
  const dia = String(hoy.getDate()).padStart(2, '0')
  return `${anio}-${mes}-${dia}`
}

const fechaMinima = ref(fechaHoyString())

// Hora actual en formato "HH:MM", para poder comparar contra las horas del selector.
function horaActualString() {
  const ahora = new Date()
  const hh = String(ahora.getHours()).padStart(2, '0')
  const mm = String(ahora.getMinutes()).padStart(2, '0')
  return `${hh}:${mm}`
}

// Si la fecha elegida es hoy, solo se muestran las horas que aún no han pasado.
// Si es una fecha futura, se muestran todas las horas normales (8am a 8pm).
const horasDisponiblesFormulario = computed(() => {
  if (formulario.value.fecha === fechaHoyString()) {
    const ahora = horaActualString()
    return horasDisponibles.filter(h => h > ahora)
  }
  return horasDisponibles
})

// Si el usuario cambia la fecha y la hora que tenía elegida ya no es válida
// (por ejemplo, eligió una hora que ya pasó y luego puso la fecha de hoy), se limpia.
function alCambiarFecha() {
  if (formulario.value.fecha === fechaHoyString()) {
    const ahora = horaActualString()
    if (formulario.value.hora && formulario.value.hora <= ahora) {
      formulario.value.hora = ''
    }
  }
}

function formularioVacio() {
  return {
    cliente: '',
    tiposServicio: [],
    barbero: '',
    fecha: '',
    hora: '',
    precio: 0,
    propina: null,
    metodoPago: '',
    estadoPago: 'pagado',
    montoAbonado: null,
    fotoAntes: null,
    fotoDespues: null
  }
}

const formulario = ref(formularioVacio())

function abrirModalNuevo() {
  modoEdicion.value = false
  idEditando.value = null
  formulario.value = formularioVacio()
  errores.value = {}
  modalAbierto.value = true
}

function abrirModalEditar(servicio) {
  if (servicio.calificacion > 0) return
  if (citaYaPaso(servicio)) return
  modoEdicion.value = true
  idEditando.value = servicio.id
  formulario.value = {
    cliente: servicio.cliente,
    tiposServicio: [...servicio.tiposServicio],
    barbero: servicio.barbero,
    fecha: servicio.fecha,
    hora: servicio.hora,
    precio: servicio.precio,
    propina: servicio.propina || null,
    metodoPago: servicio.metodoPago,
    estadoPago: servicio.estadoPago,
    montoAbonado: servicio.estadoPago === 'abonado' ? servicio.montoAbonado : null,
    fotoAntes: servicio.fotoAntes || null,
    fotoDespues: servicio.fotoDespues || null
  }
  errores.value = {}
  modalAbierto.value = true
}

function cerrarModal() {
  modalAbierto.value = false
}

// Aplica las reglas de exclusividad entre cortes, barba y "Corte + barba":
// - Solo un corte de cabello (Mullet, Buzz Cut, Taper Fade, French Crop) a la vez.
// - "Corte + barba" no se puede combinar con un corte suelto ni con "Barba" (ya los incluye).
// - "Barba" sola sí se puede combinar con un corte, pero no con "Corte + barba".
// Cejas, depilación y tinte no se ven afectados por estas reglas.
function alCambiarServicio(nombre) {
  const yaSeleccionado = formulario.value.tiposServicio.includes(nombre)
  if (!yaSeleccionado) {
    actualizarPrecioAutomatico()
    return
  }

  if (grupoCortes.includes(nombre)) {
    formulario.value.tiposServicio = formulario.value.tiposServicio.filter(
      n => n === nombre || (!grupoCortes.includes(n) && n !== 'Corte + barba')
    )
  } else if (nombre === 'Corte + barba') {
    formulario.value.tiposServicio = formulario.value.tiposServicio.filter(
      n => n === nombre || (!grupoCortes.includes(n) && n !== 'Barba')
    )
  } else if (nombre === 'Barba') {
    formulario.value.tiposServicio = formulario.value.tiposServicio.filter(
      n => n === nombre || n !== 'Corte + barba'
    )
  }

  actualizarPrecioAutomatico()
}

function actualizarPrecioAutomatico() {
  const total = formulario.value.tiposServicio.reduce((acc, nombre) => {
    const encontrado = catalogoServicios.value.find(t => t.nombre === nombre)
    return acc + (encontrado ? encontrado.precio : 0)
  }, 0)
  formulario.value.precio = total
}

function nombresServicios(lista) {
  if (!lista || lista.length === 0) return ''
  return lista.join(', ')
}

function claseEstado(estado) {
  if (estado === 'pagado') return 'estado-verde'
  if (estado === 'abonado') return 'estado-azul'
  return 'estado-naranja'
}

function precioTotal(servicio) {
  return Number(servicio.precio || 0) + Number(servicio.propina || 0)
}

function saldoRestante(servicio) {
  return Math.max(0, precioTotal(servicio) - Number(servicio.montoAbonado || 0))
}

// Indica si la fecha y hora de la cita ya pasaron respecto al momento actual.
function fechaHoraServicio(servicio) {
  return new Date(`${servicio.fecha}T${servicio.hora}:00`)
}

function citaYaPaso(servicio) {
  if (!servicio.fecha || !servicio.hora) return false
  return fechaHoraServicio(servicio).getTime() < Date.now()
}

function manejarFoto(event, campo) {
  const archivo = event.target.files[0]
  if (!archivo) return
  const lector = new FileReader()
  lector.onload = () => {
    formulario.value[campo] = lector.result
  }
  lector.readAsDataURL(archivo)
}

function clienteEsFrecuente() {
  if (!formulario.value.cliente || formulario.value.cliente.trim().length < 2) return false
  const nombre = formulario.value.cliente.trim().toLowerCase()
  const visitas = servicios.value.filter(
    s => s.cliente.trim().toLowerCase() === nombre && s.id !== idEditando.value
  ).length
  return visitas >= 5
}

function validarFormulario() {
  const err = {}
  if (!formulario.value.cliente || formulario.value.cliente.trim().length < 2) {
    err.cliente = 'Escribe el nombre del cliente.'
  }
  if (!formulario.value.tiposServicio || formulario.value.tiposServicio.length === 0) {
    err.tipoServicio = 'Selecciona al menos un tipo de servicio.'
  }
  if (!formulario.value.barbero) {
    err.barbero = 'Selecciona quién atendió.'
  }
  if (!formulario.value.fecha) {
    err.fecha = 'Selecciona la fecha.'
  } else if (formulario.value.fecha < fechaMinima.value) {
    err.fecha = 'No se puede agendar una fecha anterior a hoy.'
  }
  if (!formulario.value.hora) {
    err.hora = 'Selecciona la hora.'
  } else if (formulario.value.fecha === fechaHoyString() && formulario.value.hora <= horaActualString()) {
    err.hora = 'Esa hora ya pasó, elige una hora posterior a la actual.'
  }
  if (formulario.value.precio === null || formulario.value.precio === '' || Number(formulario.value.precio) <= 0) {
    err.precio = 'Selecciona al menos un servicio válido.'
  }
  if (formulario.value.propina !== null && formulario.value.propina !== '' && Number(formulario.value.propina) < 0) {
    err.propina = 'La propina no puede ser negativa.'
  }
  if (!formulario.value.metodoPago) {
    err.metodoPago = 'Selecciona el método de pago.'
  }
  if (formulario.value.estadoPago === 'abonado') {
    const monto = Number(formulario.value.montoAbonado)
    const totalConPropina = Number(formulario.value.precio) + Number(formulario.value.propina || 0)
    if (!formulario.value.montoAbonado || monto <= 0) {
      err.montoAbonado = 'Indica cuánto abona.'
    } else if (monto >= totalConPropina) {
      err.montoAbonado = 'Si abona el total, selecciona el estado "pagado".'
    }
  }
  errores.value = err
  return Object.keys(err).length === 0
}

function guardarServicio() {
  if (!validarFormulario()) return

  guardando.value = true

  setTimeout(() => {
    const precioFinal = Number(formulario.value.precio)
    const propinaFinal = Number(formulario.value.propina) || 0
    const totalFinal = precioFinal + propinaFinal
    let montoAbonadoFinal = 0
    let historialAbonos = []

    if (formulario.value.estadoPago === 'pagado') {
      montoAbonadoFinal = totalFinal
      historialAbonos = [{ monto: totalFinal, fecha: fechaHoyString() }]
    } else if (formulario.value.estadoPago === 'abonado') {
      montoAbonadoFinal = Number(formulario.value.montoAbonado)
      historialAbonos = [{ monto: montoAbonadoFinal, fecha: fechaHoyString() }]
    }

    if (modoEdicion.value) {
      const index = servicios.value.findIndex(s => s.id === idEditando.value)
      if (index !== -1) {
        servicios.value[index] = {
          ...servicios.value[index],
          ...formulario.value,
          precio: precioFinal,
          propina: propinaFinal,
          montoAbonado: montoAbonadoFinal,
          historialAbonos
        }
      }
    } else {
      servicios.value.push({
        ...formulario.value,
        id: Date.now(),
        precio: precioFinal,
        propina: propinaFinal,
        montoAbonado: montoAbonadoFinal,
        historialAbonos,
        calificacion: 0,
        observaciones: '',
        archivado: false
      })
    }
    guardando.value = false
    modalAbierto.value = false
  }, 900)
}

function abrirModalAbono(servicio) {
  servicioAbonando.value = servicio
  montoNuevoAbono.value = null
  errorAbono.value = ''
  modalAbonoAbierto.value = true
}

function cancelarAbono() {
  modalAbonoAbierto.value = false
  servicioAbonando.value = null
  montoNuevoAbono.value = null
  errorAbono.value = ''
}

function confirmarAbono() {
  const servicio = servicioAbonando.value
  if (!servicio) return

  const saldo = saldoRestante(servicio)
  const monto = Number(montoNuevoAbono.value)

  if (!montoNuevoAbono.value || monto <= 0) {
    errorAbono.value = 'Indica cuánto abona.'
    return
  }
  if (monto > saldo) {
    errorAbono.value = `El abono no puede superar el saldo pendiente (${formatearPrecio(saldo)}).`
    return
  }

  // Ojo: buscamos por id sobre "servicios.value" (la lista completa), así que esto
  // funciona igual de bien para un ticket activo que para uno ya archivado.
  const index = servicios.value.findIndex(s => s.id === servicio.id)
  if (index !== -1) {
    const actualizado = { ...servicios.value[index] }
    actualizado.montoAbonado = Number(actualizado.montoAbonado || 0) + monto
    actualizado.historialAbonos = [...(actualizado.historialAbonos || []), { monto, fecha: fechaHoyString() }]

    if (actualizado.montoAbonado >= precioTotal(actualizado)) {
      actualizado.montoAbonado = precioTotal(actualizado)
      actualizado.estadoPago = 'pagado'
    } else {
      actualizado.estadoPago = 'abonado'
    }

    servicios.value[index] = actualizado
  }

  cancelarAbono()
}

function calificarServicio(servicio, n) {
  if (servicio.estadoPago !== 'pagado') return
  const index = servicios.value.findIndex(s => s.id === servicio.id)
  if (index !== -1) {
    if (servicios.value[index].calificacion > 0) return
    servicios.value[index].calificacion = n
  }
}

function pedirConfirmacionEliminar(servicio) {
  if (citaYaPaso(servicio)) return
  servicioAEliminar.value = servicio
  modalEliminarAbierto.value = true
}

function cancelarEliminar() {
  modalEliminarAbierto.value = false
  servicioAEliminar.value = null
}

function confirmarEliminar() {
  servicios.value = servicios.value.filter(s => s.id !== servicioAEliminar.value.id)
  modalEliminarAbierto.value = false
  servicioAEliminar.value = null
}

// --- Activos vs archivados -------------------------------------------------
// "Activos" = todavía visibles en la lista principal de tickets (no se han
// mandado a un cierre de caja). "Archivados" = ya pasaron por "Cerrar caja".
function serviciosActivos() {
  return servicios.value.filter(s => !s.archivado)
}

function serviciosArchivados() {
  return servicios.value.filter(s => s.archivado)
}

function serviciosArchivadosOrdenados() {
  return [...serviciosArchivados()].sort((a, b) => {
    const fa = `${a.fecha} ${a.hora}`
    const fb = `${b.fecha} ${b.hora}`
    return fa < fb ? 1 : fa > fb ? -1 : 0
  })
}

function serviciosFiltrados() {
  return serviciosActivos().filter(s => {
    const pasaBarbero = filtroBarbero.value === 'Todos' || s.barbero === filtroBarbero.value
    const pasaEstado = filtroEstado.value === 'Todos' || s.estadoPago === filtroEstado.value
    return pasaBarbero && pasaEstado
  })
}

function cambiarOrden(campo) {
  if (ordenarPor.value === campo) {
    ordenDireccion.value = ordenDireccion.value === 'asc' ? 'desc' : 'asc'
  } else {
    ordenarPor.value = campo
    ordenDireccion.value = 'desc'
  }
}

function ordenarServicios(lista) {
  const dir = ordenDireccion.value === 'asc' ? 1 : -1
  return [...lista].sort((a, b) => {
    if (ordenarPor.value === 'fecha') {
      const fa = `${a.fecha} ${a.hora}`
      const fb = `${b.fecha} ${b.hora}`
      return fa < fb ? -1 * dir : fa > fb ? 1 * dir : 0
    }
    if (ordenarPor.value === 'precio') {
      return (precioTotal(a) - precioTotal(b)) * dir
    }
    if (ordenarPor.value === 'calificacion') {
      return (a.calificacion - b.calificacion) * dir
    }
    return 0
  })
}

function turnoDeHora(hora) {
  if (!hora) return 'Mañana'
  const h = parseInt(hora.split(':')[0], 10)
  if (h < 12) return 'Mañana'
  if (h < 18) return 'Tarde'
  return 'Noche'
}

function serviciosPorTurno() {
  const filtrados = ordenarServicios(serviciosFiltrados())
  const grupos = { 'Mañana': [], 'Tarde': [], 'Noche': [] }
  filtrados.forEach(s => grupos[turnoDeHora(s.hora)].push(s))
  return grupos
}

function esHoy(fecha) {
  return fecha === fechaHoyString()
}

// IMPORTANTE: antes esta función filtraba también por "no archivado", así que
// apenas cerrabas caja las estadísticas y comisiones de "hoy" se iban a cero,
// aunque esa plata sí se hubiera cobrado hoy mismo. Ahora cuenta todos los
// servicios de la fecha de hoy, estén o no archivados.
function serviciosHoy() {
  return servicios.value.filter(s => esHoy(s.fecha))
}

// --- Sumas de dinero ---------------------------------------------------
// Dinero cobrado hoy (incluye lo ya archivado si cerraste caja hoy mismo).
function totalVendidoHoy() {
  return serviciosHoy().reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

// Servicios archivados de días anteriores (no de hoy), para no contar dos
// veces la plata de hoy si ya cerraste caja.
function serviciosArchivadosAnteriores() {
  return servicios.value.filter(s => s.archivado && !esHoy(s.fecha))
}
function totalArchivadoAnterior() {
  return serviciosArchivadosAnteriores().reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

// Caso raro pero posible: servicios de días anteriores que quedaron activos
// porque nunca se cerró caja ese día. Se muestran aparte para que no se
// pierdan del total general.
function serviciosActivosAnteriores() {
  return servicios.value.filter(s => !s.archivado && !esHoy(s.fecha))
}
function totalActivoAnterior() {
  return serviciosActivosAnteriores().reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

// Total general = hoy + archivados de antes + activos sueltos de antes.
// Esto es exactamente la suma de "montoAbonado" de TODOS los servicios,
// así que siempre cuadra sin importar si el servicio está archivado o no.
function totalGeneralRecaudado() {
  return totalVendidoHoy() + totalArchivadoAnterior() + totalActivoAnterior()
}

// Total recaudado solo dentro de lo archivado (de hoy o de antes), para
// mostrar dentro del modal de "Archivados".
function totalArchivadoTotal() {
  return serviciosArchivados().reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

function totalPendienteArchivado() {
  return serviciosArchivados().reduce((acc, s) => acc + saldoRestante(s), 0)
}

function promedioCalificacionHoy() {
  const calificados = serviciosHoy().filter(s => s.calificacion > 0)
  if (calificados.length === 0) return 0
  return Number((calificados.reduce((acc, s) => acc + s.calificacion, 0) / calificados.length).toFixed(1))
}

function barberoConMasCortesHoy() {
  const conteo = {}
  serviciosHoy().forEach(s => { conteo[s.barbero] = (conteo[s.barbero] || 0) + 1 })
  let mejor = null
  let max = 0
  for (const b in conteo) {
    if (conteo[b] > max) {
      max = conteo[b]
      mejor = b
    }
  }
  return mejor ? `${mejor} (${max})` : 'Sin datos'
}

function historialCliente() {
  const q = busquedaCliente.value.trim().toLowerCase()
  if (!q) return null
  const coincidencias = servicios.value.filter(s => s.cliente.toLowerCase().includes(q))
  const visitas = coincidencias.length
  const totalGastado = coincidencias.reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
  return { nombre: busquedaCliente.value, visitas, totalGastado }
}

// Las deudas se calculan sobre TODOS los servicios (archivados o no), porque
// un cliente le sigue debiendo a la barbería aunque ya hayas cerrado caja.
function deudasPorCliente() {
  const deudas = {}
  servicios.value
    .filter(s => saldoRestante(s) > 0)
    .forEach(s => {
      const saldo = saldoRestante(s)
      deudas[s.cliente] = (deudas[s.cliente] || 0) + saldo
    })
  return Object.entries(deudas).map(([cliente, total]) => ({ cliente, total }))
}

function gananciaBarberoHoy(barbero) {
  return serviciosHoy()
    .filter(s => s.barbero === barbero)
    .reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

function comisionBarbero(barbero) {
  const pct = Number(comisiones.value[barbero] || 0)
  return gananciaBarberoHoy(barbero) * pct / 100
}

function abrirCatalogo() {
  formCatalogoNombre.value = ''
  formCatalogoPrecio.value = null
  catalogoEditandoIndex.value = null
  modalCatalogoAbierto.value = true
}

function cerrarCatalogo() {
  modalCatalogoAbierto.value = false
  formCatalogoNombre.value = ''
  formCatalogoPrecio.value = null
  catalogoEditandoIndex.value = null
}

function agregarOActualizarServicioCatalogo() {
  if (!formCatalogoNombre.value || formCatalogoNombre.value.trim().length < 2) return
  if (!formCatalogoPrecio.value || Number(formCatalogoPrecio.value) <= 0) return

  if (catalogoEditandoIndex.value !== null) {
    catalogoServicios.value[catalogoEditandoIndex.value] = {
      nombre: formCatalogoNombre.value.trim(),
      precio: Number(formCatalogoPrecio.value)
    }
  } else {
    catalogoServicios.value.push({
      nombre: formCatalogoNombre.value.trim(),
      precio: Number(formCatalogoPrecio.value)
    })
  }
  formCatalogoNombre.value = ''
  formCatalogoPrecio.value = null
  catalogoEditandoIndex.value = null
}

function editarServicioCatalogo(idx) {
  catalogoEditandoIndex.value = idx
  formCatalogoNombre.value = catalogoServicios.value[idx].nombre
  formCatalogoPrecio.value = catalogoServicios.value[idx].precio
}

function eliminarServicioCatalogo(idx) {
  catalogoServicios.value.splice(idx, 1)
}

function abrirCierreCaja() {
  const hoy = serviciosHoy()
  resumenCierre.value = {
    cantidad: hoy.length,
    efectivo: hoy.filter(s => s.metodoPago === 'Efectivo').reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0),
    transferencia: hoy.filter(s => s.metodoPago === 'Transferencia').reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0),
    tarjeta: hoy.filter(s => s.metodoPago === 'Tarjeta').reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0),
    pendientes: hoy.reduce((acc, s) => acc + saldoRestante(s), 0)
  }
  modalCierreAbierto.value = true
}

function cancelarCierreCaja() {
  modalCierreAbierto.value = false
  resumenCierre.value = null
}

function confirmarCierreCaja() {
  const hoyStr = fechaHoyString()
  servicios.value = servicios.value.map(s => (s.fecha === hoyStr && !s.archivado) ? { ...s, archivado: true } : s)
  modalCierreAbierto.value = false
  resumenCierre.value = null
}

function formatearFecha(fecha) {
  if (!fecha) return ''
  const [anio, mes, dia] = fecha.split('-')
  return `${dia}/${mes}/${anio}`
}

function formatearPrecio(precio) {
  return Number(precio || 0).toLocaleString('es-CO', { style: 'currency', currency: 'COP', minimumFractionDigits: 0 })
}

// Saldo pendiente total: se calcula sobre TODOS los servicios (no solo los
// activos), porque un servicio archivado que quedó "abonado" o "pendiente"
// sigue debiendo plata aunque ya no aparezca en la lista principal.
function totalPendientes() {
  return servicios.value.reduce((acc, s) => acc + saldoRestante(s), 0)
}

function totalPendientesCantidad() {
  return servicios.value.filter(s => saldoRestante(s) > 0).length
}
</script>

<style>
* {
  box-sizing: border-box;
}

html, body, #app {
  width: 100%;
  min-width: 100%;
  margin: 0;
  padding: 0;
  display: block;
  overflow-x: hidden;
}

.pagina {
  font-family: Arial, sans-serif;
  font-size: 28px;
  width: 100%;
  max-width: 100%;
  margin: 0;
  padding: 16px 24px;
  background: #f5f1e8;
  min-height: 100vh;
  color: #222;
  overflow-x: hidden;
}

.lista-tickets {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 18px;
}

.spinner-boton {
  display: inline-block;
  width: 16px;
  height: 16px;
  border: 3px solid rgba(255, 255, 255, 0.4);
  border-top-color: #fff;
  border-radius: 50%;
  animation: girar 0.7s linear infinite;
}
@keyframes girar {
  to { transform: rotate(360deg); }
}

.marca {
  position: relative;
  text-align: center;
  margin: -16px -24px 16px -24px;
  padding: 150px 24px 24px;
  border-bottom: 3px solid #7a1f1f;
  min-height: 160px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  color: #fff;
  background-image:
    linear-gradient(to bottom, rgba(0, 0, 0, 0.15), rgba(0, 0, 0, 0.65)),
    url('./assets/img/barberia.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
.marca h1 {
  margin: 0;
  font-size: 36px;
  color: #fff;
  text-shadow: 0 2px 6px rgba(0, 0, 0, 0.6);
}
.marca p {
  margin: 2px 0 0;
  font-size: 24px;
  color: #f0f0f0;
  text-shadow: 0 1px 4px rgba(0, 0, 0, 0.6);
}

.fila-botones-top {
  display: flex;
  gap: 8px;
  margin-bottom: 14px;
}
.fila-botones-top .btn-secundario {
  flex: 1;
}
.btn-secundario {
  background: #fff;
  color: #7a1f1f;
  border: 2px solid #7a1f1f;
  padding: 12px;
  font-weight: bold;
  font-size: 22px;
  border-radius: 4px;
  cursor: pointer;
}

.btn-nuevo {
  width: 100%;
  background: #7a1f1f;
  color: #fff;
  border: none;
  padding: 12px;
  font-weight: bold;
  font-size: 28px;
  border-radius: 4px;
  cursor: pointer;
  margin-bottom: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.btn-nuevo:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.caja-resumen {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 10px 14px;
  margin-bottom: 14px;
  font-size: 26px;
}
.linea-resumen {
  display: flex;
  justify-content: space-between;
  padding: 3px 0;
}
.linea-resumen.verde span:last-child { color: #2e6b2e; font-weight: bold; }
.linea-resumen.rojo span:last-child { color: #7a1f1f; font-weight: bold; }
.linea-resumen.total-general {
  border-top: 2px solid #ddd6c9;
  margin-top: 6px;
  padding-top: 8px;
}
.linea-resumen.total-general span:first-child { font-weight: bold; }
.linea-resumen.total-general span:last-child { font-size: 1.05em; }

.caja-estadisticas,
.caja-historial-cliente,
.caja-deudas,
.caja-comisiones {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 10px 14px;
  margin-bottom: 14px;
  font-size: 26px;
}
.caja-estadisticas h3,
.caja-deudas h3,
.caja-comisiones h3 {
  margin: 0 0 6px;
  font-size: 26px;
  color: #7a1f1f;
}

.caja-historial-cliente label { margin-top: 0; }
.resultado-historial {
  margin-top: 8px;
  background: #f5f1e8;
  border-radius: 4px;
  padding: 8px 10px;
  font-size: 24px;
}
.resultado-historial p { margin: 2px 0; }

.caja-deudas { border-color: #e3b3b3; }
.linea-deuda {
  display: flex;
  justify-content: space-between;
  padding: 3px 0;
  color: #7a1f1f;
  font-weight: bold;
}

.orden-botones {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
  margin-bottom: 14px;
}
.orden-botones .etiqueta { font-size: 22px; }
.btn-orden {
  background: #fff;
  border: 1px solid #ccc;
  border-radius: 14px;
  padding: 6px 14px;
  font-size: 22px;
  cursor: pointer;
}
.btn-orden.activo {
  background: #7a1f1f;
  color: #fff;
  border-color: #7a1f1f;
}

.turno-grupo { margin-bottom: 10px; }
.turno-titulo {
  font-size: 28px;
  font-weight: bold;
  color: #7a1f1f;
  border-bottom: 2px solid #7a1f1f;
  padding-bottom: 4px;
  margin-bottom: 12px;
}

.alerta-fidelidad {
  background: #fff3d6;
  color: #8a5a00;
  border: 1px solid #e0b94d;
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 21px;
  font-weight: bold;
  margin: 8px 0;
}

.fotos-antes-despues {
  display: flex;
  gap: 10px;
  margin: 8px 0;
}
.foto-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.foto-mini {
  width: 100%;
  max-height: 120px;
  object-fit: cover;
  border-radius: 4px;
  margin-top: 4px;
}
.foto-preview {
  width: 100%;
  max-height: 160px;
  object-fit: cover;
  border-radius: 8px;
  margin-top: 8px;
  border: 1px solid #e4ddd0;
}

.caja-comisiones .comisiones-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}
.btn-mini.cerrar-caja {
  background: #7a1f1f;
  color: #fff;
  flex: none;
  padding: 8px 14px;
}
.linea-comision {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  padding: 4px 0;
}
.comision-nombre { flex: 1; }
.comision-pct { display: flex; align-items: center; gap: 4px; }
.comision-pct input {
  width: 70px;
  padding: 4px;
  font-size: 22px;
  margin: 0;
}
.comision-monto { font-weight: bold; color: #2e6b2e; min-width: 110px; text-align: right; }

.catalogo-lista { max-height: 260px; overflow-y: auto; overflow-x: hidden; margin-bottom: 8px; }
.catalogo-item {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 4px 8px;
  padding: 8px 0;
  border-bottom: 1px solid #eee;
  font-size: 20px;
}
.catalogo-nombre { flex: 1 1 60%; }
.catalogo-precio { flex: 1 1 30%; color: #2e6b2e; font-weight: bold; text-align: right; }
.catalogo-acciones { flex: 1 1 100%; display: flex; gap: 6px; justify-content: flex-end; }
.catalogo-acciones .btn-mini { padding: 6px 12px; font-size: 18px; flex: none; }

.resumen-cierre { margin-bottom: 10px; }

.filtros {
  display: flex;
  gap: 8px;
  margin-bottom: 14px;
}
.filtros select {
  flex: 1;
  width: 50%;
  padding: 6px;
  font-size: 24px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.vacio {
  text-align: center;
  color: #888;
  margin: 24px 0;
  font-size: 28px;
}

.ticket {
  position: relative;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 20px;
  margin-bottom: 18px;
  font-size: 28px;
}

.ticket-archivado {
  background: #faf8f3;
}
.etiqueta-archivado {
  position: absolute;
  top: 14px;
  left: 20px;
  font-size: 18px;
  text-transform: uppercase;
  color: #8a8375;
  font-weight: bold;
}
.ticket-archivado .ticket-titulo {
  margin-top: 18px;
}

.estado {
  position: absolute;
  top: 14px;
  right: 14px;
  font-size: 22px;
  font-weight: bold;
  text-transform: uppercase;
  padding: 3px 10px;
  border-radius: 12px;
}
.estado-verde { background: #e2f3e2; color: #2e6b2e; }
.estado-naranja { background: #fbe8d3; color: #b5590a; }
.estado-azul { background: #dde8f7; color: #1f4e7a; }

.ticket-titulo {
  font-size: 34px;
  font-weight: bold;
  margin-bottom: 8px;
}

.separador {
  border-top: 1px solid #cfcfcf;
  margin: 10px 0;
}

.servicios-bloque {
  padding: 4px 0;
}
.servicio-item {
  padding: 2px 0 2px 6px;
  border-left: 3px solid #7a1f1f;
  margin-top: 4px;
}

.ticket-fila {
  display: flex;
  justify-content: space-between;
  padding: 3px 0;
  gap: 10px;
}

.fila-abonado span:last-child { color: #1f4e7a; font-weight: bold; }
.fila-saldo span:last-child { color: #7a1f1f; font-weight: bold; }

.historial-abonos {
  margin: 8px 0;
  background: #f5f1e8;
  border-radius: 4px;
  padding: 6px 10px;
}
.linea-abono {
  display: flex;
  justify-content: space-between;
  font-size: 22px;
  padding: 2px 0;
}

.btn-mini.abonar {
  width: 100%;
  background: #1f4e7a;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px;
  font-size: 24px;
  font-weight: bold;
  cursor: pointer;
  margin: 8px 0;
}

.fecha-hora {
  display: flex;
  gap: 10px;
  margin: 8px 0;
}
.caja-fecha {
  flex: 1;
  min-width: 0;
  background: #f5f1e8;
  border-radius: 4px;
  padding: 6px 10px;
  display: flex;
  flex-direction: column;
}
.etiqueta {
  font-size: 22px;
  color: #888;
  text-transform: uppercase;
}

.ticket-total span {
  font-weight: bold;
  font-size: 32px;
  color: #7a1f1f;
}

.calificacion-seccion {
  margin: 8px 0;
}
.estrellas {
  margin-top: 4px;
}
.estrellas.bloqueada .estrella {
  cursor: not-allowed;
  opacity: 0.5;
}
.estrella {
  font-size: 40px;
  color: #ddd;
  cursor: pointer;
  margin-right: 2px;
}
.estrella.llena { color: #b5590a; }
.estrella.llena.baja { color: #7a1f1f; }

.aviso-baja {
  font-size: 24px;
  color: #7a1f1f;
  margin: 4px 0 0;
}

.obs-input {
  width: 100%;
  margin-top: 4px;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: inherit;
  font-size: 26px;
  resize: vertical;
  min-height: 50px;
}

.ticket-acciones {
  display: flex;
  gap: 8px;
  margin-top: 14px;
}
.btn-mini {
  flex: 1;
  border: none;
  border-radius: 4px;
  padding: 8px;
  font-size: 26px;
  cursor: pointer;
  font-weight: bold;
}
.btn-mini.editar { background: #2e6b2e; color: white; }
.btn-mini.editar:disabled { background: #a9c7a9; cursor: not-allowed; }
.btn-mini.eliminar { background: #7a1f1f; color: white; }
.btn-mini.eliminar:disabled { background: #d9a8a8; cursor: not-allowed; }
.btn-mini.eliminar:hover:not(:disabled) { background: #5f1717; }
.btn-mini.editar:hover:not(:disabled) { background: #235723; }
.btn-mini.cancelar { background: #e0e0e0; color: #222; }
.btn-mini.cancelar:hover { background: #cfcfcf; }
.btn-nuevo:hover:not(:disabled) { background: #631a1a; }

label { display: block; margin-top: 18px; margin-bottom: 2px; font-weight: bold; font-size: 23px; color: #4a4a4a; }
input, select, textarea {
  width: 100%;
  padding: 11px 14px;
  margin-top: 4px;
  border-radius: 8px;
  border: 1.5px solid #ddd6c9;
  font-family: inherit;
  font-size: 26px;
  background: #fdfcf9;
  transition: border-color .15s ease, box-shadow .15s ease;
}
input:focus, select:focus, textarea:focus {
  outline: none;
  border-color: #7a1f1f;
  box-shadow: 0 0 0 3px rgba(122, 31, 31, 0.15);
  background: #fff;
}
.precio-bloqueado {
  background: #f5f1e8;
  color: #7a1f1f;
  font-weight: bold;
  cursor: not-allowed;
  border-style: dashed;
}
.pista-precio {
  font-size: 19px;
  color: #8a8375;
  margin: 5px 0 0;
  line-height: 1.4;
}
.error {
  color: #7a1f1f;
  background: #fbeeee;
  border-left: 3px solid #7a1f1f;
  padding: 5px 10px;
  border-radius: 4px;
  font-size: 20px;
  margin: 5px 0;
}

.fila-doble {
  display: flex;
  gap: 20px;
}
.fila-doble > div {
  flex: 1;
  min-width: 0;
}

.lista-checks {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
  gap: 10px;
  margin-top: 8px;
}
.check-item {
  display: flex;
  align-items: center;
  gap: 10px;
  font-weight: normal;
  margin-top: 0;
  font-size: 21px;
  padding: 11px 14px;
  border: 1.5px solid #e4ddd0;
  border-radius: 8px;
  background: #fdfcf9;
  cursor: pointer;
  transition: border-color .15s ease, background .15s ease;
}
.check-item:hover {
  border-color: #cbb2b2;
}
.check-item:has(input:checked) {
  border-color: #7a1f1f;
  background: #fbeeee;
}
.check-item input[type="checkbox"] {
  width: 19px;
  height: 19px;
  margin: 0;
  accent-color: #7a1f1f;
}

.fondo-modal {
  position: fixed;
  inset: 0;
  background: rgba(20, 12, 8, 0.55);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px;
  z-index: 40;
}
/* El modal de abono se puede abrir DESDE DENTRO del modal de Archivados
   (o del de catálogo/cierre en un futuro), así que necesita quedar por
   encima de cualquier otro modal abierto, no detrás. */
.fondo-modal.fondo-modal-encima {
  z-index: 60;
}
.modal {
  background: #fff;
  padding: 34px 38px 30px;
  border-radius: 14px;
  width: 100%;
  max-width: 620px;
  max-height: 90vh;
  overflow-y: auto;
  overflow-x: hidden;
  box-shadow: 0 28px 60px rgba(20, 8, 8, 0.4);
  border-top: 6px solid #7a1f1f;
}
.modal.modal-ancho {
  max-width: 960px;
}
.modal h2 {
  margin: 0 0 20px;
  color: #7a1f1f;
  font-size: 32px;
  padding-bottom: 14px;
  border-bottom: 2px solid #f0e4d8;
}
.modal-acciones {
  display: flex;
  gap: 16px;
  margin-top: 26px;
  padding-top: 18px;
  border-top: 1px solid #f0e4d8;
}
.modal-acciones .btn-nuevo, .modal-acciones .btn-mini { margin: 0; }
</style>