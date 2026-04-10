<template>
  <section class="flex items-center justify-center min-h-screen bg-[#08a04b] text-white px-4 py-10 mt-8">
    <div
      class="relative bg-white/95 backdrop-blur-sm text-gray-900 rounded-3xl border border-white/20 w-full max-w-6xl p-6 md:p-8 grid gap-6 mt-1 overflow-hidden shadow-[0_8px_30px_rgba(0,0,0,0.35)]"
    >

      <div class="relative z-10 space-y-6">
        
        <header class="text-center space-y-2">
          <h1 class="text-3xl md:text-4xl font-extrabold text-gray-900">Panel de gastos</h1>
          <p class="text-gray-600">Gestioná y controlá tus gastos</p>
        </header>

        <!-- MENSAJE -->
        <div
          v-if="showFloatingMessage"
          class="p-4 rounded-2xl text-black font-medium border-l-4 shadow-sm bg-green-50 border-green-200 text-green-800"
        >
          {{ floatingMessage }}
        </div>

        <!-- ÚLTIMOS GASTOS -->
        <div v-if="userId" class="bg-white rounded-2xl border border-gray-100 shadow-md p-6">
          <h2 class="text-xl font-bold text-green-700 mb-4">Últimos 3 gastos</h2>

          <ul v-if="latestGastos.length" class="space-y-3">
            <li
              v-for="gasto in latestGastos"
              :key="gasto.id"
              class="bg-gray-50 border border-gray-200 rounded-xl p-3"
            >
              <!-- HEADER -->
              <div class="flex justify-between items-start">
                <p class="text-sm font-semibold text-gray-800">
                  {{ gasto.name || 'Sin nombre' }}
                </p>

                <!-- 🔥 BOTÓN ARRIBA DERECHA -->
                <button
                  @click="openDeleteModal(gasto.id)"
                  class="text-xs text-red-500 hover:text-white hover:bg-red-500 px-2 py-1 rounded-lg transition"
                >
                  Eliminar
                </button>
              </div>

              <p class="text-sm text-gray-600 mt-1">
                {{ gasto.category || 'Sin categoría' }}
              </p>

              <p v-if="gasto.description" class="text-sm text-gray-700 mt-1">
                {{ gasto.description }}
              </p>

              <p class="text-sm font-bold text-red-600 mt-1">
                - {{ formatCurrency(gasto.amount) }}
              </p>
            </li>
          </ul>
        </div>

        <!-- TODOS LOS GASTOS -->
        <div v-if="userId" class="bg-white rounded-2xl shadow-md border border-gray-100 p-6">
          <h2 class="text-xl font-bold text-gray-800 mb-4">Todos los gastos</h2>

          <ul v-if="gastos.length > 0" class="space-y-3 max-h-56 overflow-y-auto pr-2">
            <li
              v-for="gasto in gastos"
              :key="gasto.id"
              class="bg-gray-50 border border-gray-200 p-3 rounded-xl shadow-sm text-sm"
            >
              <div class="flex justify-between items-start">
                <p class="font-medium text-gray-800">
                  {{ gasto.name }} - {{ formatCurrency(gasto.amount) }}
                </p>

                <button
                  @click="openDeleteModal(gasto.id)"
                  class="text-xs text-red-500 hover:text-white hover:bg-red-500 px-2 py-1 rounded-lg transition"
                >
                  Eliminar
                </button>
              </div>

              <p class="text-gray-600 italic">
                {{ gasto.category }}
              </p>

              <p v-if="gasto.description" class="text-sm text-gray-700 mt-1">
                {{ gasto.description }}
              </p>

              <p class="text-gray-500 text-xs mt-1">
                {{ formatDate(gasto.createdAt) }}
              </p>
            </li>
          </ul>
        </div>

      </div>
    </div>

    <!-- 🔥 MODAL ELIMINAR -->
    <div v-if="showDeleteModal" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50">
      <div class="bg-white rounded-2xl p-6 w-full max-w-sm shadow-xl">
        <h3 class="text-lg font-bold text-gray-800 mb-2">
          Eliminar gasto
        </h3>

        <p class="text-gray-600 mb-6">
          ¿Seguro que querés eliminar este gasto? Esta acción no se puede deshacer.
        </p>

        <div class="flex justify-end gap-3">
          <button
            @click="closeDeleteModal"
            class="px-4 py-2 rounded-lg border border-gray-300 text-gray-600 hover:bg-gray-100 transition"
          >
            Cancelar
          </button>

          <button
            @click="confirmDelete"
            class="px-4 py-2 rounded-lg bg-red-500 text-white hover:bg-red-600 transition"
          >
            Eliminar
          </button>
        </div>
      </div>
    </div>

  </section>
</template>

<script>
import { db } from "../services/firebase";
import {
  doc,
  onSnapshot,
  collection,
  query,
  where,
  orderBy,
  deleteDoc,
} from "firebase/firestore";
import { getAuth, onAuthStateChanged } from "firebase/auth";

export default {
  data() {
    return {
      floatingMessage: null,
      showFloatingMessage: false,
      userId: null,
      gastos: [],

      // 🔥 MODAL STATE
      showDeleteModal: false,
      gastoToDelete: null,
    };
  },

  computed: {
    latestGastos() {
      return this.gastos.slice(0, 3);
    },
  },

  methods: {
   formatCurrency(value) {
  const amount = parseFloat(value);
  if (isNaN(amount)) return "Monto inválido";

  return new Intl.NumberFormat("es-AR", {
    style: "currency",
    currency: "ARS",
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  })
  .format(amount)
  .replace(/\s/g, ""); // 👈 esto elimina el espacio entre $ y número
},

    formatDate(tsOrIso) {
      if (tsOrIso?.toDate) return tsOrIso.toDate().toLocaleDateString("es-AR");
      return new Date(tsOrIso).toLocaleDateString("es-AR");
    },

    mostrarMensajeTemporal(mensaje) {
      this.floatingMessage = mensaje;
      this.showFloatingMessage = true;

      setTimeout(() => {
        this.showFloatingMessage = false;
        this.floatingMessage = null;
      }, 2000);
    },

    listenForGastos() {
      const gastosRef = query(
        collection(db, "gastos"),
        where("uid", "==", this.userId),
        orderBy("createdAt", "desc")
      );

      onSnapshot(gastosRef, (querySnapshot) => {
        this.gastos = querySnapshot.docs.map((d) => ({
          id: d.id,
          ...d.data(),
        }));
      });
    },

    // 🔥 MODAL CONTROL
    openDeleteModal(id) {
      this.gastoToDelete = id;
      this.showDeleteModal = true;
    },

    closeDeleteModal() {
      this.showDeleteModal = false;
      this.gastoToDelete = null;
    },

    async confirmDelete() {
      try {
        await deleteDoc(doc(db, "gastos", this.gastoToDelete));
        this.mostrarMensajeTemporal("Gasto eliminado correctamente");
      } catch (error) {
        console.error(error);
        this.mostrarMensajeTemporal("Error al eliminar");
      }

      this.closeDeleteModal();
    },
  },

  mounted() {
    onAuthStateChanged(getAuth(), async (user) => {
      if (!user) return;

      this.userId = user.uid;

      if (!sessionStorage.getItem("user_active_tracked")) {
        if (window.va) {
          window.va("track", "user_active");
        }
        sessionStorage.setItem("user_active_tracked", "true");
      }

      this.listenForGastos();
    });
  },
};
</script>