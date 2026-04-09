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

        <!-- BOTON AGREGAR GASTO -->
    
          

        <!-- ÚLTIMOS GASTOS -->
        <div v-if="userId" class="bg-white rounded-2xl border border-gray-100 shadow-md p-6">
          <h2 class="text-xl font-bold text-green-700 mb-4">Últimos 3 gastos</h2>

          <ul v-if="latestGastos.length" class="space-y-3">
            <li v-for="gasto in latestGastos" :key="gasto.id" class="bg-gray-50 border border-gray-200 rounded-xl p-3">
              <p class="text-sm font-semibold text-gray-800">{{ gasto.name || 'Sin nombre' }}</p>
              <p class="text-sm text-gray-600">{{ gasto.category || 'Sin categoría' }}</p>
              <p class="text-sm font-bold text-red-600">- {{ formatCurrency(gasto.amount) }}</p>
            </li>
          </ul>

          <p v-else class="text-sm text-gray-500">Todavía no hay gastos cargados.</p>
        </div>

        <!-- LISTA COMPLETA DE GASTOS -->
        <div v-if="userId" class="bg-white rounded-2xl shadow-md border border-gray-100 p-6">
          <h2 class="text-xl font-bold text-gray-800 mb-4">Todos los gastos</h2>

          <ul v-if="gastos.length > 0" class="space-y-3 max-h-56 overflow-y-auto pr-2">
            <li
              v-for="gasto in gastos"
              :key="gasto.id"
              class="bg-gray-50 border border-gray-200 p-3 rounded-xl shadow-sm text-sm"
            >
              <p class="font-medium text-gray-800">
                {{ gasto.name }} - {{ formatCurrency(gasto.amount) }}
              </p>
              <p class="text-gray-600 italic">{{ gasto.category }}</p>
              <p class="text-gray-500 text-xs mt-1">{{ formatDate(gasto.createdAt) }}</p>
            </li>
          </ul>

          <p v-else class="text-gray-500 text-sm">No hay gastos registrados aún.</p>
        </div>

      </div>
    </div>
  </section>
</template>

<script>
import { db } from "../services/firebase";
import {
  doc,
  setDoc,
  onSnapshot,
  collection,
  query,
  where,
  orderBy,
  deleteDoc,
  getDocs,
} from "firebase/firestore";
import { getAuth, onAuthStateChanged } from "firebase/auth";

export default {
  data() {
    return {
      floatingMessage: null,
      showFloatingMessage: false,
      userId: null,

      // ❌ INGRESOS (comentado)
      // tempInitialAmount: null,
      // initialAmount: 0,
      // remainingAmount: 0,
      // additionalAmounts: [],
      // tempAdditionalAmount: null,
      // tempAdditionalDescription: "",
      // initialAmountError: false,
      // additionalAmountError: false,
      // additionalDescriptionError: false,

      gastos: [],
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
      }).format(amount);
    },

    formatDate(tsOrIso) {
      if (!tsOrIso) return "Fecha desconocida";
      if (tsOrIso?.toDate) return tsOrIso.toDate().toLocaleDateString("es-AR");
      return new Date(tsOrIso).toLocaleDateString("es-AR");
    },

    goToAddExpense() {
      this.$router.push("/cargar-gasto");
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
        this.gastos = querySnapshot.docs.map((d) => ({ id: d.id, ...d.data() }));
      });
    },
  },

  async mounted() {
    onAuthStateChanged(getAuth(), async (user) => {
      if (!user) return;
      this.userId = user.uid;

      this.listenForGastos();
    });
  },
};
</script>