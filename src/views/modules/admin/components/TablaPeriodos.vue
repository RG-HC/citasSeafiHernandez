<script setup>
import "flatpickr/dist/flatpickr.css";
import "@simonwep/pickr/dist/themes/classic.min.css";
import { ref, reactive,onMounted, watch } from 'vue';
import {helpers, required} from "@vuelidate/validators";
import ModalComponent from "@/components/ModalComponent";
import flatPickr from "vue-flatpickr-component";
import useVuelidate from "@vuelidate/core/dist/index.esm";
import {PeriodosService} from "@/api/connection/providers/Periodos.service";
import {UtilsDate} from "@/helpers/Dates/UtilsDate";
import NotificacionError from "@/helpers/notifications/NotificacionError";
import {NotificacionesModal} from "@/helpers/notifications/notificacionesModal";
import {NotificacionesRecepcion} from "@/helpers/notifications/NotificacionesRecepcion";
import {NotificacionesCitas} from "@/helpers/notifications/NotificacionCitas";
import PantallaCarga from "@/helpers/notifications/PantallaCarga";


const diasPeriodo = ref ([])
const cargandoDatos = ref(false)
const respuestaBack = ref(null)
const paginaActual = ref(1)
const registrosPorPagina = ref(5)
const totalPaginas = ref(0)
const totalRegistros = ref(0)
const primerIndice = ref(0)
const ultimoIndice = ref(0)
const paginacion = ref()
const modalPeriodo = ref()
const datenow = new Date().toLocaleDateString();
const isNew = ref(true)





const state = reactive({
  searchQuery: '',
  idPeriodo: '',
  nombrePeriodo: '',
  fechaInicio: datenow,
  fechaFin: datenow,
  activo: true,
})

const rules = {
  searchQuery: {},
  nombrePeriodo: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  },
  fechaInicio: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  },
  fechaFin: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  },
  activo: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  }
}

const v$ = useVuelidate(rules, state)

const humanfriendlyConfig = {
  defaultDate: "today",
  dateFormat: "d/m/Y",
  minDate: new Date(new Date().getFullYear() - 1, 12, 1),
  maxDate: new Date(new Date().getFullYear() + 1, 12, 1),
}

//Cargar dias
const cargarDatos = async () => {
  try {
    cargandoDatos.value = true
    if (state.searchQuery.length > 0)
      paginaActual.value = 1
    respuestaBack.value = await PeriodosService.ObtenerListaPeriodos(state.searchQuery, paginaActual.value, registrosPorPagina.value, 3, 'asc')
    paginaActual.value = respuestaBack.value.paginaActual
    registrosPorPagina.value = respuestaBack.value.registrosPorPagina
    totalPaginas.value = respuestaBack.value.totalPaginas
    totalRegistros.value = respuestaBack.value.totalRegistros
    diasPeriodo.value = respuestaBack.value.registros
    primerIndice.value = respuestaBack.value.primerIndice
    ultimoIndice.value = respuestaBack.value.ultimoIndice


  } catch (e) {
    throw e
  } finally {
    cargandoDatos.value = false
  }
}

const cambiarPagina = (num) => {
  paginaActual.value = num
  cargarDatos()
}

watch(paginaActual, (newPage) => {
    cambiarPagina(newPage)
})

//mostrar modal del periodo
const mostrarModalP = async (idPeriodo, index)=>{
  let inputFechaInicio = document.querySelector("#inputFechaInicio ")
  let inputFechaFin = document.querySelector("#inputFechaFin ")
  await limpiarModal()
  if (!idPeriodo){
    isNew.value = true
    inputFechaInicio.disabled = false;
    inputFechaFin.disabled = false;
  }else{
    isNew.value = false
    inputFechaInicio.disabled = true
    inputFechaFin.disabled = true
    const dato = await diasPeriodo.value[index]

    state.idPeriodo = dato.idPeriodo
    state.nombrePeriodo = dato.nombrePeriodo
    let fechaConvertidaInicio = UtilsDate.toDayMonthYear(dato.fechaInicio)
    state.fechaInicio = fechaConvertidaInicio
    let fechaConvertidaFin = UtilsDate.toDayMonthYear(dato.fechaFin)
    state.fechaFin = fechaConvertidaFin
    state.activo = dato.activo
  }
  modalPeriodo.value.show()
}

//eliminar periodo
const eliminarPeriodo = async (idPeriodo, fechaInicio, fechaFin) => {
  const confirmado = await NotificacionesRecepcion.ConfirmacionEliminar(fechaInicio + " - "+ fechaFin, state.nombrePeriodo)
  if (confirmado.isConfirmed) {
    try {
      const resp = await PeriodosService.eliminarPeriodo(idPeriodo)
      if (resp) {
        await NotificacionesModal.PantallaExito('Período Eliminado con Éxito')
      }
    } catch (error) {
      await NotificacionError.Agendar(error)
    } finally {
      await cargarDatos()
    }

  }
}

const AgregarPeriodo = async (idPeriodo) => {
    await modalPeriodo.value.hide()
  if (idPeriodo !== '') {

    const resp = await NotificacionesRecepcion.ConfirmacionEditar(state.fechaInicio + " - " + state.fechaFin, 'Modificar') // revisar las notificaciones
    if (resp.isConfirmed)
      try {
        PantallaCarga.mostrar()
        const resp = await PeriodosService.modificarPeriodo(idPeriodo, state.nombrePeriodo, UtilsDate.toDayMonthYear(state.fechaInicio), UtilsDate.toDayMonthYear(state.fechaFin), state.activo)
        if (resp) {
          await NotificacionesModal.PantallaExito('Evento Editado con Éxito');

        }
        await limpiarModal()
      } catch (error) {
        await NotificacionError.Agendar(error)
      } finally {
        PantallaCarga.ocultar()
        await cargarDatos()
      }
  } else { 

   try {
     //const vld = await PeriodosService.ObtenerListaPeriodos(idPeriodo, state.nombrePeriodo, UtilsDate.toDayMonthYear(state.fechaInicio), UtilsDate.toDayMonthYear(state.fechaFin), state.activo)
     const date = await NotificacionesRecepcion.ValidarDiaInhabil(state.fechaInicio + " - " + state.fechaFin, 'Agregar')

     if (date.isConfirmed) {
       try {
         PantallaCarga.mostrar()
         const resp = await PeriodosService.agregarPeriodo(idPeriodo, state.nombrePeriodo, UtilsDate.toDayMonthYear(state.fechaInicio), UtilsDate.toDayMonthYear(state.fechaFin), state.activo)
         await modalPeriodo.value.hide()
         if (resp) {
           await NotificacionesModal.PantallaExito('Evento Agregado con Éxito');
         }
         await limpiarModal()
       } catch (error) {
         await NotificacionError.Agendar(error)
       } finally {
         PantallaCarga.ocultar()
         await cargarDatos();
       }


     }
   }catch (e) {


     const resp = await NotificacionesCitas.Continuar(e)
     if (resp.isConfirmed) {
       try {
         PantallaCarga.mostrar()
         const resp = await PeriodosService.agregarPeriodo(idPeriodo, state.nombrePeriodo, UtilsDate.toDayMonthYear(state.fechaInicio), UtilsDate.toDayMonthYear(state.fechaFin), state.activo)
         await modalPeriodo.value.hide()
         if (resp) {
           await NotificacionesModal.PantallaExito('Evento Agregado con Éxito');

         }
         await limpiarModal()


       } catch (error) {
         await NotificacionError.Agendar(error)
       } finally {
         PantallaCarga.ocultar()
         await cargarDatos();       
       }
     }
   }
  }
}

const limpiarModal = async () => {
  v$.value.$reset()
  state.idPeriodo = ''
  state.nombrePeriodo = ''
  state.fechaInicio = datenow
  state.fechaFin = datenow
  state.activo = true
}

onMounted(() => {
  cargarDatos()  
})

</script>

<template>

<!-----------Ventana vista nuevo período-->

<div class="card"><!--Iniico contenido del periodo-->
  <div class="card-header align-items-center d-flex"><!--Inicio del titulo y botton superior-->
    <h3 class="mb-0 flex-grow-1"> Nuevo Período</h3>
     <div class="flex-shrink-0">
      <div class="form-check form-switch form-switch-right form-switch-md">
        <b-button @click="mostrarModalP()" variant="pantone-green">
        <i class="bx bx-plus-circle fs-2 align-middle"></i>Nuevo período
        </b-button>
      </div>
      </div>
  </div><!--Fin del botón y titulo superior-->

  <!----------inicio de las columnas y la opcion buscar-->
  <div class="card-body border border-dashed border-end-0 border-start-0">
    <div class="row g-3 d-flex justify-content-end">
      <div class="col-md-6 col-sm-12 col-lg-4">
        <div class="search-box">
          <input v-model="v$.searchQuery.$model" type="text" class="form-control search bg-light border-light"
          placeholder="Buscar período" @keyup.stop="cargarDatos()">
          <i class="ri-search-line search-icon"></i>
        </div>
      </div>
    </div>
  </div><!----------Fin de las columnas y la opcion buscar-->

  <div class="card-body">
    <div class="flex-grow-1 overflow-hidden">
      <div class="table-responsive">
        <table class="table align-middle table-nowrap mb-0 table-hover mb-4">
        <thead class="text-muted">
        <tr>
          <th class="sort">ID</th>
          <th class="sort">Nombre</th>
          <th class="sort text-center">Fecha Inicio</th>
          <th class="sort text-center">Fecha Fin</th>
          <th class="sort">Activo</th>
          <th class="sort text-center">Acciones</th>
        </tr>
        </thead>
        <tbody>
          <!-- Inicio donde se muestran los datos -->
          <tr v-for="(diasPeriodo, index) in diasPeriodo">
            <th scope="row">{{index + 1}}</th>
            <td class="fw-semibold">{{diasPeriodo.nombrePeriodo}}</td>
            <td class="text-center">{{UtilsDate.toDayMonthYear(diasPeriodo.fechaInicio)}}</td>
            <td class="text-center">{{UtilsDate.toDayMonthYear(diasPeriodo.fechaFin)}}</td>
            <td class="text-muted"><i :class="diasPeriodo.activo === true ? 'mdi mdi-checkbox-marked fs-1 text-seafi-aqua' : 'mdi mdi-close-box fs-1'"></i></td>
            <td class="text-center">
              <!--------Iniico de los botones dentro de las acciones-->
              <div>
                  <button class="btn btn-light btn-sm dropdown" type="button"
                          data-bs-toggle="dropdown" aria-expanded="false">
                    <i class="mdi mdi-dots-horizontal"></i>
                  </button>
                  <ul class="dropdown-menu">
                    <li>
                      <a class="dropdown-item" style="cursor: pointer" @click="mostrarModalP(diasPeriodo.idPeriodo, index)">
                        <i class="mdi mdi-calendar-edit mdi-18px align-center me-2 text-seafi-aqua"></i>
                        Editar
                      </a>
                    </li>
                    <li>
                      <a class="dropdown-item" style="cursor: pointer" @click="eliminarPeriodo(diasPeriodo.idPeriodo, UtilsDate.toDayMonthYear(diasPeriodo.fechaInicio), UtilsDate.toDayMonthYear(diasPeriodo.fechaFin))">
                        <i class="mdi mdi-calendar-remove mdi-18px align-center me-2 text-seafi-red "></i>
                        Eliminar
                      </a>
                    </li>
                  </ul>
                </div>
                <!----------Fin de los botones de las acciones -->
            </td>
          </tr>
        </tbody>
        </table>
      </div>
    </div>
    <!------Iniico de la paginación-->
    <div class="text-center" v-if="cargandoDatos">
      <div class="spinner">
        <div class="bounce1"></div>
        <div class="bounce2"></div>
        <div class="bounce3"></div>
      </div>
    </div>

    <div v-else>
      <div class="noresult" v-if="diasPeriodo.length === 0">
        <div class="text-center">
          <h5 class="mt-2">No hay registros</h5>
          <p class="text-muted mb-0">
            No se encontraron resultados para mostrar
          </p>
        </div>
      </div>
      <div class="row g-0 text-center text-sm-start align-items-center mb-4" v-else>
        <div class="col-sm-6">
          <div>
            <p class="mb-sm-0 text-muted">
              Mostrando del <span class="fw-semibold">{{primerIndice}}</span> al
              <span class="fw-semibold">{{ultimoIndice}}</span> de
              <span class="fw-semibold text-decoration-underline">{{totalRegistros}}</span>
              registros
            </p>
          </div>
        </div>
        <div class="col-sm-6">
          <ul class="pagination pagination-separated justify-content-center justify-content-sm-end mb-sm-0">
            <b-pagination
               ref="paginacion"
               v-model="paginaActual"
               :total-rows="totalRegistros"
               :per-page="registrosPorPagina"
               first-number
               last-number
            ></b-pagination>
          </ul>
        </div>
      </div>
    </div>
  </div>
  <!-----Fin de la paginacón-->

  <!-------Inicio del modal------------>
  <ModalComponent ref="modalPeriodo">
    <template #header>
      <h5 class="modal-title"><span>{{isNew === true ? 'Nuevo' : 'Modificar'}}</span> Período</h5>
      <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
    </template>
    <template #body>
      <div class="text-center">
        <i
        class="mdi text-seafi-aqua" 
        :class="isNew ? 'mdi-calendar-expand-horizontal' :'mdi-calendar-edit'"
        style="font-size:100px;"
        ></i>
      </div>
      <div class="mb-3">
        <label class="form-label">Nombre del Período
          <span class="text-danger">*</span>
        </label>
        <input 
        type="text"
        class="form-control"
        v-model.trim="v$.nombrePeriodo.$model"
        :class="{'is-invalid': v$.nombrePeriodo.$error,}" id="nombreSistema"
        placeholder="Ingresa un nombre para el período"
        >
        <div v-for="(item, index) in v$.nombrePeriodo.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
      </div>
      <div class="mb-3">
          <label class="form-label">Fecha Inicio<span
              class="text-danger">*</span></label>
          <flatPickr
              v-model="v$.fechaInicio.$model"
              :config="humanfriendlyConfig"
              class="form-control flatpickr-input"
              id="inputFechaInicio"
              :class="{'is-invalid': v$.fechaInicio.$error},!isNew ? 'text-muted' :''"
              :style="!isNew ? 'cursor: not-allowed': ''"
          ></flatPickr>
          <div v-for="(item, index) in v$.fechaInicio.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
        </div>
        <div class="mb-3">
          <label class="form-label">Fecha Fin<span
              class="text-danger">*</span></label>
          <flatPickr
              v-model="v$.fechaFin.$model"
              :config="humanfriendlyConfig"
              class="form-control flatpickr-input"
              id="inputFechaFin"
              :class="{'is-invalid': v$.fechaFin.$error},!isNew ? 'text-muted' :''"
              :style="!isNew ? 'cursor: not-allowed': ''"
          ></flatPickr>
          <div v-for="(item, index) in v$.fechaFin.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
        </div>
        <div 
            class="form-switch form-switch-lg form-check-seafi-aqua" 
            dir="ltr" >
             <input 
              type="checkbox" 
              class="form-check-input" 
              v-model="v$.activo.$model"
              true-value="true"
              false-value="false"
              :class="{'is-invalid' : v$.activo.$error,}"
              id="estaActivo">
              <label class="form-check-label" for="estaActivo">Activo 
              <span class="text-danger" data-v-375a1dc0-s="">*</span>
              </label>
        </div>
    </template>
    <template #footer>
        <div class="d-flex align-items-center col-12">
          <div class="col-6 px-2">
            <button class="btn btn-seafi-aqua w-100" type="button" ref="enviando" :disabled="v.$invalid" @click="AgregarPeriodo(state.idPeriodo )">Aceptar</button>
          </div>
          <div class="col-6 px-2">
            <a class="btn btn-soft-seafi-gray w-100" data-bs-dismiss="modal">Cancelar</a>
          </div>
        </div>
      </template>
  </ModalComponent>
  <!--------Fin del Modal-------->
</div> 
<!-------Fin del contenido de Período -->
</template>