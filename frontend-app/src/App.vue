<template>
  <div class="app">
    <header>
      <h1>Sistema de Pedidos</h1>
      <span class="env-badge">DEVELOPER -v</span>
    </header>

    <main>
      <!-- Formulario -->
      <section class="card">
        <h2>{{ editando ? 'Editar Pedido' : 'Nuevo Pedido' }}</h2>
        <form @submit.prevent="guardar" class="form">
          <div class="form-row">
            <div class="field">
              <label>Cliente</label>
              <input v-model="form.cliente" required placeholder="Nombre del cliente" />
            </div>
            <div class="field">
              <label>Producto</label>
              <input v-model="form.producto" required placeholder="Nombre del producto" />
            </div>
          </div>
          <div class="form-row">
            <div class="field">
              <label>Cantidad</label>
              <input v-model.number="form.cantidad" type="number" min="1" required />
            </div>
            <div class="field">
              <label>Precio Unitario ($)</label>
              <input v-model.number="form.precio" type="number" min="0" step="0.01" required />
            </div>
            <div class="field">
              <label>Estado</label>
              <select v-model="form.estado">
                <option value="PENDIENTE">Pendiente</option>
                <option value="EN_PROCESO">En Proceso</option>
                <option value="ENTREGADO">Entregado</option>
                <option value="CANCELADO">Cancelado</option>
              </select>
            </div>
          </div>
          <div class="form-actions">
            <button type="submit" class="btn btn-primary">
              {{ editando ? 'Actualizar' : 'Crear Pedido' }}
            </button>
            <button v-if="editando" type="button" class="btn btn-secondary" @click="cancelarEdicion">
              Cancelar
            </button>
          </div>
        </form>
        <p v-if="error" class="error-msg">{{ error }}</p>
      </section>

      <!-- Tabla de pedidos -->
      <section class="card">
        <div class="table-header">
          <h2>Pedidos ({{ pedidos.length }})</h2>
          <button class="btn btn-secondary" @click="cargar">Recargar</button>
        </div>

        <div v-if="cargando" class="loading">Cargando...</div>

        <table v-else-if="pedidos.length > 0">
          <thead>
            <tr>
              <th>#</th>
              <th>Cliente</th>
              <th>Producto</th>
              <th>Cantidad</th>
              <th>Precio</th>
              <th>Total</th>
              <th>Estado</th>
              <th>Fecha</th>
              <th>Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in pedidos" :key="p.id">
              <td>{{ p.id }}</td>
              <td>{{ p.cliente }}</td>
              <td>{{ p.producto }}</td>
              <td>{{ p.cantidad }}</td>
              <td>${{ p.precio.toFixed(2) }}</td>
              <td>${{ (p.cantidad * p.precio).toFixed(2) }}</td>
              <td>
                <span :class="['badge', estadoClase(p.estado)]">
                  {{ estadoLabel(p.estado) }}
                </span>
              </td>
              <td>{{ formatFecha(p.fechaCreacion) }}</td>
              <td class="acciones">
                <button class="btn btn-sm btn-edit" @click="editar(p)">Editar</button>
                <button class="btn btn-sm btn-delete" @click="eliminar(p.id)">Eliminar</button>
              </td>
            </tr>
          </tbody>
        </table>

        <p v-else class="empty">No hay pedidos. ¡Crea el primero!</p>
      </section>
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const API = '/api/pedidos'

const pedidos = ref([])
const cargando = ref(false)
const error = ref('')
const editando = ref(false)

const formVacio = () => ({
  cliente: '',
  producto: '',
  cantidad: 1,
  precio: 0,
  estado: 'PENDIENTE'
})

const form = ref(formVacio())
let editId = null

async function cargar() {
  cargando.value = true
  error.value = ''
  try {
    const { data } = await axios.get(API)
    pedidos.value = data
  } catch (e) {
    error.value = 'Error al cargar pedidos: ' + (e.response?.data?.message || e.message)
  } finally {
    cargando.value = false
  }
}

async function guardar() {
  error.value = ''
  try {
    if (editando.value) {
      await axios.put(`${API}/${editId}`, form.value)
    } else {
      await axios.post(API, form.value)
    }
    form.value = formVacio()
    editando.value = false
    editId = null
    await cargar()
  } catch (e) {
    error.value = 'Error al guardar: ' + (e.response?.data?.message || e.message)
  }
}

function editar(p) {
  editando.value = true
  editId = p.id
  form.value = {
    cliente: p.cliente,
    producto: p.producto,
    cantidad: p.cantidad,
    precio: p.precio,
    estado: p.estado
  }
}

function cancelarEdicion() {
  editando.value = false
  editId = null
  form.value = formVacio()
  error.value = ''
}

async function eliminar(id) {
  if (!confirm('¿Eliminar este pedido?')) return
  try {
    await axios.delete(`${API}/${id}`)
    await cargar()
  } catch (e) {
    error.value = 'Error al eliminar: ' + e.message
  }
}

function estadoClase(estado) {
  return {
    PENDIENTE: 'badge-yellow',
    EN_PROCESO: 'badge-blue',
    ENTREGADO: 'badge-green',
    CANCELADO: 'badge-red'
  }[estado] || 'badge-gray'
}

function estadoLabel(estado) {
  return {
    PENDIENTE: 'Pendiente',
    EN_PROCESO: 'En Proceso',
    ENTREGADO: 'Entregado',
    CANCELADO: 'Cancelado'
  }[estado] || estado
}

function formatFecha(fecha) {
  if (!fecha) return '-'
  return new Date(fecha).toLocaleDateString('es-CO', {
    day: '2-digit', month: '2-digit', year: 'numeric'
  })
}

onMounted(cargar)
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: Arial, sans-serif; background: #f0f2f5; color: #333; }

header {
  background: #1a73e8; color: white;
  padding: 16px 32px;
  display: flex; align-items: center; gap: 12px;
}
header h1 { font-size: 1.4rem; }
.env-badge {
  background: #34a853; color: white;
  font-size: 0.75rem; padding: 2px 10px;
  border-radius: 12px; font-weight: bold;
}

main { max-width: 1100px; margin: 24px auto; padding: 0 16px; }

.card {
  background: white; border-radius: 8px; padding: 24px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.1); margin-bottom: 24px;
}
.card h2 { margin-bottom: 16px; font-size: 1.1rem; color: #1a73e8; }

.form { display: flex; flex-direction: column; gap: 12px; }
.form-row { display: flex; gap: 12px; flex-wrap: wrap; }
.field { display: flex; flex-direction: column; gap: 4px; flex: 1; min-width: 150px; }
.field label { font-size: 0.85rem; color: #555; font-weight: bold; }
.field input, .field select {
  padding: 8px 10px; border: 1px solid #ddd;
  border-radius: 6px; font-size: 0.9rem;
}
.field input:focus, .field select:focus { outline: none; border-color: #1a73e8; }
.form-actions { display: flex; gap: 8px; }

.btn {
  padding: 8px 18px; border: none; border-radius: 6px;
  cursor: pointer; font-size: 0.9rem; font-weight: bold;
}
.btn-primary { background: #1a73e8; color: white; }
.btn-primary:hover { background: #1557b0; }
.btn-secondary { background: #f0f0f0; color: #555; }
.btn-secondary:hover { background: #e0e0e0; }
.btn-sm { padding: 4px 10px; font-size: 0.8rem; }
.btn-edit { background: #e8f0fe; color: #1a73e8; }
.btn-delete { background: #fce8e6; color: #d93025; }

.table-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
table { width: 100%; border-collapse: collapse; font-size: 0.88rem; }
th { background: #f8f9fa; text-align: left; padding: 10px 12px; border-bottom: 2px solid #dee2e6; color: #555; }
td { padding: 10px 12px; border-bottom: 1px solid #f0f0f0; }
tr:hover td { background: #fafafa; }
.acciones { display: flex; gap: 6px; }

.badge {
  display: inline-block; padding: 3px 10px;
  border-radius: 12px; font-size: 0.78rem; font-weight: bold;
}
.badge-yellow { background: #fef7e0; color: #f9ab00; }
.badge-blue   { background: #e8f0fe; color: #1a73e8; }
.badge-green  { background: #e6f4ea; color: #34a853; }
.badge-red    { background: #fce8e6; color: #d93025; }

.loading, .empty { text-align: center; padding: 32px; color: #888; }
.error-msg { color: #d93025; margin-top: 8px; font-size: 0.9rem; }
</style>
