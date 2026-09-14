<template>
  <div class="pagina">
    <div class="marca">
      <h1>Barbería Don Ramiro</h1>
      <p>Registro de servicios</p>
    </div>

    <button class="btn-nuevo" @click="abrirModalNuevo">Registrar servicio</button>

    <div class="caja-resumen">
      <div class="linea-resumen">
        <span>Servicios registrados</span>
        <span>{{ servicios.length }}</span>
      </div>
      <div class="linea-resumen verde">
        <span>Total recaudado</span>
        <span>{{ formatearPrecio(totalIngresos()) }}</span>
      </div>
      <div class="linea-resumen rojo" v-if="totalPendientesCantidad() > 0">
        <span>Con saldo pendiente ({{ totalPendientesCantidad() }})</span>
        <span>{{ formatearPrecio(totalPendientes()) }}</span>
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

    <p class="vacio" v-if="!cargando && serviciosFiltrados().length === 0">No hay servicios para mostrar</p>

    <div class="lista-tickets">
    <div class="ticket" v-for="s in serviciosFiltrados()" :key="s.id">
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
      <div class="ticket-fila ticket-total"><span>Total</span><span>{{ formatearPrecio(s.precio) }}</span></div>
      <div class="ticket-fila fila-abonado" v-if="s.montoAbonado > 0 && s.montoAbonado < s.precio">
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

      <div class="ticket-acciones">
        <button
          class="btn-mini editar"
          @click="abrirModalEditar(s)"
          :disabled="s.calificacion > 0"
          :title="s.calificacion > 0 ? 'No se puede editar un servicio ya calificado' : ''"
        >Editar</button>
        <button class="btn-mini eliminar" @click="pedirConfirmacionEliminar(s)">Eliminar</button>
      </div>
    </div>
    </div>

    <div class="fondo-modal" v-if="modalAbierto">
      <div class="modal">
        <h2>{{ modoEdicion ? 'Editar servicio' : 'Nuevo servicio' }}</h2>

        <form @submit.prevent="guardarServicio">
          <label>Cliente</label>
          <input type="text" v-model="formulario.cliente">
          <p class="error" v-if="errores.cliente">{{ errores.cliente }}</p>

          <label>Tipo de servicio</label>
          <div class="lista-checks">
            <label class="check-item" v-for="t in tiposServicio" :key="t.nombre">
              <input
                type="checkbox"
                :value="t.nombre"
                v-model="formulario.tiposServicio"
                @change="actualizarPrecioAutomatico"
              >
              <span>{{ t.nombre }} ({{ formatearPrecio(t.precio) }})</span>
            </label>
          </div>
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
              <input type="date" v-model="formulario.fecha" :min="fechaMinima">
              <p class="error" v-if="errores.fecha">{{ errores.fecha }}</p>
            </div>
            <div>
              <label>Hora</label>
              <select v-model="formulario.hora">
                <option value="" disabled>Selecciona...</option>
                <option v-for="h in horasDisponibles" :key="h" :value="h">{{ h }}</option>
              </select>
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
            <p class="pista-precio" v-if="formulario.precio > 0">Total del servicio: {{ formatearPrecio(formulario.precio) }}</p>
            <p class="error" v-if="errores.montoAbonado">{{ errores.montoAbonado }}</p>
          </template>

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

    <div class="fondo-modal" v-if="modalAbonoAbierto">
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
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useLocalStorage } from '@vueuse/core'

const barberos = ['Don Ramiro', 'Carlos Pérez', 'Andrés Gómez']
const tiposServicio = [
  { nombre: 'Corte tradicional', precio: 15000 },
  { nombre: 'Depilación', precio: 10000 },
  { nombre: 'Barba', precio: 10000 },
  { nombre: 'Corte + barba', precio: 25000 },
  { nombre: 'Cejas', precio: 8000 },
  { nombre: 'Tinte', precio: 30000 }
]
const metodosPago = ['Efectivo', 'Transferencia', 'Tarjeta']
const estadosPago = ['pagado', 'abonado', 'pendiente']

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

const modalAbierto = ref(false)
const modoEdicion = ref(false)
const idEditando = ref(null)
const errores = ref({})
const guardando = ref(false)

const modalAbonoAbierto = ref(false)
const servicioAbonando = ref(null)
const montoNuevoAbono = ref(null)
const errorAbono = ref('')

function fechaHoyString() {
  const hoy = new Date()
  const anio = hoy.getFullYear()
  const mes = String(hoy.getMonth() + 1).padStart(2, '0')
  const dia = String(hoy.getDate()).padStart(2, '0')
  return `${anio}-${mes}-${dia}`
}

const fechaMinima = ref(fechaHoyString())

function formularioVacio() {
  return {
    cliente: '',
    tiposServicio: [],
    barbero: '',
    fecha: '',
    hora: '',
    precio: 0,
    metodoPago: '',
    estadoPago: 'pagado',
    montoAbonado: null
  }
}

const formulario = ref(formularioVacio())

const modalEliminarAbierto = ref(false)
const servicioAEliminar = ref(null)

const filtroBarbero = ref('Todos')
const filtroEstado = ref('Todos')

function abrirModalNuevo() {
  modoEdicion.value = false
  idEditando.value = null
  formulario.value = formularioVacio()
  errores.value = {}
  modalAbierto.value = true
}

function abrirModalEditar(servicio) {
  if (servicio.calificacion > 0) return 
  modoEdicion.value = true
  idEditando.value = servicio.id
  formulario.value = {
    cliente: servicio.cliente,
    tiposServicio: [...servicio.tiposServicio],
    barbero: servicio.barbero,
    fecha: servicio.fecha,
    hora: servicio.hora,
    precio: servicio.precio,
    metodoPago: servicio.metodoPago,
    estadoPago: servicio.estadoPago,
    montoAbonado: servicio.estadoPago === 'abonado' ? servicio.montoAbonado : null
  }
  errores.value = {}
  modalAbierto.value = true
}

function cerrarModal() {
  modalAbierto.value = false
}

function actualizarPrecioAutomatico() {
  const total = formulario.value.tiposServicio.reduce((acc, nombre) => {
    const encontrado = tiposServicio.find(t => t.nombre === nombre)
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

function saldoRestante(servicio) {
  return Math.max(0, Number(servicio.precio || 0) - Number(servicio.montoAbonado || 0))
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
  }
  if (formulario.value.precio === null || formulario.value.precio === '' || Number(formulario.value.precio) <= 0) {
    err.precio = 'Selecciona al menos un servicio válido.'
  }
  if (!formulario.value.metodoPago) {
    err.metodoPago = 'Selecciona el método de pago.'
  }
  if (formulario.value.estadoPago === 'abonado') {
    const monto = Number(formulario.value.montoAbonado)
    if (!formulario.value.montoAbonado || monto <= 0) {
      err.montoAbonado = 'Indica cuánto abona.'
    } else if (monto >= Number(formulario.value.precio)) {
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
    let montoAbonadoFinal = 0
    let historialAbonos = []

    if (formulario.value.estadoPago === 'pagado') {
      montoAbonadoFinal = precioFinal
      historialAbonos = [{ monto: precioFinal, fecha: fechaHoyString() }]
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
          montoAbonado: montoAbonadoFinal,
          historialAbonos
        }
      }
    } else {
      servicios.value.push({
        ...formulario.value,
        id: Date.now(),
        precio: precioFinal,
        montoAbonado: montoAbonadoFinal,
        historialAbonos,
        calificacion: 0,
        observaciones: ''
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

  const index = servicios.value.findIndex(s => s.id === servicio.id)
  if (index !== -1) {
    const actualizado = { ...servicios.value[index] }
    actualizado.montoAbonado = Number(actualizado.montoAbonado || 0) + monto
    actualizado.historialAbonos = [...(actualizado.historialAbonos || []), { monto, fecha: fechaHoyString() }]

    if (actualizado.montoAbonado >= actualizado.precio) {
      actualizado.montoAbonado = actualizado.precio
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

function serviciosFiltrados() {
  return servicios.value.filter(s => {
    const pasaBarbero = filtroBarbero.value === 'Todos' || s.barbero === filtroBarbero.value
    const pasaEstado = filtroEstado.value === 'Todos' || s.estadoPago === filtroEstado.value
    return pasaBarbero && pasaEstado
  })
}

function formatearFecha(fecha) {
  if (!fecha) return ''
  const [anio, mes, dia] = fecha.split('-')
  return `${dia}/${mes}/${anio}`
}

function formatearPrecio(precio) {
  return Number(precio || 0).toLocaleString('es-CO', { style: 'currency', currency: 'COP', minimumFractionDigits: 0 })
}

function totalIngresos() {
  return servicios.value.reduce((acc, s) => acc + Number(s.montoAbonado || 0), 0)
}

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
  max-width: 1100px;
  margin: 0 auto;
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
  text-align: center;
  margin-bottom: 16px;
  border-bottom: 2px solid #7a1f1f;
  padding-bottom: 8px;
}
.marca h1 {
  margin: 0;
  font-size: 36px;
  color: #7a1f1f;
}
.marca p {
  margin: 2px 0 0;
  font-size: 24px;
  color: #666;
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
.btn-mini.cancelar { background: #e0e0e0; color: #222; }

label { display: block; margin-top: 8px; font-weight: bold; font-size: 24px; }
input, select, textarea {
  width: 100%;
  padding: 8px;
  margin-top: 3px;
  border-radius: 4px;
  border: 1px solid #ccc;
  font-family: inherit;
  font-size: 28px;
}
.precio-bloqueado {
  background: #eee;
  color: #444;
  font-weight: bold;
  cursor: not-allowed;
}
.pista-precio {
  font-size: 20px;
  color: #888;
  margin: 2px 0 0;
}
.error { color: #7a1f1f; font-size: 22px; margin: 2px 0; }

.fila-doble {
  display: flex;
  gap: 10px;
}
.fila-doble > div {
  flex: 1;
  min-width: 0;
}

.lista-checks {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-top: 4px;
}
.check-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: normal;
  margin-top: 0;
  font-size: 24px;
}
.check-item input[type="checkbox"] {
  width: auto;
  margin: 0;
}

.fondo-modal {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px;
  z-index: 40;
}
.modal {
  background: #fff;
  padding: 20px;
  border-radius: 6px;
  width: 100%;
  max-width: 400px;
  max-height: 90vh;
  overflow-y: auto;
  overflow-x: hidden;
}
.modal h2 { margin-top: 0; color: #7a1f1f; font-size: 32px; }
.modal-acciones { display: flex; gap: 10px; margin-top: 16px; }
.modal-acciones .btn-nuevo, .modal-acciones .btn-mini { margin: 0; }
</style>