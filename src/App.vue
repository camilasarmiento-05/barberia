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
        <span>Total recaudado (pagado)</span>
        <span>{{ formatearPrecio(totalIngresos()) }}</span>
      </div>
      <div class="linea-resumen rojo" v-if="totalPendientesCantidad() > 0">
        <span>Pendiente por cobrar ({{ totalPendientesCantidad() }})</span>
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
      <span class="estado" :class="s.estadoPago === 'pagado' ? 'estado-verde' : 'estado-naranja'">{{ s.estadoPago }}</span>

      <div class="ticket-titulo">{{ s.cliente }}</div>
      <div class="separador"></div>

      <div class="ticket-fila"><span>Servicio</span><span>{{ s.tipoServicio }}</span></div>
      <div class="ticket-fila"><span>Barbero</span><span>{{ s.barbero }}</span></div>

      <div class="fecha-hora">
        <div class="caja-fecha">
          <span class="etiqueta">Fecha</span>
          <span>{{ formatearFecha(s.fecha) }}</span>
        </div>
        <div class="caja-fecha">
          <span class="etiqueta">Hora</span>
          <span>{{ formatearHora(s.hora) }}</span>
        </div>
      </div>

      <div class="ticket-fila">
        <span>Pago</span>
        <span>{{ s.metodoPago }}</span>
      </div>

      <div class="separador"></div>
      <div class="ticket-fila ticket-total"><span>Total</span><span>{{ formatearPrecio(s.precio) }}</span></div>

      <div class="separador"></div>

      <div class="calificacion-seccion">
        <span class="etiqueta">Calificación</span>
        <div class="estrellas">
          <span
            v-for="n in [1,2,3,4,5]"
            :key="n"
            class="estrella"
            :class="{ llena: n <= s.calificacion, baja: n <= s.calificacion && s.calificacion <= 2 }"
            @click="calificarServicio(s, n)"
          >★</span>
        </div>
        <p class="aviso-baja" v-if="s.calificacion > 0 && s.calificacion <= 2">Cliente insatisfecho, revisar servicio</p>
      </div>

      <label class="etiqueta">Observaciones</label>
      <textarea
        class="obs-input"
        v-model="s.observaciones"
        placeholder="Escribe una observación..."
      ></textarea>

      <div class="ticket-acciones">
        <button class="btn-mini editar" @click="abrirModalEditar(s)">Editar</button>
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
          <select v-model="formulario.tipoServicio" @change="actualizarPrecioAutomatico">
            <option value="" disabled>Selecciona...</option>
            <option v-for="t in tiposServicio" :key="t.nombre" :value="t.nombre">{{ t.nombre }}</option>
          </select>
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
              <input type="date" v-model="formulario.fecha">
              <p class="error" v-if="errores.fecha">{{ errores.fecha }}</p>
            </div>
            <div>
              <label>Hora</label>
              <input type="time" v-model="formulario.hora">
              <p class="error" v-if="errores.hora">{{ errores.hora }}</p>
            </div>
          </div>

          <label>Precio</label>
          <input type="number" v-model.number="formulario.precio">
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

    <div class="fondo-modal" v-if="modalEliminarAbierto">
      <div class="modal">
        <h2>¿Eliminar servicio?</h2>
        <p v-if="servicioAEliminar">
          Cliente: <strong>{{ servicioAEliminar.cliente }}</strong> - {{ servicioAEliminar.tipoServicio }}
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
  { nombre: 'Corte clásico', precio: 15000 },
  { nombre: 'Corte moderno', precio: 18000 },
  { nombre: 'Barba', precio: 10000 },
  { nombre: 'Corte + barba', precio: 25000 },
  { nombre: 'Cejas', precio: 8000 },
  { nombre: 'Tinte', precio: 30000 },
  { nombre: 'Otro', precio: 0 }
]
const metodosPago = ['Efectivo', 'Transferencia', 'Tarjeta']
const estadosPago = ['pagado', 'pendiente']

const servicios = useLocalStorage('br-servicios-don-ramiro', [])

const modalAbierto = ref(false)
const modoEdicion = ref(false)
const idEditando = ref(null)
const errores = ref({})
const guardando = ref(false)

function formularioVacio() {
  return {
    cliente: '',
    tipoServicio: '',
    barbero: '',
    fecha: '',
    hora: '',
    precio: null,
    metodoPago: '',
    estadoPago: 'pagado'
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
  modoEdicion.value = true
  idEditando.value = servicio.id
  formulario.value = {
    cliente: servicio.cliente,
    tipoServicio: servicio.tipoServicio,
    barbero: servicio.barbero,
    fecha: servicio.fecha,
    hora: servicio.hora,
    precio: servicio.precio,
    metodoPago: servicio.metodoPago,
    estadoPago: servicio.estadoPago
  }
  errores.value = {}
  modalAbierto.value = true
}

function cerrarModal() {
  modalAbierto.value = false
}

function actualizarPrecioAutomatico() {
  const encontrado = tiposServicio.find(t => t.nombre === formulario.value.tipoServicio)
  if (encontrado && encontrado.nombre !== 'Otro') {
    formulario.value.precio = encontrado.precio
  }
}

function validarFormulario() {
  const err = {}
  if (!formulario.value.cliente || formulario.value.cliente.trim().length < 2) {
    err.cliente = 'Escribe el nombre del cliente.'
  }
  if (!formulario.value.tipoServicio) {
    err.tipoServicio = 'Selecciona el tipo de servicio.'
  }
  if (!formulario.value.barbero) {
    err.barbero = 'Selecciona quién atendió.'
  }
  if (!formulario.value.fecha) {
    err.fecha = 'Selecciona la fecha.'
  }
  if (!formulario.value.hora) {
    err.hora = 'Selecciona la hora.'
  }
  if (formulario.value.precio === null || formulario.value.precio === '' || Number(formulario.value.precio) <= 0) {
    err.precio = 'El precio debe ser mayor a 0.'
  }
  if (!formulario.value.metodoPago) {
    err.metodoPago = 'Selecciona el método de pago.'
  }
  errores.value = err
  return Object.keys(err).length === 0
}

function guardarServicio() {
  if (!validarFormulario()) return

  guardando.value = true

  setTimeout(() => {
    if (modoEdicion.value) {
      const index = servicios.value.findIndex(s => s.id === idEditando.value)
      if (index !== -1) {
        servicios.value[index] = {
          ...servicios.value[index],
          ...formulario.value,
          precio: Number(formulario.value.precio)
        }
      }
    } else {
      servicios.value.push({
        ...formulario.value,
        id: Date.now(),
        precio: Number(formulario.value.precio),
        calificacion: 0,
        observaciones: ''
      })
    }
    guardando.value = false
    modalAbierto.value = false
  }, 900)
}

function calificarServicio(servicio, n) {
  const index = servicios.value.findIndex(s => s.id === servicio.id)
  if (index !== -1) {
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

function formatearHora(hora) {
  if (!hora) return ''
  return hora
}

function formatearPrecio(precio) {
  return Number(precio || 0).toLocaleString('es-CO', { style: 'currency', currency: 'COP', minimumFractionDigits: 0 })
}

function totalIngresos() {
  return servicios.value.filter(s => s.estadoPago === 'pagado').reduce((acc, s) => acc + Number(s.precio || 0), 0)
}

function totalPendientes() {
  return servicios.value.filter(s => s.estadoPago !== 'pagado').reduce((acc, s) => acc + Number(s.precio || 0), 0)
}

function totalPendientesCantidad() {
  return servicios.value.filter(s => s.estadoPago !== 'pagado').length
}
</script>

<style>
html, body, #app {
  width: 100%;
  min-width: 100%;
  margin: 0;
  padding: 0;
  display: block;
}

.pagina {
  font-family: Arial, sans-serif;
  max-width: 1100px;
  margin: 0 auto;
  padding: 16px 24px;
  background: #f5f1e8;
  min-height: 100vh;
  color: #222;
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
  font-size: 18px;
  color: #7a1f1f;
}
.marca p {
  margin: 2px 0 0;
  font-size: 12px;
  color: #666;
}

.btn-nuevo {
  width: 100%;
  background: #7a1f1f;
  color: #fff;
  border: none;
  padding: 12px;
  font-weight: bold;
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
  font-size: 13px;
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
  font-size: 12px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.vacio {
  text-align: center;
  color: #888;
  margin: 24px 0;
}

.ticket {
  position: relative;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 20px;
  margin-bottom: 18px;
  font-size: 14px;
}

.estado {
  position: absolute;
  top: 14px;
  right: 14px;
  font-size: 11px;
  font-weight: bold;
  text-transform: uppercase;
  padding: 3px 10px;
  border-radius: 12px;
}
.estado-verde { background: #e2f3e2; color: #2e6b2e; }
.estado-naranja { background: #fbe8d3; color: #b5590a; }

.ticket-titulo {
  font-size: 17px;
  font-weight: bold;
  margin-bottom: 8px;
}

.separador {
  border-top: 1px solid #eee;
  margin: 10px 0;
}

.ticket-fila {
  display: flex;
  justify-content: space-between;
  padding: 3px 0;
}

.fecha-hora {
  display: flex;
  gap: 10px;
  margin: 8px 0;
}
.caja-fecha {
  flex: 1;
  background: #f5f1e8;
  border-radius: 4px;
  padding: 6px 10px;
  display: flex;
  flex-direction: column;
}
.etiqueta {
  font-size: 11px;
  color: #888;
  text-transform: uppercase;
}

.ticket-total span {
  font-weight: bold;
  font-size: 16px;
  color: #7a1f1f;
}

.calificacion-seccion {
  margin: 8px 0;
}
.estrellas {
  margin-top: 4px;
}
.estrella {
  font-size: 20px;
  color: #ddd;
  cursor: pointer;
  margin-right: 2px;
}
.estrella.llena { color: #b5590a; }
.estrella.llena.baja { color: #7a1f1f; }

.aviso-baja {
  font-size: 12px;
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
  font-size: 13px;
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
  font-size: 13px;
  cursor: pointer;
  font-weight: bold;
}
.btn-mini.editar { background: #2e6b2e; color: white; }
.btn-mini.eliminar { background: #7a1f1f; color: white; }
.btn-mini.cancelar { background: #e0e0e0; color: #222; }

label { display: block; margin-top: 8px; font-weight: bold; font-size: 12px; }
input, select, textarea {
  width: 100%;
  padding: 8px;
  margin-top: 3px;
  border-radius: 4px;
  border: 1px solid #ccc;
  font-family: inherit;
}
.error { color: #7a1f1f; font-size: 11px; margin: 2px 0; }

.fila-doble {
  display: flex;
  gap: 10px;
}
.fila-doble > div {
  flex: 1;
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
}
.modal h2 { margin-top: 0; color: #7a1f1f; font-size: 16px; }
.modal-acciones { display: flex; gap: 10px; margin-top: 16px; }
.modal-acciones .btn-nuevo, .modal-acciones .btn-mini { margin: 0; }
</style>
